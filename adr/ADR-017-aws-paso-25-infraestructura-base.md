# ADR-017: Infraestructura AWS base (paso 25) y decisiones de costo

## Estado
Aceptado. Desplegado y probado el 2026-10-09 en la cuenta `783495713599` (us-east-1).
Cuenta en **plan de pago sin créditos**: todo recurso con costo se enciende solo para
probar y se apaga al terminar (instrucción del usuario).

## Contexto
§85 fija el orden IAM → ECR → red → RDS → mensajería → ECS. §48 exige verificar precios y
crear el presupuesto antes que cualquier recurso. Precios consultados con la API oficial
de AWS Pricing (2026-10-09):

| Recurso | Precio us-east-1 |
|---|---|
| RDS PostgreSQL db.t4g.micro Single-AZ | USD 0,016/h |
| Amazon MQ RabbitMQ (mínimo actual: mq.m7g.medium) | USD 0,137/h (≈ USD 100/mes) |
| Fargate | USD 0,04048 vCPU-h + USD 0,004445 GB-h (0,25 vCPU/0,5 GB ≈ USD 0,012/h) |
| IPv4 pública | USD 0,005/h |
| NAT Gateway | ≈ USD 32/mes (no usado) |

## Decisiones

1. **Presupuesto primero** (`00-budget.yaml`): USD 20/mes, alertas al 50 %, 80 % real y
   100 % previsto.
2. **OIDC + ECR** (`01`, `02`): rol de mínimo privilegio solo para push a `itinerarios-*`
   desde el environment `production` de los 5 repos. ECR vacío = USD 0.
3. **Red sin NAT** (`03-network.yaml`): VPC con 2 subredes públicas (RDS exige 2 AZ).
   Las tareas Fargate usarán IP pública en vez de NAT (ahorro de ≈ USD 32/mes). RDS solo
   acepta 5432 desde el SG de la app; `AdminCidr` (/29–/32) solo para pruebas temporales.
   - Por qué /29: la red del usuario sale a internet por un *pool* de IPs (.99 para HTTPS,
     .100 para TCP); un /32 dejaba la conexión rechazada. Sigue siendo lejano a 0.0.0.0/0
     y se elimina al terminar.
4. **RDS** (`04-rds.yaml`): una instancia db.t4g.micro **PostgreSQL 16.15** con las 3 bases
   lógicas, cada una con su usuario (`aws/init-rds-databases.sql`, §56). Resuelve el riesgo
   "local 16 vs RDS 18": misma línea mayor que Compose. Password maestra en Secrets Manager
   (`ManageMasterUserPassword`). `DeletionPolicy: Delete` porque es entorno de pruebas sin
   datos y un snapshot retenido seguiría cobrando.
5. **RabbitMQ como contenedor en ECS, no Amazon MQ.** Amazon MQ ya no ofrece instancia micro
   para RabbitMQ; el mínimo cuesta ≈ USD 100/mes encendido. La imagen `rabbitmq:4` de
   Compose como tarea Fargate cuesta ≈ USD 9/mes encendida. Se pierde HA administrada,
   aceptable para el alcance académico (§26 ya priorizaba simplicidad). Se prueba en el paso 27.

## Prueba ejecutada
- Despliegue de los 5 stacks; RDS disponible en ≈ 7 min.
- `init-rds-databases.sql`: 3 bases con dueños distintos, SSL activo; `airport_user` **no**
  puede conectarse a `itinerary_db`.
- La imagen de `airport-service` corrió contra RDS: Flyway aplicó V1 en AWS, el servicio
  consultó API Colombia y persistió en RDS.
- RDS eliminado y `AdminCidr` retirado. Inventario final sin recursos con costo; RDS estuvo
  encendido ≈ 16 min. Evidencia: `evidence/logs/aws-paso-25-2026-10-09.txt`.

## Hallazgos corregidos durante la prueba
- **API Colombia invierte latitud/longitud en 51 de 52 aeropuertos.** El Mapper ahora las
  corrige (regla geográfica de Colombia) y trata `"N/A"` como código ausente (evita violar
  los UNIQUE de `iata_code`/`icao_code`). Antes el mapa ubicaba mal los aeropuertos.
- **Outbox perdía eventos** (paso 23): reintentaba cada 2 s y abandonaba el evento tras 5
  intentos, así que ~10 s de broker caído perdían la notificación. `V3__add_outbox_next_attempt_at.sql`
  agrega backoff exponencial (2 s → 5 min, 20 intentos) y re-encola los eventos abandonados.
  Demo: 40 s de RabbitMQ caído y el evento se publicó al volver, con 1 sola notificación
  (`evidence/logs/outbox-double-write-2026-10-09.txt`).

## Estado que queda en AWS (USD 0)
Stacks `itinerarios-budget`, `itinerarios-github-oidc`, `itinerarios-ecr` (vacío) e
`itinerarios-network` (sin NAT, sin ENIs). RDS se recrea con `04-rds.yaml` cuando se necesite.

## Addendum — Paso 26: push de imágenes a ECR (2026-10-13)

- Código promovido `feature → dev → main` en los 6 repos (PR #1 y #2), con CI en verde en cada PR.
- En los 5 repos de código: environment `production` y variables `AWS_ACCOUNT_ID`, `AWS_REGION`,
  `AWS_ROLE_NAME` (con `gh`).
- **Primer intento fallido:** `Not authorized to perform sts:AssumeRoleWithWebIdentity`. GitHub ahora
  emite el claim `sub` **inmutable** (`use_immutable_subject: true`):
  `repo:FernandoMartinez018@136218670/itinerarios-airport-service@1362085165:environment:production`.
  `01-github-oidc.yaml` pasa a `repo:${GitHubOwner}@${GitHubOwnerId}/<repo>@*:environment:production`.
  Fijar el ownerId impide que otra cuenta con el mismo nombre asuma el rol; el repoId va con
  comodín porque solo el dueño crea repos bajo su cuenta.
- **Resultado:** `push-image` en verde en los 5 repos; imágenes en ECR con tag = SHA corto de `main`
  y `latest` (≈ 650 MB en total). Se **borraron tras verificar** (decisión de costo: USD 0) y se
  regeneran con el próximo push a `main`. Evidencia: `evidence/logs/aws-paso-26-ecr-2026-10-13.txt`.
- Pendiente menor: el escaneo básico de ECR quedó en `None` pese a `ScanOnPush`; no se activa
  Inspector (escaneo avanzado, con costo).
