# Guion de defensa EP6 — Kafka en DigitalFix

EP6 es individual, 5 a 10 minutos. Este guion cubre todo lo que pide la
pauta; cada persona lo usa como base para su propia defensa, con sus
palabras. Los tiempos son una referencia para que la suma quede entre 7 y 9
minutos, dejando margen para preguntas.

Antes de presentar: tener Kafka, Kafka UI y los microservicios (`workorders`,
`audit`, `report`, `kafka-admin`) levantados y probados una vez completos,
para no descubrir un problema en vivo.

---

## 1. Introducción (30 s)

"DigitalFix usa Kafka para que los eventos de negocio de las órdenes de
trabajo —se crean, cambian de estado— lleguen a varios servicios
independientes sin que el que los genera (`workorders`) sepa quién los
consume ni cuántos son. Hoy lo consumen `audit` y `report`; mañana podría
sumarse otro sin tocar `workorders`."

---

## 2. Zookeeper y brokers (1–1.5 min)

- El clúster tiene **3 Zookeeper + 3 brokers** (`confluentinc/cp-zookeeper`
  y `cp-kafka`, Docker Compose).
- Zookeeper guarda los metadatos del clúster: qué brokers existen, cuál es
  el controlador, la configuración de cada tópico. Los brokers son los que
  realmente almacenan y sirven los mensajes.
- Con 3 Zookeeper en vez de 1, el clúster tolera la caída de uno sin perder
  el quórum (necesita mayoría: 2 de 3 vivos). Con 1 solo, cualquier caída
  tumba la coordinación de todo el clúster.
- Con 3 brokers, cada tópico puede replicarse 3 veces: un mensaje no
  depende de un solo disco.

Punto para quien pregunte "¿por qué no usar un solo broker?": en un solo
broker, si se cae, el clúster completo queda sin servicio y cualquier dato
no reflejado en disco en ese instante se pierde. Con 3, el liderazgo de una
partición se reasigna a otro broker que ya tenía la réplica.

---

## 3. Por qué 2 tópicos, 3 particiones, 3 réplicas (1 min)

| Tópico | Particiones | Réplicas | Retención | Quién produce | Quién consume |
|---|---|---|---|---|---|
| `digitalfix.workorders.events` | 3 | 3 | 7 días | `workorders` | `audit` (grupo `audit-timeline`), `report` (grupo `report-metrics`) |
| `digitalfix.audit.events` | 3 | 3 | 30 días | `audit` | `report` (grupo `report-audit-trail`) |

(Además existe una DLT por cada tópico — se explica en la sección de
limitaciones, punto 11.)

- **Dos tópicos** porque son dos eventos de negocio distintos con dueños
  distintos: el evento crudo de la orden (lo produce `workorders`) y el
  registro normalizado de auditoría (lo produce `audit`, derivado del
  primero). No es un capricho de diseño, es que `report` necesita ambos
  por razones distintas: el primero para el resumen de órdenes, el segundo
  para contar eventos de auditoría.
- **3 particiones**: permite que hasta 3 instancias de un mismo consumer
  group procesen en paralelo sin pisarse. Con 1 partición, un grupo con
  varios hilos no ganaría nada — Kafka no reparte una partición entre dos
  consumidores del mismo grupo.
- **3 réplicas** con `min.insync.replicas=2`: cada escritura confirmada con
  `acks=all` queda en al menos 2 de los 3 brokers antes de responder al
  productor. Si un broker cae, no se pierde ningún mensaje ya confirmado.

---

## 4. Kafka UI (1 min, mostrar en vivo)

Abrir `http://localhost:8089`, iniciar sesión, y mostrar:

- Los tópicos, sus particiones y su configuración de retención.
- Los mensajes de `digitalfix.workorders.events` (clic en un mensaje:
  mostrar la clave `ordenId` y el valor JSON).
- Los consumer groups activos: `audit-timeline`, `report-metrics`,
  `report-audit-trail`, cada uno con su lag.

"Esto es una herramienta visual externa; para administración programática
—crear tópicos, cambiar retención, ampliar particiones desde una API— está
`ms-digitalfix-kafka-admin`, que se muestra en la sección 8."

---

