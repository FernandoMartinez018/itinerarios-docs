# Evidencias pendientes - Nivel 3

Checklist de capturas/videos que debe prepararse una vez que el stack corra
realmente (seccion 96 de la especificacion maestra).

## Transactional Outbox (Fase 12)

- [x] Fila nueva en `outbox_events` con `status=PENDING` inmediatamente despues
      de un `POST /api/itineraries` exitoso (consultable con `psql` o cualquier
      cliente PostgreSQL contra `itinerary_db`).
- [x] La misma fila cambiando a `status=PUBLISHED` con `published_at` seteado,
      unos segundos despues (el ciclo del `OutboxPublisher`).
- [x] Demostracion del Double Write Problem resuelto (`logs/outbox-double-write-2026-10-09.txt`; reveló y corrigió la pérdida de eventos, ADR-017): apagar RabbitMQ
      (`docker compose stop rabbitmq`), crear un itinerario (debe seguir
      funcionando - el POST no depende de RabbitMQ), verificar que el
      `outbox_events` queda en `PENDING`/`FAILED` con `retry_count` subiendo,
      volver a levantar RabbitMQ (`docker compose start rabbitmq`), y verificar
      que el evento se publica solo en el siguiente ciclo del scheduler sin
      perderse ni duplicarse.
- [x] Corrida de `OutboxPublisherTest` en verde (incluye backoff).

## Pendientes de fases posteriores

- [x] Paso 25 — infraestructura base AWS: presupuesto, OIDC, ECR, red y RDS probados (`logs/aws-paso-25-2026-10-09.txt`, ADR-017). RDS apagado tras la prueba.
- [x] Paso 26 — push a ECR vía OIDC (ADR-017 addendum).
- [x] Paso 27 — microservicios en ECS Fargate: flujo completo con notificación vía RabbitMQ en AWS (`logs/aws-paso-27-ecs-2026-10-13.txt`). Apagado tras la prueba.
- [x] Paso 28 — Lambda cloud: SendNotificationFunction procesa el evento en AWS e idempotencia verificada (`logs/aws-paso-28-lambda-2026-10-13.txt`). Apagado tras la prueba.
- [ ] Paso 29 — pruebas cloud integradas.

## CI/CD (Fase 13)

- [x] Workflow `ci.yml` en verde (job `build-test`) en los 5 repositorios (PR #1 → `dev`).
- [x] Job `push-image` en verde: imagen en ECR con tag = SHA corto en los 5 repos
      (`logs/aws-paso-26-ecr-2026-10-13.txt`; imágenes borradas después por costo).
- [ ] Job `deploy` en verde con smoke test (requiere Fase 15).
- [ ] Demostracion de rollback: re-desplegar el tag SHA anterior.
