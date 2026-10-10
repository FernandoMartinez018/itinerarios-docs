# Diseño de Base de Datos

## Regla fundamental (secciones 12 y 102)

Cada microservicio tiene su propia base PostgreSQL. Nunca se comparte una base entre
microservicios, y nunca se crean Foreign Keys entre bases de datos distintas. Las
referencias entre agregados de distintos contextos son siempre **lógicas** (un valor
que se sabe que debería existir en otra base, pero cuya integridad no la garantiza
PostgreSQL sino la aplicación, vía HTTP).

Ver `relational-model.puml` para el diagrama completo.

## airport_db

Única tabla: `airports`. Ver migración real en
`itinerarios-airport-service/src/main/resources/db/migration/V1__create_airports_table.sql`.

Decisiones de diseño:

- `external_id` es el `id` de API Colombia (convertido a `String`), no el código IATA,
  porque el IATA puede ser nulo en aeropuertos regionales pequeños y no sirve como
  identificador único confiable.
- `iata_code` e `icao_code` son `UNIQUE` pero **nullable** — no todos los aeropuertos
  colombianos pequeños tienen ambos códigos asignados.
- No hay columna `country` porque el sistema solo maneja aeropuertos de Colombia
  (alcance explícito de la especificación, sección 1).

## itinerary_db

Tablas: `itineraries` (Nivel 1) y `outbox_events` (Nivel 3, Fase 12), ambas en la
misma base. Ver migraciones reales en
`itinerarios-itinerary-service/src/main/resources/db/migration/`
(`V1__create_itineraries_table.sql`, `V2__create_outbox_events_table.sql`).

`outbox_events` vive deliberadamente en la misma base que `itineraries` y no en
una base separada: el patrón Transactional Outbox depende de que ambas tablas
participen de la misma transacción de PostgreSQL (ver
`adr/ADR-008-transactional-outbox.md`) — separarlas en bases distintas
invalidaría por completo la garantía que el patrón busca dar.

Decisiones de diseño:

- `departure_airport_id` y `arrival_airport_id` son `VARCHAR(50)`, no `BIGINT`, porque
  almacenan el código IATA (ej. `"BOG"`), no el id interno de `airport_db` — coherente
  con que el payload de `ItineraryCreatedEvent` (sección 25) también usa códigos IATA.
- `duration_minutes` tiene un `CHECK (duration_minutes > 0)` a nivel de base de datos,
  además de la validación `@Positive` en el DTO de entrada de la aplicación. Es
  intencional tener ambas: la de aplicación da un mensaje de error claro al usuario;
  la de base de datos protege la integridad incluso si algo escribiera directo a la
  tabla saltándose la aplicación.

## notification_db (Nivel 2, no implementado todavía)

Única tabla: `notification_history` (sección 16). `itinerary_id` es una referencia
lógica sin FK real, por la misma regla de aislamiento entre bases.

## Migraciones

Flyway, una carpeta `db/migration` por microservicio, versionado `V1__`, `V2__`, etc.
Nunca editar una migración que ya corrió en un entorno compartido (sección 99) — cualquier
cambio de esquema posterior a `V1` requiere una nueva migración (`V2__`, `V3__`, ...).
