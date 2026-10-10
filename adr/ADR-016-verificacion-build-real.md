# ADR-016: Verificación con build real y correcciones (pasos 1–24)

## Estado
Aceptado. Builds Maven **ejecutados** el 2026-10-09 (Maven 3.9.16, JDK 21.0.12).
Complementa ADR-015 y reemplaza su estado "pendiente de verificación".

## Contexto
ADR-015 corrigió los poms sin poder compilar. En esta revisión sí hubo acceso a Maven Central:
se corrió `mvn -B clean verify` en los 4 servicios backend. Aparecieron errores que la
revisión estática no podía detectar.

## Resultado del build

| Servicio | Antes | Después |
|---|---|---|
| airport-service | no resolvía dependencias | 88 tests, BUILD SUCCESS |
| itinerary-service | no resolvía dependencias | 19 tests, BUILD SUCCESS |
| notification | BUILD SUCCESS (2 tests) | 4 tests, BUILD SUCCESS |
| gateway | BUILD SUCCESS (2 tests) | sin cambios |

## Decisiones

1. **resilience4j 2.3.0 → 2.4.0.** `resilience4j-spring-boot4:2.3.0` no existe en Maven Central;
   la única versión publicada es 2.4.0 (verificado en `maven-metadata.xml`).
2. **Jackson 2 → Jackson 3 en código de runtime.** Boot 4.1 solo autoconfigura
   `tools.jackson.databind.json.JsonMapper`. `ItineraryService`, `OutboxPublisher` y
   `RedisCacheConfig` inyectaban `com.fasterxml.jackson.databind.ObjectMapper`, que no
   existe como bean, así que el contexto **no habría arrancado**. Se migró a
   `tools.jackson.databind.ObjectMapper`. En RabbitMQ, `Jackson2JsonMessageConverter`
   pasó a `JacksonJsonMessageConverter` (Spring AMQP 4). Las anotaciones
   `com.fasterxml.jackson.annotation.*` se mantienen porque Jackson 3 las conserva.
   - Alternativa descartada: agregar `spring-boot-jackson2`. Es un módulo de transición,
     deprecado, y dejaría dos mappers con configuraciones distintas.
3. **Caché Redis tipada.** `GenericJackson2JsonRedisSerializer`, sin *default typing*,
   devolvía `LinkedHashMap` en un HIT y provocaba un `ClassCastException` en el Service, justo en
   la evidencia "Redis hit". Ahora se usa `JacksonJsonRedisSerializer<AirportDto>`.
   - Alternativa descartada: activar *default typing*. Abre deserialización polimórfica
     (riesgo de seguridad conocido) para un caso donde el tipo siempre es `AirportDto`.
4. **Errores de Spring MVC con su código HTTP (§65).** El `@ExceptionHandler(Exception.class)`
   convertía en 500 una ruta inexistente (404), un path variable no numérico (400) y
   un JSON malformado (400). Ahora esos casos responden 400 explícito y los
   `org.springframework.web.ErrorResponse` usan su propio código. Los 500 reales se registran en el log.
5. **Idempotencia del consumidor.** `NotificationProcessor` ignora eventos ya notificados
   (`existsByEventIdAndStatus`). Ante una carrera, `DataIntegrityViolationException`
   (índice V2) se trata como duplicado y no se registra un FAILED falso.
6. **Tests de API con MockMvc (§37):** `AirportControllerTest`, `ItineraryControllerTest`
   (200/201/204/400/404/500/502/503, JSON de §64 y Correlation ID).
   `RedisCacheConfigTest` hace round-trip del serializer.
7. **Tests unitarios corregidos:** `@InjectMocks` no inyectaba el mapper ni el `ObjectMapper` reales
   (no son `@Mock`) y producía un NPE. Ahora el servicio se construye explícitamente.
8. **Perfil `prod` (§67)** en airport, itinerary y notification: sin valores por defecto para
   secretos ni hosts. ECS debe definir `SPRING_PROFILES_ACTIVE=prod`.
