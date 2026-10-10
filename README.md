# itinerarios-docs

Documentacion tecnica del Sistema de Itinerarios Personales.

## Indice

- **architecture/** — diagramas PlantUML: overview, componentes, clases, secuencia,
  despliegue, context map (DDD), flujo asincrono (Nivel 2).
- **ddd/** — bounded contexts, lenguaje ubicuo, agregados.
- **database/** — modelo relacional y decisiones de diseño de base de datos.
- **api/** — contrato REST de cada microservicio.
- **events/** — contrato AsyncAPI de `ItineraryCreatedEvent` (Nivel 2, definido de
  antemano para que Publisher y Consumer no diverjan cuando se implementen).
- **deployment/** — guias de ejecucion local, Docker, y placeholder de AWS (Nivel 3).
- **testing/** — que cubren las pruebas unitarias y las pruebas E2E con Selenium.
- **adr/** — decisiones arquitectonicas, incluyendo los riesgos de version que se
  detectaron durante la construccion real (ADR-010) y la estrategia de sincronizacion
  de aeropuertos (ADR-011).
- **evidence/** — checklist de capturas/videos pendientes para demostrar Nivel 1.

## Estado del proyecto (actualizado a esta sesion)

| Componente | Estado |
|---|---|
| itinerarios-airport-service | Nivel 1 y Nivel 2 completos en codigo |
| itinerarios-itinerary-service | Nivel 1, Nivel 2 y Fase 12 (Transactional Outbox, Nivel 3) completos en codigo |
| itinerarios-gateway | Nivel 1 y Nivel 2 completos en codigo, con riesgo de version documentado (ADR-010) |
| itinerarios-frontend | Nivel 1 completo, pendiente de `npm install` real |
| itinerarios-infrastructure | docker-compose.yml completo (Nivel 1 + Nivel 2) + e2e-tests generados |
| itinerarios-notification | Nivel 2 completo en codigo (Fase 8), corre como Spring Boot normal (ADR-012) |
| itinerarios-docs | Este documento |

Nivel 1: completo en codigo y documentacion.
Nivel 2: **completo en codigo** -- Fase 7 (RabbitMQ), Fase 8 (Notification),
Fase 9 (Redis), Fase 10 (Resiliencia) y Fase 11 (Observabilidad).
Nivel 3: **Fase 12 (Transactional Outbox) completa** en `itinerary-service`
(ver ADR-008). **Fase 13 (CI/CD)**: pipelines escritos en los 5 repos de codigo
(build-test funcional; push a ECR via OIDC y deploy a ECS escritos pero inertes
hasta crear la infraestructura AWS -- ver `deployment/ci-cd.md` y ADR-009).
**Fase 14 iniciada**: plantillas CloudFormation de presupuesto, OIDC/IAM y ECR (validadas con cfn-lint, sin desplegar). Faltan VPC/RDS/ECS/Lambda (Fase 14-15, con costo real) y los componentes
adicionales segun prioridad (seccion 108).

## Auditoria 2026-10-09 (pasos 1-24)

Ver ADR-015. Corregido: dependencias Spring Boot 4 en los 4 poms, Gateway en Boot
4.0.x con Spring Cloud 2025.1.1, runner de pruebas del frontend (Vitest, verificado
en verde), LocalStack en Compose (perfil `serverless`), indice de idempotencia en
`notification_history`, documentacion desalineada.

Pendiente antes de cerrar Nivel 1 y Nivel 2: ejecutar `verificar-stack.ps1`
(en `itinerarios-infrastructure`), tests MockMvc, handler Lambda de Notification
y las evidencias de `evidence/`.

## Build real 2026-10-09 (ADR-016)

Los 4 servicios backend compilan y pasan `mvn -B clean verify` (airport 88 tests,
itinerary 19, notification 4, gateway 2). Se corrigieron: resilience4j 2.4.0, migracion
a Jackson 3 (sin ella el contexto no arrancaba), cache Redis tipada (ClassCastException
en HIT), codigos HTTP de errores de Spring MVC, idempotencia de Notification, tests
MockMvc, perfil `prod`, agente OTel fijado y tramo de notificacion en Selenium.
Stack verificado en ejecucion el mismo dia (addendum de ADR-016): flujo por Gateway,
Selenium, Lambda en LocalStack y observabilidad. Estado de evidencias en `evidence/`.

## Riesgos de verificacion acumulados (a resolver en el primer build/deploy real)

Ningun componente de este proyecto se ejecuto realmente en el entorno donde se
genero -- no hubo acceso de red completo a Maven Central, npm, Docker Hub ni
GitHub Releases. Los riesgos concretos, documentados cada uno en su ADR
correspondiente:

1. **ADR-010**: compatibilidad exacta de Spring Cloud Gateway 2025.1.1 con
   Spring Boot 4.1.1 (nombre de artefactos, ruta de configuracion YAML).
2. **ADR-013**: resolucion de `resilience4j-spring-boot4` sin conflictos de
   dependencias contra Spring Boot 4.1.1.
3. **ADR-014**: descarga del agente OpenTelemetry en build time, y conectividad
   real entre los 4 microservicios, el otel-collector, Jaeger y Prometheus
   dentro de la red de Docker Compose.
4. ~~Frontend: paquetes npm de Angular 22~~ — verificado: Angular 22.2.2, TypeScript 6.0.3, build y tests en verde (ADR-015).

Ninguno de estos riesgos invalida el diseño; todos son de "verificar el nombre
exacto/version exacta contra el registro real", no de arquitectura.
