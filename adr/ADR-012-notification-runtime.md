# ADR-012: Notification Service como aplicacion Spring Boot normal (Nivel 2)

## Estado
Aceptado.

## Contexto
La especificacion (secciones 31-32) plantea Notification Service como una funcion
serverless: LocalStack para desarrollo local, AWS Lambda en la nube. Sin embargo, la
misma seccion 32 advierte explicitamente que LocalStack no reproduce automaticamente
la integracion RabbitMQ -> Lambda, y que si se necesita algo ejecutable localmente,
puede optarse por un consumer/adapter normal en vez de simular Lambda de punta a punta.

## Decision
Para Nivel 2, `itinerarios-notification` es una aplicacion Spring Boot estandar con un
`@RabbitListener` de larga duracion, igual en estilo a Airport/Itinerary Service. No se
usa LocalStack ni se simula Lambda todavia.

## Justificacion
- Simular correctamente RabbitMQ -> Lambda localmente (via LocalStack) es
  significativamente mas complejo que un consumer normal, y la especificacion ya
  autoriza esa simplificacion explicitamente.
- El objetivo de Nivel 2 (seccion 96, evidencia) es demostrar el flujo evento ->
  notificacion -> historial funcionando, no la arquitectura serverless en si.
- El codigo de negocio (`NotificationProcessor`) queda desacoplado del mecanismo de
  entrega (`ItineraryEventConsumer`), por lo que migrar a un handler Lambda en Nivel 3
  (seccion 58-59, imagen `public.ecr.aws/lambda/java:21`) implica escribir un nuevo
  adapter de entrada que invoque el mismo `NotificationProcessor`, no reescribir la
  logica de negocio.

## Consecuencias
- Se pospone explicitamente la validacion de la integracion serverless real (AWS
  Lambda + Amazon MQ para RabbitMQ como event source) hasta Nivel 3 / despliegue AWS.
- El Dockerfile y el `docker-compose.yml` de Nivel 2 tratan a Notification Service
  igual que a los demas microservicios (imagen JRE normal, puerto expuesto, sin
  runtime de Lambda).
