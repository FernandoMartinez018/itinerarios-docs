# ADR-011: Sincronizacion on-demand de aeropuertos (no batch)

## Estado
Aceptado.

## Contexto
Airport Service necesita datos de API Colombia, pero puede sincronizarlos de dos
formas: (a) un job que descarga todo el catalogo de aeropuertos de Colombia de una vez
al arrancar o periodicamente, o (b) consultar bajo demanda y persistir solo lo que
efectivamente se usa.

## Decision
Sincronizacion on-demand: `GET /api/airports/iata/{iataCode}` consulta primero
`airport_db`; si no existe, consulta el Adapter contra API Colombia y persiste el
resultado antes de responder. No existe ningun job de sincronizacion masiva en
Nivel 1.

## Justificacion
- La especificacion no pide explicitamente sincronizar el catalogo completo, solo
  "validar que los aeropuertos utilizados existan" (seccion 1).
- API Colombia tiene un limite de 60 solicitudes/minuto por IP; un job de sincronizacion
  masiva al arrancar competiria por ese limite con el trafico real de validacion de
  itinerarios, sin necesidad real de tener el catalogo completo de antemano.
- Agregar un job de sincronizacion es complejidad no solicitada (violaria la regla de
  "no introducir servicios pagos innecesarios" en espiritu, aunque no en letra: aqui
  seria complejidad de codigo innecesaria, no costo economico).

## Consecuencias
- La primera vez que se crea un itinerario con un codigo IATA nunca antes consultado,
  esa peticion especifica es mas lenta (incluye la llamada HTTP a API Colombia). Las
  siguientes veces con el mismo codigo son inmediatas (ya esta en `airport_db`).
- El endpoint `GET /api/airports` (lista completa) solo devuelve los aeropuertos que
  ya se consultaron al menos una vez, no el catalogo completo de Colombia. Esto es
  visible en la UI: el mapa de aeropuertos empieza vacio y se va poblando a medida que
  se crean itinerarios. Se documenta este comportamiento en el mensaje que muestra
  `AirportListComponent` cuando la lista esta vacia.
