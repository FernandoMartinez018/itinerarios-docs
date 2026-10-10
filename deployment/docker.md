# Docker

Cada microservicio (`airport-service`, `itinerary-service`, `gateway`) usa el mismo
patrón de Dockerfile multi-stage:

1. **Etapa build:** `maven:3.9-eclipse-temurin-21`, compila y empaqueta el jar.
2. **Etapa runtime:** `eclipse-temurin:21-jre-alpine`, copia solo el jar final.

El frontend usa un patrón equivalente pero con Node/nginx:

1. **Etapa build:** `node:22-alpine`, corre `npm install` + `ng build --configuration production`.
2. **Etapa runtime:** `nginx:alpine`, sirve el build estático con fallback de rutas a
   `index.html` (necesario para el routing de Angular).

Ningún Dockerfile hardcodea secretos ni URLs de entorno — todo llega vía variables de
entorno inyectadas por `docker-compose.yml` (sección 41).

Ver `deployment/local.md` y `itinerarios-infrastructure/README.md` para cómo se
orquestan todos juntos.
