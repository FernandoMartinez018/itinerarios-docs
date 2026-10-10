# ADR-014: OpenTelemetry via agente Java (auto-instrumentacion) + Prometheus/Grafana

## Estado
Aceptado e implementado (Fase 11).

## Contexto
La seccion 61-62 exige trazas distribuidas (OpenTelemetry + Jaeger) y metricas
(Prometheus + Grafana) en los 4 microservicios backend (Airport, Itinerary,
Notification, Gateway).

## Decision: trazas via agente Java, no SDK manual
Se uso el agente Java de OpenTelemetry (`opentelemetry-javaagent.jar`,
descargado en el Dockerfile de cada servicio JVM y activado con
`JAVA_TOOL_OPTIONS=-javaagent:...`) en vez de instrumentar manualmente el codigo
con el SDK de OpenTelemetry.

### Justificacion
- Cero cambios de codigo: el agente instrumenta por bytecode Spring MVC, WebFlux,
  Spring Data JPA, RestClient, Spring AMQP (RabbitMQ), y mas, automaticamente.
- Reduce el riesgo de que la instrumentacion manual quede inconsistente entre los
  4 servicios (cada uno tendria que replicar la misma configuracion de SDK).
- Es el enfoque que la propia documentacion oficial de OpenTelemetry recomienda
  como punto de partida antes de instrumentacion manual selectiva.

### Contrapartida aceptada
Se descarga siempre la ultima version estable del agente
(`.../releases/latest/download/opentelemetry-javaagent.jar`) en vez de fijar una
version exacta, para no quedar con una version desactualizada al momento de
escribir este documento. Para un entorno de produccion real, se recomienda fijar
una version exacta y actualizarla deliberadamente, no depender de "latest".

## Decision: metricas via Micrometer/Prometheus, no via OTel Metrics
Cada servicio expone `/actuator/prometheus` (Micrometer + micrometer-registry-
prometheus), scrapeado directamente por Prometheus. El agente de OpenTelemetry
tiene `OTEL_METRICS_EXPORTER=none` explicitamente, para no correr dos pipelines
de metricas en paralelo que terminarian reportando datos parcialmente
duplicados o inconsistentes entre si.

## Coexistencia de dos identificadores de correlacion
El `X-Correlation-ID` propio (implementado desde Nivel 1, seccion 63) y el
`trace_id`/`span_id` que el agente de OpenTelemetry agrega automaticamente al MDC
de logging conviven sin unificarse. Sirven a propositos distintos: el
`correlationId` es para buscar logs de texto de un request especifico entre
servicios; el `trace_id` es para abrir la traza visual completa en Jaeger.
Unificarlos hubiera significado reemplazar el filtro de correlacion ya probado
de Nivel 1 sin necesidad real -- ambos IDs aparecen en la misma linea de log.

## Topologia

```text
[4 microservicios] --OTLP/gRPC--> [otel-collector] --OTLP/gRPC--> [Jaeger]
[4 microservicios] --scrape-------------------------------------> [Prometheus] --> [Grafana]
```

Se opto por interponer un otel-collector en vez de que cada servicio exporte
directo a Jaeger, porque es la topologia estandar recomendada (permite agregar
procesamiento -- muestreo, enriquecimiento, multiples destinos -- sin tocar la
configuracion de cada microservicio despues).

## Verificacion pendiente
No se pudo levantar el stack real en este entorno (sin red completa hacia
GitHub/Docker Hub) para confirmar que el agente descarga correctamente en build
time, que el Collector enruta bien hacia Jaeger, y que Prometheus scrapea los 4
targets sin error de resolucion DNS dentro de la red de Docker Compose. Mismo
tipo de riesgo documentado en ADR-010 y ADR-013 -- verificar en el primer
`docker compose up` real.
