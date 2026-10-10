# ADR-004: RabbitMQ como broker de mensajeria

## Estado
Aceptado (implementacion pendiente - Nivel 2, Fase 7).

## Contexto
Se necesita comunicacion asincrona entre Itinerary Service y Notification Service.

## Decision
RabbitMQ en vez de Kafka.

## Justificacion (seccion 26 de la especificacion maestra)
- Open source, facil de ejecutar en Docker.
- Adecuado para el volumen y complejidad de un proyecto academico (un solo tipo de
  evento, un solo consumidor).
- Menor complejidad operativa que Kafka para este alcance (no se necesita
  particionamiento, retencion configurable a largo plazo, ni multiples consumer groups).

## Consecuencias
Si el proyecto creciera a multiples tipos de eventos con necesidad de replay historico
o altisimo throughput, Kafka seria la opcion mas adecuada. Se acepta esa limitacion
porque esta fuera del alcance actual.
