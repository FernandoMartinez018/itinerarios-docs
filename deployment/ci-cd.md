# CI/CD (Fase 13)

Cada repositorio de codigo (`airport-service`, `itinerary-service`, `notification`,
`gateway`, `frontend`) tiene `.github/workflows/ci.yml` con tres jobs:

| Job | Cuando corre | Que hace |
|---|---|---|
| `build-test` | push y pull request a `main`/`qa`/`dev` (ramas reales del equipo) | Tests + build + `docker build` (sin credenciales AWS) |
| `push-image` | solo push a `main`, tras `build-test` | Autentica con AWS via OIDC, construye y sube la imagen a ECR con tag = SHA corto del commit (y `latest` como auxiliar) |
| `deploy` | solo si la variable `DEPLOY_ENABLED=true` | Renderiza la task definition con la nueva imagen, despliega a ECS, espera estabilidad y ejecuta un smoke test |

## Estado real

- `build-test` es plenamente funcional.
- `push-image` y `deploy` estan **escritos pero no se pueden ejecutar todavia**:
  dependen de recursos AWS que se crean en las Fases 14-15 (IAM, ECR, ECS). Hasta
  entonces `push-image` falla en el paso "Configure AWS credentials". Es esperado.
- `deploy` esta desactivado por defecto (`DEPLOY_ENABLED` sin definir) y necesita
  `.aws/task-definition.json` en cada repo, que se crea en la Fase 15.

## Configuracion requerida en GitHub (por repositorio)

Variables de repositorio (Settings > Secrets and variables > Actions > Variables).
Ninguna es un secreto: no hay Access Keys almacenadas (seccion 51/90).

| Variable | Ejemplo | Usada por |
|---|---|---|
| `AWS_ACCOUNT_ID` | `123456789012` | push-image, deploy |
| `AWS_REGION` | `us-east-1` | push-image, deploy |
| `AWS_ROLE_NAME` | `github-actions-itinerarios` | push-image, deploy |
| `DEPLOY_ENABLED` | `true` | deploy (activarla solo en Fase 15) |
| `ECS_CLUSTER` | `itinerarios` | deploy |
| `ECS_SERVICE_<SERVICIO>` | `itinerarios-airport` | deploy |
| `SMOKE_URL_<SERVICIO>` | `https://.../actuator/health` | deploy |

Ademas, crear un Environment llamado `production` y, si se desea, agregarle
required reviewers: asi un push a `main` queda esperando aprobacion manual antes de
subir imagen o desplegar.

## Configuracion requerida en AWS (Fase 14, secciones 51-53)

1. IAM > Identity providers > OpenID Connect:
   URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
2. Crear el rol `github-actions-itinerarios` con esta trust policy (reemplazar
   `<OWNER>` y `<ACCOUNT_ID>`); limita el rol a la rama `main` de los repos del proyecto:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": [
          "repo:<OWNER>/itinerarios-airport-service:environment:production",
          "repo:<OWNER>/itinerarios-itinerary-service:environment:production",
          "repo:<OWNER>/itinerarios-notification:environment:production",
          "repo:<OWNER>/itinerarios-gateway:environment:production",
          "repo:<OWNER>/itinerarios-frontend:environment:production"
        ]
      }
    }
  }]
}
```

   Los jobs usan `environment: production`, por lo que GitHub emite el claim `sub`
   como `repo:<owner>/<repo>:environment:production`; la trust policy de arriba ya
   usa ese formato. La plantilla lista para desplegar esta en
   `itinerarios-infrastructure/aws/cloudformation/01-github-oidc.yaml`. Verificar el
   claim real en el primer run.

3. Politica de permisos con minimo privilegio (seccion 53): para `push-image`,
   solo acciones de ECR sobre los 5 repositorios `itinerarios-*`
   (`ecr:GetAuthorizationToken` sobre `*`, y `BatchCheckLayerAvailability`,
   `InitiateLayerUpload`, `UploadLayerPart`, `CompleteLayerUpload`, `PutImage`
   sobre esos repositorios). Para `deploy` se agregan, en Fase 15, `ecs:*` acotado
   a los servicios concretos, `ecs:DescribeServices`, `ecs:RegisterTaskDefinition`
   e `iam:PassRole` limitado a los roles de tarea. Nunca permisos administrativos.

## Rollback (seccion 84)

Como cada imagen queda identificada por el SHA del commit, volver atras es
re-desplegar el tag anterior: en el job `deploy`, o manualmente con
`aws ecs update-service` apuntando a la task definition previa. Por eso `latest`
nunca se usa para desplegar, solo como referencia secundaria.

## Riesgos / puntos a verificar

- Las versiones de las acciones (`aws-actions/configure-aws-credentials@v4`,
  `amazon-ecr-login@v2`, `amazon-ecs-render-task-definition@v1`,
  `amazon-ecs-deploy-task-definition@v2`) son las que conozco como vigentes; no se
  pudieron validar contra el marketplace en este entorno.
- El claim `sub` de OIDC con `environment` (ver nota arriba): formato documentado por GitHub, aun no probado contra una cuenta real.
- El pipeline `e2e.yml` de `itinerarios-infrastructure` hace checkout de los otros
  repos como hermanos; si son privados necesita un token con acceso de lectura.
