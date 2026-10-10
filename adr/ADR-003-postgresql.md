# ADR-003: PostgreSQL como motor relacional

## Estado
Aceptado.

## Contexto
La especificación exige persistencia relacional por microservicio, con migraciones.

## Decisión
PostgreSQL 16 (imagen `postgres:16-alpine`), una instancia por microservicio en
Docker Compose, gestionada con Flyway.

## Justificación
Open source, sin costo de licencia, amplio soporte en Spring Data JPA, y soportado
nativamente como servicio administrado en AWS (RDS) para el despliegue de Nivel 3
(sección 56-57), sin necesidad de migrar de motor entre desarrollo local y cloud.

## Consecuencias
Ninguna negativa relevante para el alcance del proyecto.
