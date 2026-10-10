# ADR-001: Arquitectura de microservicios

## Estado
Aceptado.

## Contexto
La especificación académica exige explícitamente una arquitectura de microservicios,
con bases de datos independientes y comunicación HTTP + asíncrona.

## Decisión
Tres bounded contexts, cada uno como microservicio independiente con su propia base
de datos: Airport, Itinerary, Notification. Un Gateway como único punto de entrada.

## Consecuencias
- Positivas: aislamiento de fallos, despliegue independiente, alineado con el
  requisito académico.
- Negativas: mayor complejidad operativa (múltiples bases, múltiples pipelines de CI,
  necesidad de correlación de logs entre servicios) frente a un monolito, aceptada
  conscientemente porque es un requisito explícito, no una elección técnica libre.
