# ADR-006: Flyway para migraciones

## Estado
Aceptado.

## Contexto
Se necesita versionado de esquema de base de datos por microservicio.

## Decision
Flyway, con `ddl-auto: validate` en Hibernate (nunca `update` ni `create`), para que
el esquema real siempre sea el que definen las migraciones versionadas, nunca uno
generado implicitamente por JPA.

## Consecuencias
Cualquier cambio de entidad JPA requiere escribir la migracion correspondiente a mano
(seccion 99/100 de la especificacion maestra) antes de que la aplicacion arranque
correctamente contra una base con `ddl-auto: validate`.
