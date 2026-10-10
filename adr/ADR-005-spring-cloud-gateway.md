# ADR-005: Spring Cloud Gateway como API Gateway

## Estado
Aceptado.

## Contexto
Se necesita un único punto de entrada con routing, CORS, rate limiting y correlation id.

## Decision
Spring Cloud Gateway (stack reactivo WebFlux), sobre Spring Cloud 2025.1.1 ("Oakwood").

## Justificacion
Se mantiene el ecosistema Spring de todo el backend en vez de introducir una tecnologia
distinta (ej. Kong, NGINX como gateway de aplicacion, Traefik) solo para el Gateway,
reduciendo la superficie de tecnologias distintas que el equipo debe mantener.

Ver ADR-010 para la decision especifica de version (Spring Boot 4.x / Spring Cloud
2025.1.1) y los riesgos de que ese ecosistema sea muy reciente al momento de construir
el proyecto.

## Consecuencias
El Gateway usa un modelo reactivo (WebFlux) mientras el resto de los microservicios
son Spring MVC bloqueante. Es una inconsistencia de estilo de programacion aceptada
porque el Gateway es infraestructura de borde, no logica de negocio, y el estandar de
facto para Spring Cloud Gateway es reactivo.
