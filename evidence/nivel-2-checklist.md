# Evidencias — Nivel 2

Sección 96 de la especificación maestra. **[x] = verificado en ejecución real** (2026-10-09,
`logs/observabilidad-2026-10-09.txt` y `logs/lambda-localstack-2026-10-09.log`). Las capturas se
toman con el stack arriba.

| | Evidencia | Verificación automática | Captura |
|---|---|---|---|
| [x] | RabbitMQ: queue `notification.itinerary-created.queue` con 1 consumer | API management | pendiente (`localhost:15672`, guest/guest) |
| [x] | Evento `ItineraryCreatedEvent` (Outbox → RabbitMQ) | outbox `PUBLISHED` + notificación generada | pendiente (logs de notification-service) |
| [x] | Notification: fila en `notification_history`, `GET /api/notifications` | `verificar-stack` + Selenium | pendiente |
| [x] | Redis hit/miss | 1ª llamada 0,65 s, 2ª llamada 0,013 s; claves `airports-by-iata::*` | pendiente (`redis-cli KEYS '*'`) |
| [x] | Circuit Breaker CLOSED → OPEN → HALF_OPEN → CLOSED | demo en vivo: `logs/circuit-breaker-retry-2026-10-09.txt` (abierto: 503 en 15 ms) | pendiente (panel de CB en Grafana durante la demo) |
| [x] | Retry con backoff y jitter | demo en vivo: 3 intentos en la primera falla; no reintenta con el breaker abierto | pendiente (logs de itinerary-service) |
| [x] | Jaeger: trazas de los 4 servicios | API de Jaeger lista gateway, itinerary, airport, notification | pendiente (traza de `POST /api/itineraries` en `localhost:16686`) |
| [x] | Correlation ID: mismo `X-Correlation-ID` en Gateway, Itinerary y Airport | `evid-corr-778` en los 3 logs | pendiente |
| [x] | Prometheus: 4 targets `up` | API `/api/v1/targets` | pendiente (`localhost:9090/targets`) |
| [x] | LocalStack + Lambda: `SendNotificationFunction` (java21) procesa el evento y registra SENT | `serverless/deploy-notification-lambda.ps1` | pendiente |
| [x] | Idempotencia: un evento repetido no duplica la fila SENT | tests de `NotificationProcessor` + índice único V2 | opcional |
| [x] | Grafana: datasource conectado y dashboard "Itinerarios - Visión general" (requests, 5xx, latencia, CB) | API de Grafana: health OK, consultas con datos | pendiente (`localhost:3000`) |
