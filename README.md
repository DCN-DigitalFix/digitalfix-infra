# DigitalFix

Sistema de gestión de órdenes de trabajo de mantención eléctrica.
Caso semestral DSY1107 (Desarrollo Cloud Native I).

## Módulos

| Módulo | Puerto | Descripción |
|---|---|---|
| `DigitalFix-Frontend/Frontend-DC-DigitalFix` | 4200 | SPA Angular + MSAL (Entra ID) |
| `ms-digitalfix-bff` | 8080 | BFF: valida JWT, autoriza por rol/scope, delega a workorders |
| `ms-digitalfix-workorders` | 8081 | Dominio de OT (JPA: H2 dev / Oracle prod) + productor RabbitMQ |
| `ms-digitalfix-report` | 8082 | Reportería (JWT) |
| `ms-digitalfix-notify` | 8083 | Consumidor RabbitMQ (3 colas + DLQ) |
| `ms-digitalfix-rabbitmq-admin` | 8090 | Administración de colas/exchanges/bindings |
| `docker-compose.yml` | — | Clúster RabbitMQ de 2 nodos |

## Ejecución local

1. RabbitMQ: `docker compose up -d`
2. Microservicios: en cada carpeta `./mvnw spring-boot:run`
3. Frontend: `npm install && npm start`

## Perfiles

- `workorders`: perfil `dev` (H2) por defecto; `prod` usa Oracle vía `SPRING_PROFILES_ACTIVE=prod` y variables `ORACLE_HOST`, `ORACLE_PORT`, `ORACLE_SERVICE`, `ORACLE_USER`, `ORACLE_PASSWORD`.

## Buenas prácticas aplicadas

- Validación JWT (firma, iss, aud, exp) en cada microservicio.
- Autorización conjuntiva por rol (App Roles) y scope (OAuth 2.0).
- Errores uniformes en JSON (401/403/404/400).
- Mensajería desacoplada con RabbitMQ: colas, exchanges direct/topic, DLQ y ACK manual.
- Configuración centralizada de nombres de colas/exchanges en `application.yml`.
- Separación de configuración de mensajería en clases `@Configuration`.
