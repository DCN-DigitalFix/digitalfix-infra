# DigitalFix

Sistema de gestión de órdenes de trabajo de mantención eléctrica.
Caso semestral DSY1107 (Desarrollo Cloud Native I).

## Módulos

| Módulo | Puerto | Descripción |
|---|---|---|
| `DigitalFix-Frontend/Frontend-DC-DigitalFix` | 4200 | SPA Angular + MSAL (Entra ID) |
| `ms-digitalfix-bff` | 8080 | BFF: valida JWT, autoriza por rol/scope, delega al resto de servicios |
| `ms-digitalfix-workorders` | 8081 | Dominio de OT (JPA: H2 dev / Oracle prod) + productor RabbitMQ y Kafka |
| `ms-digitalfix-catalog` | 8082 | Catálogo (JWT, mismo tenant `gozalo23` que el resto) |
| `ms-digitalfix-audit` | 8083 | Consumidor Kafka (grupo `audit-timeline`): timeline de auditoría por orden; además produce a `digitalfix.audit.events` |
| `ms-digitalfix-notify` | 8084 | Consumidor RabbitMQ (3 colas + DLQ) |
| `ms-digitalfix-report` | 8086 | Consumidor Kafka (grupos `report-metrics` y `report-audit-trail`): resumen, certificados y conteo de auditoría |
| `ms-digitalfix-kafka-admin` | 8087 | Administración del clúster Kafka: tópicos, particiones, configuración, consumer groups, estado del clúster |
| Kafka UI | 8089 | Administración visual de brokers/tópicos (`docker-compose.kafka-ui.yml`) |
| `ms-digitalfix-rabbitmq-admin` | 8090 | Administración de colas/exchanges/bindings |
| `docker-compose.yml` | — | Clúster RabbitMQ de 2 nodos |
| `docker-compose.kafka.yml` | — | Clúster Kafka: 3 Zookeeper + 3 brokers |

`audit` y `report` leen el mismo tópico `digitalfix.workorders.events`, cada
uno con su propio consumer group, así que ninguno le quita mensajes al otro.
`report` además lee `digitalfix.audit.events` (lo que `audit` produce) con
un tercer grupo, `report-audit-trail`.

## Ejecución local

1. RabbitMQ: `docker compose up -d`
2. Kafka (ver sección siguiente)
3. Microservicios: en cada carpeta `./mvnw spring-boot:run`
4. Frontend: `npm install && npm start`

## Kafka: levantar el clúster y Kafka UI

Requiere un `.env` en esta carpeta con `KAFKA_HOST` (la IP o el hostname por
el que los microservicios y Kafka UI alcanzan a los brokers; en local,
`localhost`) y, para Kafka UI, `KAFKA_UI_USER`/`KAFKA_UI_PASSWORD`.

```
docker compose -f docker-compose.kafka.yml --env-file .env up -d
docker compose -f docker-compose.kafka-ui.yml --env-file .env up -d
```

El primer compose levanta 3 Zookeeper + 3 brokers (`confluentinc/cp-kafka`,
replicación 3, `min.insync.replicas=2`: tolera la caída de un broker sin
perder escrituras confirmadas con `acks=all`) y, al final, un contenedor
`kafka-init` que crea los tópicos y termina (no vuelve a correr solo si se
edita el compose; para tópicos nuevos en un clúster ya levantado hay que
crearlos a mano con `kafka-topics --create` o recrear el clúster).
`KAFKA_AUTO_CREATE_TOPICS_ENABLE=false`: los tópicos se declaran a
propósito, no se crean solos por un typo en el nombre.

Kafka UI queda en `http://localhost:8089` (usuario/clave desde `.env`;
protegido porque desde ahí también se pueden borrar tópicos).

### Tópicos

| Tópico | Particiones / réplicas | Retención | Productor | Consumidores |
|---|---|---|---|---|
| `digitalfix.workorders.events` | 3 / 3 | 7 días | `workorders` | `audit` (`audit-timeline`), `report` (`report-metrics`) |
| `digitalfix.workorders.events.DLT` | 3 / 3 | 30 días | — (error handler de `audit`/`report`) | — |
| `digitalfix.audit.events` | 3 / 3 | 30 días | `audit` | `report` (`report-audit-trail`) |
| `digitalfix.audit.events.DLT` | 3 / 3 | 30 días | — (error handler de `report`) | — |

Administrar estos tópicos (crear otros nuevos, cambiar configuración,
ampliar particiones, inspeccionar consumer groups) es exactamente lo que
hace `ms-digitalfix-kafka-admin` vía su API REST, como alternativa a usar
Kafka UI o la CLI directamente.

## Perfiles

- `workorders`: perfil `dev` (H2) por defecto; `prod` usa Oracle vía `SPRING_PROFILES_ACTIVE=prod` y variables `ORACLE_HOST`, `ORACLE_PORT`, `ORACLE_SERVICE`, `ORACLE_USER`, `ORACLE_PASSWORD`.

## Buenas prácticas aplicadas

- Validación JWT (firma, iss, aud, exp) en cada microservicio.
- Autorización conjuntiva por rol (App Roles) y scope (OAuth 2.0).
- Errores uniformes en JSON (401/403/404/400).
- Mensajería desacoplada con RabbitMQ: colas, exchanges direct/topic, DLQ y ACK manual.
- Configuración centralizada de nombres de colas/exchanges en `application.yml`.
- Separación de configuración de mensajería en clases `@Configuration`.
