# ADR-010: Spring Boot 4.1.1 + Spring Cloud 2025.1.1

## Estado
Aceptado, con un plan de contingencia documentado.

## Contexto
Al iniciar la construccion (septiembre 2026), Spring Boot 3.5.x —la ultima linea 3.x—
llego a su fin de vida (EOL) el 30 de junio de 2026. Las unicas lineas activamente
soportadas son Spring Boot 4.0.x y 4.1.x, construidas sobre Spring Framework 7.

Esto genera una pregunta de compatibilidad real para `itinerarios-gateway`: ¿Spring
Cloud Gateway ya soporta Spring Boot 4?

## Investigacion realizada
Se confirmo via busqueda web que **Spring Cloud 2025.1.1 ("Oakwood")**, liberado el 29
de enero de 2026, agrega compatibilidad explicita con Spring Boot 4.0.1+, y que la
documentacion oficial de Spring Cloud Gateway ya se construye sobre Spring Framework 7
/ Spring Boot 4.

Tambien se detecto que, desde el release train 2025.0, los artefactos de Spring Cloud
Gateway se renombraron y separaron por stack (WebFlux/WebMVC) — el artefacto clasico
`spring-cloud-gateway-server` quedo deprecado.

## Decision
Usar Spring Boot 4.1.1 en los cuatro microservicios backend, y Spring Cloud 2025.1.1
especificamente en el Gateway.

## Riesgo aceptado y plan de contingencia
Este ecosistema es muy reciente y no pudo validarse ejecutando Maven real contra
Maven Central desde el entorno donde se genero el scaffold inicial (sin acceso de red).
Puntos concretos a verificar en el primer build real, documentados tambien en el
`pom.xml` y `README.md` de `itinerarios-gateway`:

1. Nombre exacto del starter (`spring-cloud-starter-gateway-server-webflux` esperado).
2. Ruta de configuracion YAML de las rutas
   (`spring.cloud.gateway.routes` vs `spring.cloud.gateway.server.webflux.routes`).
3. Ausencia de conflictos de version en `mvn dependency:tree`.

**Plan B:** si alguno de estos puntos falla de forma bloqueante, bajar temporalmente
solo `itinerarios-gateway` a Spring Boot 3.5.x + Spring Cloud 2024.0.x (ultima linea
3.x, aunque en EOL) mientras el ecosistema 4.x/2025.x madura. Los demas microservicios
(Airport, Itinerary) no se ven afectados porque son independientes y no comparten
proceso ni classpath con el Gateway.

## Consecuencias
Se documenta esta decision como de riesgo medio-alto explicitamente, en vez de asumir
silenciosamente que "la ultima version siempre funciona" — coherente con la regla de la
seccion 4 de la especificacion maestra de verificar compatibilidad antes de fijar
versiones en los `pom.xml`.
