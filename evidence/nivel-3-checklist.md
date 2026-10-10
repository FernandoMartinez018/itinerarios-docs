# Evidencias pendientes - Nivel 3

Checklist de capturas/videos que debe prepararse una vez que el stack corra
realmente (seccion 96 de la especificacion maestra).

## Transactional Outbox (Fase 12)

- [ ] Fila nueva en `outbox_events` con `status=PENDING` inmediatamente despues
      de un `POST /api/itineraries` exitoso (consultable con `psql` o cualquier
      cliente PostgreSQL contra `itinerary_db`).
- [ ] La misma fila cambiando a `status=PUBLISHED` con `published_at` seteado,
      unos segundos despues (el ciclo del `OutboxPublisher`).
- [ ] Demostracion del Double Write Problem resuelto: apagar RabbitMQ
      (`docker compose stop rabbitmq`), crear un itinerario (debe seguir
      funcionando - el POST no depende de RabbitMQ), verificar que el
      `outbox_events` queda en `PENDING`/`FAILED` con `retry_count` subiendo,
      volver a levantar RabbitMQ (`docker compose start rabbitmq`), y verificar
      que el evento se publica solo en el siguiente ciclo del scheduler sin
      perderse ni duplicarse.
- [ ] Corrida de `OutboxPublisherTest` en verde.

## Pendientes de fases posteriores

- [ ] Despliegue AWS (Fase 14-15): no iniciado.

## CI/CD (Fase 13)

- [ ] Workflow `ci.yml` en verde (job `build-test`) en los 5 repositorios.
- [ ] Job `push-image` en verde: imagen visible en ECR con tag = SHA corto
      (requiere Fase 14: proveedor OIDC, rol IAM y repositorios ECR).
- [ ] Job `deploy` en verde con smoke test (requiere Fase 15).
- [ ] Demostracion de rollback: re-desplegar el tag SHA anterior.