## 5. Flujo productor → consumidor (1.5 min)

1. Un usuario crea una orden (`POST /api/v1/workorders`) o un supervisor
   cambia su estado (`PATCH /api/v1/workorders/{id}/estado`).
2. `WorkOrderService` guarda el cambio en la base de datos (fuente de
   verdad) y luego llama a `WorkOrderKafkaPublisher`, que publica un
   `WorkOrderEvent` en `digitalfix.workorders.events`, con la clave
   `ordenId`.
3. La publicación es `acks=all` + `enable.idempotence=true`: el broker
   confirma solo cuando todas las réplicas sincrónicas tienen el mensaje,
   y un reintento del productor nunca lo duplica en el log de Kafka.
4. `audit` y `report` leen el mismo tópico, cada uno con su propio
   consumer group y su propio offset — ninguno le quita mensajes al otro.
5. `audit` guarda el evento en su tabla (clave primaria = `eventId`, así
   un reintento de Kafka no duplica la fila) y publica un segundo evento,
   más normalizado, en `digitalfix.audit.events`. `report` lo consume con
   un tercer grupo (`report-audit-trail`) solo para contar cuántos
   eventos de auditoría existen.

"Si falla la publicación a Kafka, la orden ya quedó guardada en la base de
datos: Kafka es una notificación derivada, no la fuente de verdad. El error
se registra en el log, no revierte la operación."

---

## 6. Diseño del evento y la clave `ordenId` (1 min)

```json
{
  "eventId": "7f2e...",
  "eventType": "ORDEN_CREADA",
  "occurredAt": "2026-11-22T10:00:00Z",
  "traceId": "a91c...",
  "ordenId": "OT-0001",
  "estadoAnterior": null,
  "estadoNuevo": "INGRESADA",
  "cliente": "Cliente Prueba",
  "tecnicoAsignado": null,
  "actor": "subject-del-jwt"
}
```

- `eventType` distingue `ORDEN_CREADA` de `ESTADO_CAMBIADO`.
- `eventId` es único por publicación (`UUID.randomUUID()`), no por orden:
  es la clave de idempotencia aguas abajo.
- **La clave del mensaje Kafka es `ordenId`, no `eventId`.** Esto es
  deliberado: todos los eventos de una misma orden caen siempre en la
  misma partición, así que un solo consumidor los procesa en el orden
  exacto en que ocurrieron. Si la clave fuera `eventId` (aleatoria), dos
  eventos de la misma orden podrían caer en particiones distintas y
  procesarse fuera de orden por hilos distintos.
- Campos desconocidos no invalidan el mensaje
  (`FAIL_ON_UNKNOWN_PROPERTIES=false`): `workorders` puede agregar campos
  nuevos sin romper a los consumidores existentes.

---

## 7. Reacción en tiempo real (30 s–1 min)

"Entre que se publica el evento y que `audit`/`report` lo guardan pasa
menos de un segundo en condiciones normales — no es un batch nocturno. Se
puede ver en vivo: creo una orden, y antes de que termine de cargar la
pantalla del frontend, el evento ya aparece en Kafka UI y la línea de
tiempo de auditoría ya tiene la fila nueva."

---

## 8. Administración: particiones, réplicas, retención, consumer groups (1.5 min)

Mostrar `ms-digitalfix-kafka-admin` (puerto 8087), el microservicio que
expone la administración del clúster como API REST, protegida por rol
`Admin`:

- `GET /api/v1/kafka/topics` — lista tópicos con particiones, factor de
  replicación y configuración actual.
- `PUT /api/v1/kafka/topics/{nombre}/partitions` — amplía particiones a un
  nuevo total (Kafka no permite reducirlas; el servicio lo valida antes de
  llamar al broker).
- `PUT /api/v1/kafka/topics/{nombre}/config` — cambia configuración en caliente
  (por ejemplo `retention.ms`), sin reiniciar el clúster.
- `DELETE /api/v1/kafka/topics/{nombre}` — borra un tópico, salvo que esté en
  la lista de protegidos (los 4 tópicos de negocio y sus DLT).
- `GET /api/v1/kafka/consumer-groups` y `GET /api/v1/kafka/consumer-groups/{id}`
  — estado del grupo, miembros y sus particiones asignadas.

