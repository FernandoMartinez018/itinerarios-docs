# Agregados

## Airport (Airport Context)

- **Raíz de agregado:** `Airport`.
- **Invariantes:**
  - `externalId` es único y obligatorio (identidad frente al proveedor externo).
  - `iataCode` e `icaoCode`, cuando existen, son únicos.
  - Un `Airport` siempre pertenece exactamente a un `city` y opcionalmente a un `department`
    (ambos almacenados como texto plano, no como entidades separadas — no se modeló
    `City`/`Department` como agregados propios porque el sistema no necesita operar
    sobre ellos de forma independiente, solo mostrarlos).
- **Ningún otro microservicio debe manipular este agregado directamente.** Itinerary
  Service lo consulta solo a través de la API REST de Airport Service
  (`GET /api/airports/iata/{iataCode}`), nunca escribiéndolo.

## Itinerary (Itinerary Context)

- **Raíz de agregado:** `Itinerary`.
- **Invariantes:**
  - `durationMinutes > 0` (validado tanto en Bean Validation del DTO de entrada como en
    un `CHECK` de base de datos — doble validación intencional: la de aplicación da un
    mensaje de error claro al usuario; la de base de datos es la última línea de defensa
    si algún proceso escribe directo a la base saltándose la aplicación).
  - `departureAirportId` y `arrivalAirportId` deben corresponder a aeropuertos que
    existan según Airport Service en el momento de crear/actualizar (no se garantiza
    que sigan existiendo después — no hay FK real entre bases, ver `database/database-design.md`).
- **No contiene una entidad Airport completa.** Solo referencias por código IATA
  (`departureAirportId`, `arrivalAirportId` como `VARCHAR`). Esto es deliberado: evita
  acoplar el modelo de Itinerary al modelo de Airport, y evita compartir tablas/entidades
  entre microservicios (sección 102).

## Referencia cruzada, no relación de agregado

`Itinerary.departureAirportId` y `Itinerary.arrivalAirportId` son **referencias lógicas**
a `Airport.iataCode`, no relaciones de agregado en el sentido DDD estricto (no hay
transacción conjunta, no hay integridad referencial a nivel de base de datos, y pueden
quedar "colgando" si un aeropuerto cambiara de código — escenario no contemplado en
Nivel 1 porque los códigos IATA son estables en la práctica).
