# ADR-018: Login de usuarios con Keycloak vía el Gateway (Backend-for-Frontend)

## Estado
Aceptado (2026-10-21). Verificado en local: tests del Gateway (12) y del frontend (13),
flujo OIDC completo con `curl` y `verificar-stack.ps1`, y Selenium con login real en Keycloak.

## Contexto
En el paso 31, Keycloak se usó solo para seguridad **entre servicios** (Client Credentials). El
frontend tenía un "login" que pedía un nombre, sin autenticación. El usuario pidió que el login
use Keycloak, con un usuario de prueba cuyo correo y nombre viven fuera de Git (`.env` / SSM).

Restricción clave: la demo en AWS se sirve por **HTTP con IP pública** (sin ALB ni dominio, por
costo, ADR-017). En un contexto no seguro el navegador no expone `crypto.subtle`, que necesitan
las librerías OIDC de SPA (`keycloak-js`, `angular-auth-oidc-client`) para PKCE S256.

## Decisión
**Patrón Backend-for-Frontend en el Gateway** (recomendado por la guía *OAuth 2.0 for
Browser-Based Applications* del IETF):

1. El Gateway es cliente OIDC confidencial (`oauth2Login`, Authorization Code + PKCE S256, mismo
   cliente `gateway` del paso 31 con `standardFlowEnabled`). Los tokens se quedan en el Gateway;
   el navegador solo recibe la cookie `SESSION` (HttpOnly).
2. `/api/**` exige sesión y responde **401** sin ella (sin redirigir: es una API). `/auth/me`
   expone nombre, usuario y correo del ID token. Esto no es lógica de negocio (§33).
3. **Mismo origen para todo:** nginx del frontend reenvía `/api`, `/auth`, `/oauth2`,
   `/login/oauth2` y `/logout` al Gateway, y `/realms` y `/resources` a Keycloak. Así la
   `authorization-uri` y las `redirectUris` del realm son **relativas** y funcionan con las IP
   dinámicas de ECS sin comodines. En desarrollo, `proxy.conf.json` hace lo mismo con `ng serve`.
4. **CSRF:** cookie `XSRF-TOKEN` legible por Angular, que la reenvía en `X-XSRF-TOKEN`. Como la
   sesión va en cookie, CSRF es obligatorio. El logout es un POST con `_csrf` y cierra también la
   sesión de Keycloak (`id_token_hint`).
5. Hacia los servicios internos **no cambia nada**: el Gateway sigue enviando su token de servicio
   y se elimina el header `Cookie` (`RemoveRequestHeader=Cookie`).
6. Keycloak usa **hostname dinámico** (`KC_HOSTNAME_STRICT=false`, también en Compose): el
   navegador entra por el host del frontend y los servicios por `keycloak:8080`. Los tokens de
   servicio conservan `iss=http://keycloak:8080/...`.
7. **Sin `user-info-uri`:** el access token del usuario lleva como `iss` el host del navegador, y
   Keycloak lo rechaza en `/userinfo` por la red interna. Nombre y correo ya vienen en el ID token.

### Realm
- El import con `clientScopes` explícitos no crea los scopes estándar. Se agregan `profile`,
  `email` y `basic` con sus mappers.
- `lastName` deja de ser obligatorio en el perfil de usuario: el usuario de prueba solo tiene
  nombre, y no se inventan datos.
- Usuario de prueba: `${TEST_USER_EMAIL}` / `${TEST_USER_FIRST_NAME}`, con contraseña temporal
  (UPDATE_PASSWORD).
- Usuario técnico `e2e` ("Selenium E2E") con contraseña no temporal `${E2E_USER_PASSWORD}`, para
  Selenium y los scripts.
- Login en español (`defaultLocale: es`).

## Alternativas descartadas
- **OIDC en el SPA (keycloak-js):** no funciona sobre HTTP sin contexto seguro y deja tokens en
  el navegador (expuestos a XSS).
- **TLS con ALB + ACM o CloudFront:** requiere dominio o más piezas y costo fijo. Se reconsidera
  si el proyecto pasa a producción real.
- **Implicit flow / PKCE `plain`:** desaconsejados por OAuth 2.1.

## Consecuencias
- Sesión en memoria del Gateway: con una sola tarea basta. Si se escala, hay que usar
  Spring Session con Redis.
- `run-cloud-tests.sh`, `verificar-stack.ps1` y Selenium inician sesión con el usuario `e2e`
  antes de usar la API.
- `userName` del itinerario sigue llegando en el cuerpo, ahora desde la identidad de Keycloak. Que
  el backend lo tome del token (y no del cliente) queda como mejora pendiente.
