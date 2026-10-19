# Cómo aplicar estos cambios

Este paquete reemplaza tres módulos y el `docker-compose.yml`. El BFF, el
frontend, `report` y `audit` no se tocan.

| Carpeta | Qué hacer |
|---|---|
| `ms-digitalfix-notify` | Reemplazar el módulo completo |
| `ms-digitalfix-rabbitmq-admin` | Reemplazar el módulo completo |
| `ms-digitalfix-workorders` | Reemplazar el módulo completo |
| `docker-compose.yml` | Reemplazar |
| `.env.example`, `.gitignore` | Copiar a la raíz |

Antes de levantar nada:

```
cp .env.example .env
```

---

## Qué se arregló

### Credenciales

Los tres servicios usaban `guest`/`guest` mientras el compose creaba
`admin`/`admin`. Además, **RabbitMQ bloquea a `guest` desde cualquier
conexión que no venga del propio broker**, así que ni corrigiendo el nombre
habría conectado desde otro host o contenedor.

Ahora los tres leen `RABBIT_USER` / `RABBIT_PASS`, con `digitalfix` por
defecto, y apuntan a los dos nodos vía `addresses` en vez de un solo `host`.
Si el primer nodo cae, la conexión pasa al segundo sin reiniciar.

### Topología

Se adoptaron los nombres del caso DigitalFix, que es lo que el profesor
puede contrastar con su propio documento:

| Antes | Ahora |
|---|---|
| `digitalfix.direct` | `cmd.direct` |
| `digitalfix.topic` | `cmd.topic` |
| — | `cmd.dead.dlx` |
| `digitalfix.notificaciones` | `q.cmd.email` |
| `digitalfix.tickets` | `q.cmd.dispatch` |
| `digitalfix.documentos` | `q.cmd.report` |

Tres cosas que faltaban:

**El exchange `topic` no tenía ni un binding.** Estaba declarado y nada más.
Ahora enlaza las tres colas con `email.*`, `dispatch.#` y `report.*`, como
pide el caso. EP4 exige mostrar "exchanges de tipo direct y topic
correctamente configurados" y con cero bindings eso no se sostenía.

**No existía el dead letter exchange.** Las DLQ colgaban del mismo exchange
de comandos usando el nombre de la cola como routing key. Ahora hay un
`cmd.dead.dlx` dedicado.

**El compose mentía.** El contenedor `rabbitmq-cluster-init` tenía `|| true`
en cada línea y después imprimía "Cluster inicializado" pasara lo que pasara.
Ahora el join va dentro de `rabbit2`, encadenado con `&&`: si falla, el
contenedor se detiene y se nota. Además espera un healthcheck real en vez de
un `sleep 20` a ciegas.

### Consumidores

Había una sola clase `NotificationConsumer` con tres métodos. La pauta
describe exactamente eso en el nivel de 60%: *"mezclados dentro de un mismo
paquete o clase sin una separación clara por dominios"*.

Ahora son tres clases en `messaging/consumer/`, una por dominio, más un
`DlqMonitor` que registra las tres DLQ y lleva el contador por cola.

### Manejo de ACK — el indicador de 20%

El código anterior hacía `basicAck` al final y `basicNack(false)` en el
catch. Pero el bloque `try` solo escribía un log: no podía lanzar excepción.
En la práctica **nada llegaba nunca a una DLQ**, y no existía el concepto de
reintento.

Ahora cada consumidor tiene cuatro salidas explícitas:

| Situación | Decisión |
|---|---|
| Procesado correctamente | `basicAck` |
| `eventId` ya visto | `basicAck`, sin reprocesar |
| Mensaje inválido o `ErrorNoRecuperableException` | `basicNack(requeue=false)` → DLQ |
| `ErrorRecuperableException`, quedan intentos | `basicNack(requeue=true)` |
| `ErrorRecuperableException`, intentos agotados | `basicNack(requeue=false)` → DLQ |

La diferencia entre las dos excepciones es si el tiempo puede arreglar el
problema: un SMTP caído sí, un correo sin arroba no.

