# ADR-013: Resilience4j para Retry y Circuit Breaker

## Estado
Aceptado e implementado (Fase 10).

## Contexto
La seccion 28-29 exige Timeout, Retry, Exponential Backoff, Jitter y Circuit Breaker
en dos puntos: Airport Service -> API Colombia, e Itinerary Service -> Airport
Service. Hystrix (la libreria historica en el ecosistema Spring Cloud) esta en modo
mantenimiento desde hace anos; Resilience4j es el reemplazo estandar de facto.

## Investigacion de compatibilidad con Spring Boot 4
Se confirmo la existencia de `io.github.resilience4j:resilience4j-spring-boot4`,
un starter dedicado publicado especificamente para Spring Boot 4 (visto en Maven
Central, snapshot de marzo de 2026). Se uso ese starter en vez de
`resilience4j-spring-boot3` (pensado para Spring Boot 3.x) o de integrar
`spring-cloud-starter-circuitbreaker-resilience4j` (que en discusiones de la propia
comunidad de Spring Cloud mostraba fricciones con Spring Boot 4 / Spring Framework 7
al momento de esta investigacion).

## Decision
`resilience4j-spring-boot4` + `spring-boot-starter-aop`, con anotaciones
`@Retry` + `@CircuitBreaker` sobre los metodos que hacen la llamada HTTP real
(`ColombiaAirportAdapter` en Airport Service, `AirportServiceClient` en Itinerary
Service), configuracion en `application.yml`, y fallback methods explicitos que
traducen "circuito abierto" a la misma excepcion de dominio que ya usaba cada
servicio para "proveedor no disponible" (`ExternalProviderException` /
`ExternalServiceException`), para no introducir un tercer tipo de error HTTP.

## Decision de diseno: que cuenta como "fallo"
En Itinerary Service, `InvalidAirportException` (HTTP 404 - el aeropuerto no existe)
esta configurada explicitamente como excepcion **ignorada** por Retry y Circuit
Breaker. Es una respuesta de negocio valida, no una falla transitoria: reintentarla
no cambiaria el resultado, y contarla como fallo abriria el circuito por trafico
normal (usuarios probando codigos IATA invalidos), bloqueando tambien las
validaciones legitimas que llegaran despues. Solo cuentan como fallo real los
timeouts, conexiones rechazadas, y errores 5xx (`ExternalServiceException`).

## Verificacion pendiente
No se pudo ejecutar `mvn dependency:tree` real en este entorno (sin acceso a Maven
Central) para confirmar que `resilience4j-spring-boot4` resuelve sin conflictos
contra Spring Boot 4.1.1 / Spring Framework 7. Es el mismo tipo de riesgo que
ADR-010 documenta para Spring Cloud Gateway - verificar en el primer build real.

## Consecuencias
- La prueba de fallo controlado (`CircuitBreakerFailureTest`, en
  `itinerarios-airport-service`) se escribio contra la API de Resilience4j
  directamente (sin contexto Spring), para que sea determinista y no dependa de
  mocks de HTTP ni de tiempos de red reales.
- Ambos servicios ahora dependen de `spring-boot-starter-aop`, necesario para que
  las anotaciones de Resilience4j funcionen via proxies AOP.