9. **Agente OpenTelemetry fijado a v2.31.1** (ago-2026) en los 4 Dockerfiles, en lugar de `latest`.
10. **E2E Selenium, tramo Nivel 2 (§38):** después de crear el itinerario, se consulta
    `GET /api/notifications` vía Gateway hasta encontrar una notificación SENT nueva (máximo 30 s).
11. Se eliminaron las carpetas vacías `{...}` del frontend y de docs (brace expansion fallida).

## Pendiente (no verificable en esta revisión)
- **Arranque real del stack**: Docker Desktop no respondía (engine con error 500). Falta correr
  `verificar-stack.ps1`, Selenium y llenar `evidence/`. Solo eso cierra Nivel 1 y Nivel 2.
- **Handler Lambda de Notification + LocalStack** (§77, §106): sigue sin código.
- Mapa vacío hasta crear itinerarios (ADR-011): riesgo para la evidencia "Visualización de aeropuertos".
- `ItineraryEventPublisher` (publicación directa de Nivel 2) ya no se usa desde el Outbox; se conserva
  como referencia del Double Write Problem.

## Addendum — Arranque real del stack (2026-10-09, mismo día)

Con Docker operativo se corrió `verificar-stack.ps1 -E2E -MantenerStack`. Errores encontrados
**solo en ejecución** y corregidos:

| # | Problema | Corrección |
|---|---|---|
| 12 | `jaegertracing/all-in-one:1.62` no existe (los tags llevan parche) | `1.76.0`, última v1. Deuda: v1 está fuera de soporte; migrar a `jaegertracing/jaeger` v2 |
| 13 | Ningún repo tenía `.dockerignore`: `node_modules` rompía el build del frontend y los `.env` entraban al contexto de la imagen | `.dockerignore` en los 5 repos |
| 14 | **CORS sin efecto**: en WebFlux un `CorsConfigurationSource` suelto no se aplica, así que el navegador bloqueaba todas las llamadas de Angular | Bean `CorsWebFilter` + `CorsConfigTest` |
| 15 | API Colombia repite IATA (`BOG` = CATAM militar con OACI `N/A`, y El Dorado `SKBO`); el Adapter tomaba el primero | Se prefiere el resultado con OACI real; test agregado |
| 16 | Correlation ID propagado pero invisible: ningún servicio escribía en el log en una request exitosa | Línea de acceso con el ID en Gateway, Itinerary y Airport (sin `/actuator`); el header ya no sale duplicado |
| 17 | Listener RabbitMQ de Notification construido a mano: ignoraba `retry` y `default-requeue-rejected` (un mensaje no deserializable se reencolaba sin fin) | `SimpleRabbitListenerContainerFactoryConfigurer` |
| 18 | E2E: `sendKeys` en `input type=date` depende del idioma de Chrome; `urlContains("/itineraries")` ya se cumplía en `/new` | Valor por JS + evento `input`; `urlMatches("/itineraries$")` |

| 19 | Con el breaker abierto, el fallback devolvía `ExternalServiceException` y Retry la reintentaba 3 veces con backoff (0,5–1 s por request, anulando el fail-fast) | `CircuitOpenException` (subclase, sigue siendo 503) en `retry.ignore-exceptions`: ahora falla en 15 ms |
| 20 | Grafana sin dashboards (evidencia §96 vacía) | Dashboard provisionado desde Git: `observability/dashboards/itinerarios-overview.json` |

**Lambda (§31-32, §77):** `NotificationLambdaHandler` (RequestStreamHandler, Spring sin web,
perfil `lambda` que desactiva el listener) reutiliza `NotificationProcessor`.
`serverless/deploy-notification-lambda.ps1` empaqueta un ZIP de 59 MB (subido vía S3 porque
supera el límite de 50 MB de subida directa), crea `SendNotificationFunction` en LocalStack 4.0
y la invoca: SENT registrado. Como permite §32, RabbitMQ → Lambda no se simula; la invocación es directa.

Resultado: **Nivel 1 y Nivel 2 verificados en ejecución** (ver `evidence/`). Quedan las capturas
y el CI en GitHub Actions.
