# ADR-002: Spring por capas en vez de arquitectura hexagonal completa

## Estado
Aceptado.

## Contexto
La consigna académica original menciona arquitectura hexagonal. El equipo decidió
explícitamente usar una arquitectura Spring tradicional por capas
(`Controller -> Service -> Repository -> Database`, y
`Controller -> Service -> Adapter -> External API` para integraciones externas).

## Decisión
No implementar puertos y adaptadores de forma completa (sin interfaces de dominio
separadas de la infraestructura, sin inversión de dependencias estricta). El patrón
Adapter se mantiene, pero solo para la integración con API Colombia — no como
arquitectura interna general de cada capa.

## Justificación
Para el alcance del proyecto (CRUD + una integración externa + un evento), la
arquitectura hexagonal completa agrega una capa de indirección (puertos, casos de uso
explícitos, mappers adicionales) que no se traduce en beneficio real dado el tamaño del
dominio, y consume tiempo de desarrollo que compite con los demás requisitos
(observabilidad, resiliencia, CI/CD, Nivel 3).

## Consecuencias
- Positivas: menos código repetitivo, curva de aprendizaje más baja, velocidad de
  entrega mayor para el alcance real del proyecto.
- Negativas: si el dominio creciera significativamente (más agregados, más reglas de
  negocio complejas), la falta de una separación estricta de dominio dificultaría el
  testing aislado de reglas de negocio puras. Se acepta ese riesgo porque el dominio
  actual (dos agregados simples) no lo justifica.
