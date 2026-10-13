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

## Addendum — Paso 27: microservicios en ECS Fargate (2026-10-13)

Plantilla `05-ecs.yaml` (cfn-lint OK). Diseño para encender, probar y borrar:

| Pieza | Decisión | Motivo |
|---|---|---|
| Cómputo | 1 servicio ECS Fargate por microservicio (§88): gateway, airport, itinerary, notification (0,25 vCPU / 1 GB) | Mínimo que arranca Spring Boot |
| Mensajería/cache | RabbitMQ y Redis como tareas Fargate (0,25 vCPU / 0,5 GB), imágenes de ECR Public | Amazon MQ ≈ USD 100/mes; ECR Public evita el rate limit anónimo de Docker Hub |
| Descubrimiento | Cloud Map DNS privado `*.itinerarios.local` | Sin sidecars; Route 53 no cobra zonas borradas antes de 12 h |
| Secretos | SSM Parameter Store SecureString `/itinerarios/*` (gratis), leídos por el execution role | Secrets Manager cobra por secreto |
| Acceso | Solo el Gateway acepta 8080 y solo desde AdminCidr; sin ALB ni NAT | Prueba barata; ALB queda para un despliegue permanente |
| Perfil | `SPRING_PROFILES_ACTIVE=prod` (sin defaults de credenciales) | §67 |
| Observabilidad | Logs a CloudWatch (retención 1 día); agente OTel desactivado (no hay collector en AWS aún) | §85: la observabilidad cloud va al final |

Costo estimado encendido ≈ USD 0,13/h (Fargate ≈ 0,08, IPv4 públicas ≈ 0,03, RDS 0,016).

**Prueba (encendido 09:40–10:15 aprox.):**
- Imágenes regeneradas en ECR re-ejecutando el CI de `main` (push-image por OIDC).
- RDS + `init` de las 3 bases (con Python/psycopg porque Docker Desktop no arrancaba; misma lógica que `init-rds-databases.sql`).
- 6 servicios ECS 1/1 en ≈ 7 min.
- Vía la IP pública del Gateway: health UP; `GET /api/airports/iata/BOG` 200 (El Dorado, coordenadas corregidas);
  `POST /api/itineraries` 201; aeropuerto inexistente 400; **notificación generada vía RabbitMQ en ECS**;
  listar 200; borrar 204.
- Correlation ID `aws-ecs-001` visible en CloudWatch en gateway, itinerary y airport (§63).
- Evidencia: `evidence/logs/aws-paso-27-ecs-2026-10-13.txt`. Después se borró ECS, RDS, los parámetros SSM y las imágenes ECR, y se retiró AdminCidr.

**Pendiente para un despliegue permanente:** ALB delante del Gateway, frontend con URL de Gateway
inyectada en arranque (hoy `localhost:8080` en el bundle), job `deploy` del CI (task-definition.json y
`DEPLOY_ENABLED`), OTel → CloudWatch/X-Ray.

## Addendum — Paso 28: Notification como AWS Lambda (2026-10-13)

Plantilla `06-lambda.yaml` (cfn-lint OK): `SendNotificationFunction`, runtime `java21`, 1024 MB,
timeout 120 s, dentro de la VPC con el SG de la app (llega a RDS sin abrirlo, §57), rol con solo
`AWSLambdaVPCAccessExecutionRole`, logs con retención de 1 día.

Decisiones:
- **ZIP vía S3** (clases + `lib/*.jar`, 61,9 MB, construido con Maven local): supera los 50 MB de
  subida directa. La imagen de contenedor (`public.ecr.aws/lambda/java:21`, §58) queda como
  alternativa; hoy Docker Desktop no arranca en la máquina de desarrollo.
- **Password de la base:** parámetro `NoEcho` leído de SSM al desplegar → variable de entorno cifrada
  por Lambda. Concesión de prueba; para producción usar la *AWS Parameters and Secrets Lambda
  Extension* con un VPC endpoint (≈ USD 0,02/h) y leerla en runtime.
- Invocación directa con un `ItineraryCreatedEvent` (como en LocalStack, §32: RabbitMQ → Lambda no
  se simula; la arquitectura conceptual se mantiene).

Prueba (encendido ≈ 10:22–10:45):
- Invocación 1 (cold start): `PROCESSED`, 21 s de pared, 18,7 s facturados (Spring arranca dentro del handler).
- Invocación 2 con **el mismo evento**: `PROCESSED`, 8 ms facturados, log "duplicado ignorado".
- `notification_history`: **1 sola fila SENT** (idempotencia, ADR-015) y Flyway V1+V2 aplicados en RDS.
- Memoria máxima usada: 364 MB.
- Evidencia: `evidence/logs/aws-paso-28-lambda-2026-10-13.txt`. Después se borró Lambda, el bucket S3,
  RDS y los parámetros SSM, y se retiró AdminCidr.

Mejora sugerida: inicializar el contexto de Spring en la fase *init* y activar **SnapStart** (Java,
sin costo extra) para bajar el cold start de ~19 s a ~1-2 s.

## Addendum — Paso 29: pruebas cloud integradas (2026-10-13)

`aws/run-cloud-tests.sh` enciende todo (red con AdminCidr, SSM, RDS + init, imágenes vía CI, ECS,
Lambda), ejecuta 13 pruebas y **apaga siempre** (`trap EXIT`) con un inventario final.

Resultado del intento 3: **13 OK, 0 fallas** (`evidence/logs/aws-paso-29-cloud-tests-2026-10-13.txt`):
health; BOG = El Dorado con coordenadas en Colombia; 201/400/400; Outbox PUBLISHED; notificación SENT
vía RabbitMQ en ECS; Lambda con el mismo eventId sin duplicar (idempotencia compartida ECS + Lambda);
Airport caído → 503 en 341 ms con el breaker abierto → se recupera y vuelve a 201; Correlation ID en
gateway, itinerary y airport; PUT 200 y DELETE 204.

Errores del script corregidos en el camino (los intentos 1 y 2 se conservan como evidencia):
1. `curl | grep -q` con `pipefail`: el SIGPIPE de curl marcaba falla con la respuesta correcta.
2. Las pruebas empezaban con el Gateway UP pero los servicios aún arrancando (los fallos tempranos abrían el breaker): ahora esperan 200 de airport, itinerary y notification.
3. RDS no se puede borrar mientras ECS/Lambda importan su endpoint: el apagado ahora es secuencial (el intento 1 dejó RDS encendido ~50 min de más).
4. La red del usuario balancea entre **dos ISP** (HTTPS y TCP salen por enlaces distintos): `AdminCidr2` en 03/05 y muestreo de IPs de salida (aborta sin encender nada si hay más de 2).
5. `filter-log-events` sin ventana de tiempo, con logs en vivo, pagina por `nextToken` ~60 min: ahora usa `--start-time/--end-time` y `--max-items`.

Costo de las tres corridas ≈ USD 0,50 en total (la mayor parte, por las esperas de los errores 3 y 5).
Duración esperada de una corrida sana: ≈ 25 min encendido + ≈ 20 min de apagado (ENIs de Lambda).
