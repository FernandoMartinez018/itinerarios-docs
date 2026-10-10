# Lenguaje Ubicuo

## Airport Context

| Término | Significado |
|---|---|
| **Airport** | Aeropuerto utilizado como origen o destino de un itinerario. Agregado del Airport Context. |
| **External Airport** | Representación del aeropuerto tal como la entrega API Colombia, antes de traducirse al modelo interno. |
| **Adapter** | Componente (`ColombiaAirportAdapter`) que traduce el modelo externo al modelo interno y aísla al resto del sistema de los detalles/errores de API Colombia. |
| **External ID** | Identificador nativo del proveedor externo (el `id` numérico de API Colombia). No debe asumirse igual al `id` interno de `airport_db`. |
| **IATA Code** | Código de 3 letras del aeropuerto (ej. `BOG`). Puede ser nulo en aeropuertos regionales pequeños. Es el identificador que usa Itinerary Context para referenciar un aeropuerto. |
| **ICAO Code** | Código de 4 letras del aeropuerto. API Colombia lo expone bajo el nombre `oaciCode`; se renombra a `icaoCode` en el mapeo hacia el modelo interno. |

## Itinerary Context

| Término | Significado |
|---|---|
| **Itinerary** | Itinerario personal de viaje. Agregado principal del Itinerary Context. |
| **Departure Airport** | Aeropuerto de salida de un itinerario, referenciado por su código IATA (no por una entidad Airport completa). |
| **Arrival Airport** | Aeropuerto de llegada, misma regla que Departure Airport. |
| **Airport Validation** | Proceso por el cual Itinerary Service confirma, vía HTTP contra Airport Service, que un código IATA corresponde a un aeropuerto real antes de crear o actualizar un itinerario. |
| **ItineraryCreatedEvent** | Evento de integración emitido después de crear exitosamente un itinerario (Nivel 2). Consumido por Notification Context. |

## Notification Context (Nivel 2)

| Término | Significado |
|---|---|
| **Notification** | Aviso generado como consecuencia de un `ItineraryCreatedEvent`. Puede ser simulado (sin proveedor de correo real). |
| **Notification History** | Registro persistente de cada notificación procesada, con su estado (`PENDING`, `SENT`, `FAILED`). |

## Términos transversales

| Término | Significado |
|---|---|
| **Correlation ID** | Identificador único por request (header `X-Correlation-ID`), generado en el Gateway o en el primer servicio que recibe la petición, propagado a todos los servicios downstream y devuelto en la respuesta. Permite rastrear un request de punta a punta en los logs. |
| **Gateway** | Único punto de entrada del sistema. No contiene lógica de negocio; solo routing, CORS, rate limiting y (a futuro) propagación de JWT. |
| **Adapter** (uso general) | Patrón usado para cualquier integración con un sistema externo (API Colombia es la única instancia en Nivel 1). |
