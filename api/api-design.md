# Diseño de API

## Convenciones generales

- REST sobre HTTP, JSON como formato de intercambio.
- Cada microservicio expone su propio Swagger/OpenAPI en `/swagger-ui.html` (sección 66).
- Errores en formato consistente (`ErrorResponse`, sección 64), con `correlationId`
  para poder rastrear el request en los logs de todos los servicios involucrados.
- Todo el tráfico externo pasa por el Gateway (`http://localhost:8080` en local). Los
  puertos directos de cada microservicio (8081, 8082, 8083) solo se exponen para debugging.

## Airport Service — `/api/airports`

| Método | Ruta | Descripción | Errores posibles |
|---|---|---|---|
| GET | `/api/airports` | Lista aeropuertos ya sincronizados localmente | — |
| GET | `/api/airports/{id}` | Aeropuerto por id interno | 404 si no existe |
| GET | `/api/airports/iata/{iataCode}` | Aeropuerto por código IATA; sincroniza contra API Colombia si no existe localmente | 404 si no existe ni localmente ni en API Colombia; 502 si API Colombia falla |

## Itinerary Service — `/api/itineraries`

| Método | Ruta | Descripción | Errores posibles |
|---|---|---|---|
| GET | `/api/itineraries` | Lista todos los itinerarios | — |
| GET | `/api/itineraries/{id}` | Itinerario por id | 404 |
| POST | `/api/itineraries` | Crea un itinerario, validando ambos aeropuertos | 400 (validación de formato o aeropuerto inexistente), 503 (Airport Service caído) |
| PUT | `/api/itineraries/{id}` | Actualiza un itinerario, validando ambos aeropuertos | 404, 400, 503 |
| DELETE | `/api/itineraries/{id}` | Elimina un itinerario | 404 |

## Notification Service — `/api/notifications` (solo lectura)

| Método | Ruta | Descripción | Errores posibles |
|---|---|---|---|
| GET | `/api/notifications` | Historial completo | — |
| GET | `/api/notifications/itinerary/{itineraryId}` | Historial de un itinerario | — |

### Contrato de ItineraryRequest (POST/PUT)

```json
{
  "userName": "string, requerido",
  "departureAirportId": "string, requerido, codigo IATA",
  "arrivalAirportId": "string, requerido, codigo IATA",
  "travelDate": "yyyy-MM-dd, requerido, hoy o futuro",
  "durationMinutes": "entero > 0, requerido"
}
```

## Gateway — routing

| Path | Destino |
|---|---|
| `/api/airports/**` | `airport-service` |
| `/api/itineraries/**` | `itinerary-service` |
| `/api/notifications/**` | `notification-service` |

El Gateway no expone rutas propias mas alla del ruteo; su propio `/actuator/health`
existe solo para chequeos de salud (usado por el workflow de CI de E2E).

## Contrato externo consumido: API Colombia

Ver `events/` para el contrato de eventos internos, y el codigo de
`ColombiaAirportResponse` en `itinerarios-airport-service` para el contrato real
verificado contra `https://docs.api-colombia.com`. Dos particularidades reales de ese
proveedor que el Adapter respeta sin "corregir": el campo `oaciCode` (no `icaoCode`) y
el campo `deparmentId` (typo real del proveedor, sin la "t").