Los servicios de `service/` ahora tienen lógica que **puede fallar** —
validan campos obligatorios y llaman a un proveedor. Sin eso no habría nada
que demostrar en EP4.

### Mensajes

Se publicaba un `Map` suelto. Ahora va el envelope del caso, con `type`,
`eventId`, `timestamp`, `traceId` y `correlationId`. El `eventId` es lo que
permite deduplicar: RabbitMQ garantiza entrega **al menos una vez**, así que
un evento puede repetirse si el ACK se pierde al caer un nodo.

### Administrador

| Antes | Ahora |
|---|---|
| Solo POST y DELETE de colas, POST de bindings | + GET de cola, purga, DELETE de bindings |
| `type` como String libre | `enum TipoExchange`, se valida al deserializar |
| Solo `@NotBlank` en el nombre | + `@Pattern` y `@Size` |
| `durable` primitivo → `false` si falta | `Boolean`, por defecto `true` |
| `deleteQueue` ignoraba el resultado | 404 si la cola no existe |
| `bind()` con una variable sin usar | eliminada |
| Respuestas `204 No Content` vacías | `OperacionResponse` con recurso y detalle |

Se mantuvo el `@PreAuthorize("hasRole('Admin')")` que ya tenían, porque EP3
pide que el backend valide el JWT del IDaaS. `/api/health` queda público para
tener el contraste 200 vs 401.

### Detalle menor

En `WorkOrderService` las publicaciones estaban envueltas en
`try { } catch (Exception e) { }` con el catch vacío. Un fallo del broker
desaparecía sin dejar rastro. Ahora el publisher captura y **registra**, y el
servicio lo llama directo.

---

## Orden para probarlo

**1. El clúster**

```
docker compose up -d
docker exec digitalfix-rabbit1 rabbitmqctl cluster_status
```

En `Running Nodes` tienen que aparecer `rabbit@rabbit1` y `rabbit@rabbit2`.
No confíes en ningún mensaje de "inicializado": este comando es la prueba.

**2. notify**

```
cd ms-digitalfix-notify
./mvnw spring-boot:run
```

Recarga `http://localhost:15672`. En *Queues* deben aparecer seis colas y en
*Exchanges* los tres `cmd.*`. Entra a `cmd.topic` y verifica que tenga sus
tres bindings: esa pantalla es la evidencia de EP4.

**3. workorders**

```
cd ms-digitalfix-workorders
./mvnw spring-boot:run
```

Crea una orden y pásala a `ASIGNADA`. En el log de notify tienen que
aparecer dos mensajes de dos dominios distintos, uno de email y otro de
dispatch, desde un solo evento.

**4. La DLQ**

Los servicios reconocen un campo `simularFallo` en `datos`, con valores
`recuperable` o `permanente`. Desde la Management UI, publica en `cmd.direct`
con routing key `email.send`:

```json
{
  "type": "orden.asignada",
  "eventId": "prueba-dlq-1",
  "timestamp": "2026-10-18T22:15:03Z",
  "traceId": "manual",
  "correlationId": "OT-0001",
  "datos": {
    "ordenId": "OT-0001",
    "destinatario": "cliente@ejemplo.cl",
    "simularFallo": "recuperable"
  }
}
```

En el log se ven los tres intentos y la derivación final a la DLQ. Con
`"permanente"` va directo, sin reintentar. `GET /api/dlq` devuelve el
acumulado por cola.

**5. El administrador**

```
cd ms-digitalfix-rabbitmq-admin
./mvnw spring-boot:run
```

`GET /api/health` sin token → 200. `POST /api/queues` sin token → 401. Con un
token de rol `Cliente` → 403. El token sale del botón **Copiar access token**
en `/profile` del frontend.

---

## Pendiente

- `ms-digitalfix-catalog` y `ms-digitalfix-audit` siguen vacíos
- `DigitalFix-Backend` todavía tiene el scaffold del tutorial
  (`mi_api_spring_msal`). Conviene archivarlo o borrarlo: suma ruido en
  "buenas prácticas"
- Desplegar el clúster en su propia EC2 (`ec2-mq`) y apuntar los servicios
  con `RABBIT_ADDRESSES`