"Antes de crear un tópico o ampliar particiones, el servicio valida contra
el clúster real (`describeCluster()`): por ejemplo, rechaza con 400 un
factor de replicación mayor a la cantidad de brokers vivos, en vez de
dejar que el broker lo rechace con un error más críptico."

---

## 9. Monitoreo: offsets y lag (1 min)

En el mismo `kafka-admin`, `GET /api/v1/kafka/consumer-groups/{id}` devuelve,
por partición: `offsetActual` (hasta dónde leyó el grupo), `offsetFinal`
(hasta dónde llega el tópico) y `lag` (la diferencia).

"Lag es la métrica que le importa a quien opera el sistema: si `lag` crece
y no baja, el consumidor no está al día con la producción — está cayendo
atrás, o quedó caído. En este sistema, con el volumen actual, el lag se
mantiene en 0 casi todo el tiempo; mostrar eso en vivo demuestra que los
tres consumer groups están al día."

---

## 10. Casos de uso (30 s)

- **Auditoría**: reconstruir qué pasó con una orden y cuándo, sin consultar
  directamente la base de datos transaccional de `workorders`.
- **Reportería**: métricas agregadas (órdenes por estado, tiempo promedio
  de cierre, certificados) sin que `report` dependa de que `workorders`
  esté disponible en el momento de la consulta — lee de su propia copia.
- **Desacoplamiento de servicios**: se puede agregar un cuarto consumidor
  en el futuro (por ejemplo, notificaciones push a técnicos) sin tocar
  ninguno de los servicios existentes.

---

## 11. Limitaciones a declarar antes de que pregunten (30 s–1 min)

Decir esto proactivamente transmite más control que esperar a que lo
encuentren:

- La **DLT es un tercer tópico** además de los dos pedidos por el enunciado
  (`<tópico>.DLT`, uno por cada tópico con consumidor). Es necesaria para
  no perder mensajes inválidos o que agotaron sus reintentos, pero es
  infraestructura de resiliencia, no un tercer evento de negocio.
- `ddl-auto: update` en vez de migraciones versionadas (Flyway/Liquibase):
  aceptable para el alcance del curso, no para producción real.
- La publicación a Kafka es *best-effort* respecto del CRUD: si Kafka está
  caído, la orden se guarda igual y el evento se pierde (no hay cola de
  reintento del lado del productor). Se decidió así para no bloquear la
  operación de negocio por una falla de mensajería.
- No hay pruebas con Testcontainers contra un broker real en CI — las
  pruebas de integración usan `@EmbeddedKafka` (broker embebido en el
  mismo proceso de test), que cubre el contrato pero no las condiciones de
  red de un clúster real.

---

## 12. Demo paso a paso (2–3 min)

1. Mostrar Kafka UI con los 3 consumer groups al día (lag 0).
2. `POST /api/v1/workorders` desde el frontend (o Postman) → crear una orden
   nueva.
3. En Kafka UI, abrir `digitalfix.workorders.events` → mostrar el mensaje
   nuevo, con su clave `ordenId` y `eventType: "ORDEN_CREADA"`.
4. Ir a la pantalla de Auditoría del frontend (o `GET
   /api/v1/audit/orders/{id}/timeline`) → mostrar que el evento ya aparece en
   la línea de tiempo.
5. `PATCH /api/v1/workorders/{id}/estado` → cambiar el estado de la orden.
6. Mostrar el segundo mensaje en Kafka UI (`ESTADO_CAMBIADO`,
   `estadoAnterior`/`estadoNuevo` poblados) y la línea de tiempo
   actualizada con dos eventos.
7. Abrir la pantalla de Administración de Kafka del frontend (o `GET
   /api/v1/kafka/consumer-groups/audit-timeline`) → mostrar que el lag sigue
   en 0: el consumidor absorbió los dos eventos sin atrasarse.
8. (Opcional, si hay tiempo) Producir un mensaje inválido directamente en
   Kafka UI (`{no es json`) y mostrar que aparece en
   `digitalfix.workorders.events.DLT` en vez de tumbar el consumidor —
   demuestra el manejo de errores sin necesitar un segundo código en vivo.
