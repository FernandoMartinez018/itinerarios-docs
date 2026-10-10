# Evidencias — Nivel 1

Sección 96 de la especificación maestra. **[x] = verificado en ejecución real** (2026-10-09,
logs en `logs/`). La columna "Captura" indica lo que falta para la entrega: la verificación
automática ya pasó, pero la evidencia visual (captura o video) la toma una persona con el stack arriba
(`verificar-stack.ps1 -OmitirBuild -MantenerStack`).

| | Evidencia | Verificación automática | Captura |
|---|---|---|---|
| [x] | Aplicación Angular (login → dashboard) | Selenium | pendiente |
| [x] | Mapa Plotly con aeropuertos | Selenium (pantalla carga) | pendiente — crear 1 itinerario antes (ADR-011) |
| [x] | Swagger Airport (`localhost:8081/swagger-ui.html`) | `verificar-stack` | pendiente |
| [x] | Swagger Itinerary (`localhost:8082/swagger-ui.html`) | `verificar-stack` | pendiente |
| [x] | Crear itinerario desde la UI (validación de aeropuerto) | Selenium + `verificar-stack` (400 si no existe) | pendiente |
| [x] | Consultar itinerario | Selenium (lista) | pendiente (detalle) |
| [x] | Editar itinerario | Selenium | pendiente |
| [x] | Eliminar itinerario | Selenium | pendiente |
| [x] | Filas en PostgreSQL (`airport_db`, `itinerary_db`) | `psql` en `logs/observabilidad-2026-10-09.txt` | pendiente (`psql` o DBeaver) |
| [x] | Tests JUnit en verde (airport 89, itinerary 19, notification 5, gateway 4) | `mvn -B clean verify` | pendiente |
| [x] | `docker compose up --build` con todos los servicios saludables | `verificar-stack` | pendiente (`docker compose ps`) |
| [x] | GitHub Actions CI en verde (5 repos) | `build-test` en verde en el PR #1 → `dev` de airport, itinerary, notification, gateway y frontend (2026-10-09) | pendiente (pestaña Actions) |
| [x] | Frontend `npm test` (Vitest) y build | `verificar-stack` paso 2 | pendiente |
| [x] | Selenium `ItineraryFullFlowE2ETest` en verde | `logs/selenium-e2e-2026-10-09.txt` | pendiente |
