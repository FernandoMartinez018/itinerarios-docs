# Pruebas E2E — Selenium

Implementadas en `itinerarios-infrastructure/e2e-tests` (módulo Maven independiente,
Java 21, Selenium 4.48.0, JUnit 5). Ver el README de ese módulo para instrucciones de
ejecución.

## Por qué viven en itinerarios-infrastructure y no en itinerarios-frontend

Estas pruebas ejercitan el **sistema completo integrado** (Angular + Gateway + Airport
Service + Itinerary Service + PostgreSQL, todo corriendo vía Docker Compose), no la
lógica interna de un solo repositorio. Colocarlas junto a la orquestación de
infraestructura evita la ambigüedad de "¿esto es un test de frontend o de backend?" —
no es ninguno de los dos, es un test de sistema.

## Flujo cubierto (sección 38)

```text
Login -> Dashboard -> Aeropuertos (mapa) -> Crear itinerario
      -> Validar aeropuerto (implícito vía backend) -> Guardar
      -> Verificar en la lista -> Editar -> Eliminar
```

Un único test end-to-end (`ItineraryFullFlowE2ETest`) cubre esta cadena completa. El
tramo de RabbitMQ/Notification del mismo diagrama de la sección 38 se agrega en
Nivel 2, cuando exista Notification Service.

## Selenium Manager

Selenium 4.6+ resuelve automáticamente el binario de ChromeDriver compatible con el
Chrome instalado en la máquina — no se agregó WebDriverManager como dependencia extra,
y no hace falta configurar el PATH manualmente.

## Ejecución en CI

`itinerarios-infrastructure/.github/workflows/e2e.yml` hace checkout de los cuatro
repositorios necesarios como hermanos (replicando la estructura de carpetas local),
levanta el stack con `docker compose up --build`, espera a que el Gateway responda en
`/actuator/health`, corre Selenium en modo headless, y siempre apaga el stack al final
(incluso si el test falla), volcando los logs de todos los contenedores para
diagnóstico.
