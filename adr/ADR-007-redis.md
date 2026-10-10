# ADR-007: Redis como cache

## Estado
Aceptado e implementado (Fase 9).

## Contexto
API Colombia tiene un limite real de 60 solicitudes/minuto por IP. Sin cache, cualquier
trafico moderado agotaria ese limite rapidamente.

## Decision
Redis con patron Cache-Aside en Airport Service (seccion 27), implementado via
`@Cacheable` de Spring sobre `findById` y `findByIataCode`, con `RedisCacheManager`
configurado con TTL (default 60 minutos, `AIRPORTS_CACHE_TTL_MINUTES`) y serializacion
JSON (`GenericJackson2JsonRedisSerializer`). Los resultados de error (excepciones) nunca
se cachean.

Ademas, se usara como backend del `RequestRateLimiter` oficial de Spring Cloud Gateway
en Nivel 2 (reemplazando el rate limiting en memoria de Nivel 1, que no es apto para
multiples instancias del Gateway - ver comentario en `SimpleRateLimitGlobalFilter`) -
esta segunda parte queda pendiente, es una mejora futura sobre el Gateway.

## Consecuencias
Introduce un nuevo componente de infraestructura con estado a operar (aunque sea
in-memory y sin persistencia obligatoria para este caso de uso), y una nueva fuente
posible de inconsistencia (cache desactualizado) que debe manejarse con una politica de
expiracion razonable.
