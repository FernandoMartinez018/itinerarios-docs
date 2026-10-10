# ADR-015: Correcciones de la auditoría de pasos 1–24 (Spring Boot 4, pruebas, idempotencia)

## Estado
Aceptado. Verificado con build real en ADR-016 (resilience4j corregido a 2.4.0; migración a Jackson 3).

## Contexto
La auditoría del 2026-10-09 (pasos 1 a 24 de §111) encontró problemas que impedían
compilar, probar o cerrar formalmente Nivel 1 y Nivel 2. Nada de lo construido se había
ejecutado todavía (ver CLAUDE.md §10).

## Decisiones

### 1. Dependencias Spring Boot 4 (airport, itinerary, notification)
| Antes | Después | Motivo |
|---|---|---|
| `spring-boot-starter-web` | `spring-boot-starter-webmvc` | Nombre modular de Boot 4 |
| `flyway-core` | `spring-boot-starter-flyway` | En Boot 4 la autoconfiguración de Flyway está en su propio módulo; sin el starter no corren las migraciones y `ddl-auto: validate` falla |
| `spring-boot-starter-aop` | `spring-boot-starter-aspectj` | Renombrado en Boot 4; Resilience4j necesita AOP |
| springdoc `2.8.5` | `3.0.0` | La línea 2.x es para Boot 3 |
| resilience4j `2.2.0` | `2.3.0` | 2.2.x es anterior a Boot 4 y no publica `resilience4j-spring-boot4` |
| — | `spring-boot-starter-webmvc-test` (test) | MockMvc / `@WebMvcTest` quedaron en módulo propio |

### 2. Gateway: Spring Boot 4.0.x con Spring Cloud 2025.1.1
Spring Cloud 2025.1.x declara compatibilidad con Boot 4.0.x y su verificador puede
abortar el arranque con 4.1.x. Se aplica el plan B de ADR-010: el Gateway queda en la
línea 4.0.x (parent `4.0.1`, subir al último parche 4.0.x disponible) mientras los
demás servicios siguen en 4.1.1. No comparten classpath, así que la diferencia no
afecta la arquitectura.
**Alternativa descartada:** desactivar `spring.cloud.compatibility-verifier`; ocultaría
una incompatibilidad real.

### 3. Frontend: Vitest como runner de pruebas
No existía target `test` ni runner. Se adopta `@angular/build:unit-test` + Vitest
(runner por defecto de Angular moderno, sin navegador en CI) en lugar de Karma.
Verificado: `npm ci`, `npm test` (10 pruebas en verde) y `ng build --configuration production`.
CI y Dockerfile pasan a Node 24 (Angular CLI 22.2 exige Node ≥ 22.22.3 / 24.15).

### 4. Idempotencia en Notification
El Outbox entrega "al menos una vez". `V2__unique_sent_event_id.sql` agrega un índice
único parcial (`event_id` con `status = 'SENT'`). Pendiente en código: que
`NotificationProcessor` consulte `existsByEventIdAndStatus(eventId, SENT)` antes de
guardar, para registrar el duplicado como ignorado en vez de como FAILED.

### 5. LocalStack
Se agrega al `docker-compose.yml` en el perfil `serverless` (imagen comunitaria
`localstack/localstack:4.0`). El handler Lambda que reutiliza `NotificationProcessor`
(ADR-012) sigue pendiente.

## Riesgos abiertos
- Versiones fijadas sin acceso a Maven Central: confirmar `springdoc 3.0.0`,
  `resilience4j 2.3.0` y el último parche de Boot 4.0.x.
- **Jackson 3 en Boot 4:** si alguna clase inyecta `com.fasterxml.jackson.databind.ObjectMapper`
  (p. ej. para serializar el payload del Outbox o el converter de RabbitMQ), Boot 4 ya no
  autoconfigura ese bean (usa `tools.jackson.databind.json.JsonMapper`). Revisar en el
  primer `mvn verify` / arranque.
- El agente OpenTelemetry se descarga como `latest`; fijar versión cuando se confirme
  la compatible con Spring Framework 7.
