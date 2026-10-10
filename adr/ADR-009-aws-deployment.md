# ADR-009: CI/CD con GitHub Actions + OIDC + ECR, despliegue a ECS

## Estado
Parcialmente implementado: pipelines escritos (Fase 13); infraestructura AWS y
despliegue real pendientes (Fases 14-15).

## Contexto
La especificacion (secciones 46-55, 82-90) pide un flujo Git push -> tests -> build
-> imagen Docker -> ECR -> despliegue, con imagenes identificadas de forma inmutable
para permitir rollback, y sin credenciales AWS de larga duracion en GitHub.

## Decision
- GitHub Actions como motor de CI/CD (ya usado para CI en Nivel 1).
- Autenticacion a AWS con OIDC (`aws-actions/configure-aws-credentials`), asumiendo
  un rol IAM de minimo privilegio. No se guardan `AWS_ACCESS_KEY_ID` /
  `AWS_SECRET_ACCESS_KEY`.
- Imagenes en ECR, una por servicio, con tag inmutable = SHA corto del commit;
  `latest` solo como tag auxiliar.
- Despliegue a ECS con `amazon-ecs-render-task-definition` +
  `amazon-ecs-deploy-task-definition`, seguido de un smoke test.
- Job `push-image` y `deploy` protegidos con `environment: production`, para poder
  exigir aprobacion manual sin modificar los workflows.

## Decision de secuenciacion
El orden de la seccion 111 coloca "completar CI/CD" (24) antes de "crear
infraestructura AWS" (25-26). Para que eso sea coherente, los jobs `push-image` y
`deploy` se escribieron ahora pero `deploy` queda desactivado por defecto
(`DEPLOY_ENABLED`) y `push-image` fallara hasta que existan el proveedor OIDC, el
rol y los repositorios ECR. Se prefirio esto a omitir esos jobs, porque asi el
pipeline queda revisado y versionado junto al codigo, y activar el despliegue se
reduce a crear recursos y definir variables.

## Alternativas descartadas
- Access Keys permanentes como secretos de GitHub: descartado por la seccion 51.
- Un pipeline de despliegue unico en `itinerarios-infrastructure` para todos los
  servicios: simplificaria la configuracion pero acopla el despliegue de cada
  microservicio al de los demas, contra la independencia de despliegue buscada en
  ADR-001. Cada repo conserva su propio pipeline.

## Consecuencias
- Los cinco workflows repiten la misma estructura (generada desde una plantilla
  comun). Si cambia el patron habra que actualizar los cinco; una mejora futura
  seria extraerlo a un workflow reutilizable (`workflow_call`).
- Costos AWS: ECR, ECS y RDS no son gratuitos (secciones 48 y 91). Revisar precios
  y Free Tier vigentes y crear alertas de costo antes de la Fase 14.
