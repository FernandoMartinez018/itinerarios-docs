# Despliegue Local

## Requisitos

- Docker + Docker Compose.
- Los 7 repositorios clonados **como hermanos**, en la misma carpeta padre (ver
  `itinerarios-infrastructure/README.md` para el layout exacto).

## Pasos

```bash
cd itinerarios-infrastructure
cp .env.example .env
# editar .env con contraseñas reales

docker compose up --build
```

## Servicios

| Servicio | URL |
|---|---|
| Frontend | http://localhost:4200 |
| Gateway (unico backend que el frontend deberia usar) | http://localhost:8080 |
| Airport Service Swagger (debug) | http://localhost:8081/swagger-ui.html |
| Itinerary Service Swagger (debug) | http://localhost:8082/swagger-ui.html |
| Notification Service Swagger (debug) | http://localhost:8083/swagger-ui.html |
| RabbitMQ Management UI | http://localhost:15672 |
| Jaeger | http://localhost:16686 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 |
| LocalStack (perfil `serverless`) | http://localhost:4566 |

## Estado por nivel

Nivel 1 y Nivel 2 (RabbitMQ, Notification, Redis, Resiliencia, Observabilidad)
completos en codigo — ver `docker-compose.yml` en `itinerarios-infrastructure`.
Nivel 3: Transactional Outbox y pipelines CI/CD escritos; AWS en curso (presupuesto,
OIDC y ECR en CloudFormation). LocalStack disponible con
`docker compose --profile serverless up -d localstack`. Ver ADR-015 para las
correcciones de la auditoría y lo que falta verificar.
