# Pruebas Unitarias

## Cobertura actual (Nivel 1)

| Servicio | Clase de test | Qué cubre |
|---|---|---|
| airport-service | `AirportServiceTest` | Búsqueda por id (éxito/404), búsqueda por IATA con sincronización contra el Adapter cuando no existe localmente, y 404 cuando tampoco existe en el proveedor externo |
| airport-service | `CircuitBreakerFailureTest` | Prueba de fallo controlado (sección 29): transición CLOSED → OPEN → HALF_OPEN → CLOSED, y rechazo inmediato (`CallNotPermittedException`) mientras el circuito está abierto |
| itinerary-service | `ItineraryServiceTest` | Creación exitosa con validación de ambos aeropuertos, fallo cuando el aeropuerto de salida no existe, 404 al buscar/eliminar un id inexistente, escritura del OutboxEvent PENDING en la misma operación |
| itinerary-service | `OutboxPublisherTest` | Publicación exitosa marca PUBLISHED, fallo de RabbitMQ marca FAILED e incrementa retry_count, eventos que agotaron el límite de reintentos no se vuelven a intentar |
| notification | `NotificationProcessorTest` | Guarda historial SENT en el camino feliz, y FAILED cuando la persistencia falla |
| gateway | `SimpleRateLimitGlobalFilterTest` | Rate limiting permite tráfico bajo el límite y bloquea con HTTP 429 al superarlo |
| frontend | `session.service.spec.ts` | Login/logout del mock de sesión |
| frontend | `itinerary.service.spec.ts`, `airport.service.spec.ts` | Verbos y rutas contra el Gateway; nunca contra API Colombia |

Frontend: Vitest vía `@angular/build:unit-test` (`npm test -- --watch=false`). Verificado el 2026-10-09: 10 pruebas en verde (ADR-015).

Todas usan Mockito (backend) o TestBed (frontend) para aislar la unidad bajo prueba de
sus dependencias reales (base de datos, red, otros servicios).

## Qué falta (a completar antes de cerrar Nivel 1 formalmente)

- Tests de `ColombiaAirportMapper` (mapeo `oaciCode` -> `icaoCode`, `deparmentId` con
  el typo real del proveedor).
- Tests de `GlobalExceptionHandler` de ambos servicios backend (verificar el código
  HTTP y el shape de `ErrorResponse` para cada tipo de excepción).
- Tests de API con MockMvc (sección 37) para los controllers de los 4 servicios
  backend (dependencia `spring-boot-starter-webmvc-test` ya agregada).
- Tests de componentes Angular (`ItineraryFormComponent`, `AirportListComponent`) —
  actualmente solo hay test de servicio, no de componente.

## Convención

- Un archivo de test por clase bajo prueba, mismo paquete, sufijo `Test` (Java) o
  `.spec.ts` (Angular).
- Nombres de método de test descriptivos en español, en la línea de
  `create_fallaCuandoAeropuertoDeSalidaNoExiste`, para que un test que falla en CI se
  entienda sin abrir el código.
