# Bounded Contexts

El sistema se divide en tres bounded contexts, cada uno con su propio microservicio,
base de datos y modelo de datos independiente (regla fundamental de la sección 12:
nunca compartir bases de datos entre contextos).

## Airport Context

**Microservicio:** `itinerarios-airport-service`
**Base de datos:** `airport_db`

Responsable de:

- El concepto de `Airport` (aeropuerto) como agregado.
- Integración con el proveedor externo API Colombia, a través de un Adapter
  (`ColombiaAirportAdapter`) que traduce el modelo externo al modelo interno.
- Persistencia local de aeropuertos ya consultados (no se sincroniza todo el catálogo
  de una vez; la estrategia es on-demand, ver `adr/ADR-011-airport-sync-strategy.md`).

Este contexto es la **única** fuente de verdad sobre si un aeropuerto existe. Ningún
otro microservicio accede a `airport_db` directamente (sección 102).

## Itinerary Context

**Microservicio:** `itinerarios-itinerary-service`
**Base de datos:** `itinerary_db`

Responsable de:

- El concepto de `Itinerary` (itinerario personal de viaje) como agregado principal.
- CRUD completo de itinerarios.
- Validación de que los aeropuertos de salida y llegada existen, delegando esa pregunta
  al Airport Context vía HTTP (`AirportServiceClient`) — nunca resolviéndola localmente.
- (Nivel 2) Publicación de `ItineraryCreatedEvent` cuando se crea un itinerario.

Itinerary **no** modela una entidad `Airport` completa: solo guarda una referencia
(código IATA) al aeropuerto, evitando duplicar el modelo de otro contexto (sección 11).

## Notification Context

**Microservicio:** `itinerarios-notification` (aún no implementado — llega en Nivel 2, Fase 8)
**Base de datos:** `notification_db`

Responsable de:

- Reaccionar a `ItineraryCreatedEvent`.
- Generar y almacenar el historial de notificaciones (simuladas, sin proveedor de
  correo pago — sección 104).

## Relación entre contextos

```text
Itinerary Context  --(HTTP, síncrono)-->  Airport Context
Itinerary Context  --(evento async, RabbitMQ, Nivel 2)-->  Notification Context
```

No existe comunicación Airport Context -> Itinerary Context ni Notification Context ->
ningún otro contexto. La dependencia es unidireccional en ambos casos.
