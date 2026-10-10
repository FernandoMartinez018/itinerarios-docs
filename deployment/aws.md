# Despliegue AWS (Nivel 3 — en progreso)

Este documento se completa cuando se ejecute la Fase 15 (sección 111, punto 25).

Referencia de diseño ya acordada en la especificación maestra (secciones 47-59):

- ECR para las imágenes de `airport-service`, `itinerary-service`, `gateway`,
  `notification`, `frontend`.
- ECS para correr los microservicios contenedorizados.
- RDS PostgreSQL (una instancia por base lógica, o una instancia compartida con
  aislamiento lógico si se prioriza reducir costos — ver sección 56).
- Lambda (imagen de contenedor `public.ecr.aws/lambda/java:21`) para Notification.
- GitHub Actions + OIDC para desplegar sin credenciales AWS de larga duración
  almacenadas como secretos.

**Regla de costos (sección 91/104):** antes de crear cualquier recurso AWS real, revisar
precio actual, Free Tier/créditos vigentes, definir presupuesto, y crear alertas de
costo. Estas condiciones cambian con el tiempo y deben reverificarse en el momento real
del despliegue, no asumirse desde este documento.

## Estado (Fase 14, primera parte)

Hay plantillas CloudFormation validadas con `cfn-lint`, aun sin desplegar, en
`itinerarios-infrastructure/aws/`: alerta de presupuesto, OIDC + rol IAM de minimo
privilegio y 5 repositorios ECR (recursos gratuitos o de costo marginal). Siguen sin
crearse VPC, RDS, RabbitMQ gestionado, ECS/ALB, Lambda y Redis: tienen costo real y
requieren aprobacion y revision de precios vigentes antes de definirse.
