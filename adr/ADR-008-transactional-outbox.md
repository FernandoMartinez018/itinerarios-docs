# ADR-008: Transactional Outbox para ItineraryCreatedEvent

## Estado
Aceptado e implementado (Fase 12, Nivel 3).

## Contexto
Desde Nivel 2 (Fase 7), `ItineraryService.create()` guardaba el itinerario en
PostgreSQL y luego publicaba `ItineraryCreatedEvent` a RabbitMQ como dos
operaciones separadas y no atomicas. Si la publicacion fallaba despues de que la
transaccion de PostgreSQL ya habia hecho commit, el itinerario quedaba guardado
pero el evento se perdia para siempre - el "Double Write Problem" documentado
explicitamente como limitacion conocida en `ItineraryEventPublisher` (ya
eliminado, ver mas abajo).

## Decision
Implementar el patron Transactional Outbox (seccion 15/81):

1. `ItineraryService.create()` ahora escribe DOS filas en la MISMA transaccion
   de PostgreSQL: el `Itinerary` y un `OutboxEvent` con estado `PENDING` y el
   payload del evento serializado como JSON. O ambas se confirman, o ninguna.
2. `OutboxPublisher`, un componente `@Scheduled` separado, sondea la tabla
   `outbox_events` cada 2 segundos (configurable), publica a RabbitMQ las filas
   `PENDING`/`FAILED` que no hayan agotado su limite de reintentos, y marca cada
   una como `PUBLISHED` o incrementa su `retry_count` si vuelve a fallar.

Esto mueve el punto de fallo: ya no puede perderse un evento silenciosamente,
porque la fila en `outbox_events` sigue existiendo hasta que se publique
exitosamente. En el peor caso (RabbitMQ caido por un tiempo prolongado), el
evento simplemente espera en la tabla hasta que el broker vuelva, sin perder el
itinerario ni el evento.

## Cambios de diseno respecto a Nivel 2
- Se elimino `ItineraryEventPublisher` (publicaba directo a RabbitMQ tras el
  save, best-effort, sin garantias). Su unica responsabilidad util
  (`RabbitTemplate.convertAndSend`) la absorbio `OutboxPublisher`.
- `ItineraryService` ya no depende de RabbitMQ en absoluto - solo escribe en
  PostgreSQL (itinerario + outbox event). El acoplamiento con el broker quedo
  aislado completamente en `OutboxPublisher`.

## Por que polling en vez de Change Data Capture (CDC)
Se evaluo brevemente usar CDC (ej. Debezium leyendo el WAL de PostgreSQL) en vez
de un polling programado, pero se descarto para el alcance de este proyecto:
CDC agrega un componente de infraestructura adicional (Kafka Connect o similar)
desproporcionado para el volumen real de eventos de un proyecto academico. El
polling cada 2 segundos es una latencia aceptable para el caso de uso
(notificacion de un itinerario creado, no un requisito de tiempo real).

## Limite de reintentos
`MAX_RETRY_COUNT = 5` en `OutboxPublisher`: despues de 5 intentos fallidos, el
evento deja de reintentarse automaticamente y queda en estado `FAILED` con su
`error_message`, disponible para investigacion manual. Esto evita que un evento
permanentemente irrecuperable (ej. un problema de serializacion) consuma
recursos del scheduler indefinidamente cada 2 segundos para siempre.

## Consecuencias
- La tabla `outbox_events` crece indefinidamente (nunca se borran filas
  `PUBLISHED`). No se implemento una tarea de limpieza/archivado en este
  alcance; seria una mejora natural de Nivel 3 si el volumen real lo justificara.
- `OutboxPublisherTest` cubre los tres casos relevantes: publicacion exitosa,
  fallo con incremento de `retry_count`, y no reintentar eventos que ya
  agotaron el limite.
