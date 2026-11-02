# Traefik + Docker Socket Proxy

Reverse proxy (ingress) para Docker con certificados TLS automáticos de Let's Encrypt
usando el **DNS-01 challenge** a través de Cloudflare, y con el socket de Docker
expuesto de forma **restringida** mediante `docker-socket-proxy`.

---

## Índice

- [Arquitectura](#arquitectura)
- [Componentes](#componentes)
- [Cómo funciona](#cómo-funciona)
  - [1. El socket proxy: por qué existe](#1-el-socket-proxy-por-qué-existe)
  - [2. El descubrimiento de servicios](#2-el-descubrimiento-de-servicios)
  - [3. Las redes de Docker](#3-las-redes-de-docker)
  - [4. Los entrypoints y la redirección HTTP→HTTPS](#4-los-entrypoints-y-la-redirección-httphttps)
  - [5. TLS y Let's Encrypt con DNS-01](#5-tls-y-lets-encrypt-con-dns-01)
  - [6. El dashboard](#6-el-dashboard)
- [Autenticación del dashboard](#autenticación-del-dashboard)
  - [Opción A: BasicAuth](#opción-a-basicauth)
  - [Opción B: Cloudflare Access (Zero Trust)](#opción-b-cloudflare-access-zero-trust)
  - [Opción C: Cloudflare Access + defensa en profundidad](#opción-c-cloudflare-access--defensa-en-profundidad)
- [Puesta en marcha](#puesta-en-marcha)
- [Configuración](#configuración)
- [Cómo publicar un servicio](#cómo-publicar-un-servicio)
- [Certificados wildcard](#certificados-wildcard)
- [Operación y mantenimiento](#operación-y-mantenimiento)
- [Problemas frecuentes](#problemas-frecuentes)
- [Notas de la revisión del compose](#notas-de-la-revisión-del-compose)

---

## Arquitectura

```
                         Internet
                            │
                     :80 / :443 (host)
                            │
                   ┌────────▼─────────┐
                   │      traefik     │  ──► red "proxy" (externo)
                   │  entrypoints     │
                   │  web / websecure │
                   └───┬──────────┬───┘
                       │          │
         red docker-api│          │◄── socket-proxy :2375
          (internal)   │          │      (HAProxy sobre docker.sock)
                       │          │
                       └────┬─────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          servicio-a   servicio-b   servicio-c
          (labels     (labels      (labels
          traefik.*)   traefik.*)   traefik.*)
```

Dos contenedores, dos redes, una sola puerta de entrada.

---

## Componentes

| Contenedor | Imagen | Rol |
|---|---|---|
| `socket-proxy` | `ghcr.io/tecnativa/docker-socket-proxy:v0.5.0` | Proxy HAProxy que expone el socket de Docker por HTTP (puerto 2375) filtrando la API por secciones. |
| `traefik` | `traefik:v3.7` | Reverse proxy / load balancer. Lee la config dinámica de Docker y emite los certificados. |

Ambos servicios loguean con el driver `json-file` y rotación automática
(`max-size: 10m`, `max-file: 3`) — ver [Rotación de logs](#rotación-de-logs).

---

## Cómo funciona

### 1. El socket proxy: por qué existe

Traefik necesita leer el estado de Docker (qué contenedores hay, qué puertos exponen,
qué redes tienen, qué labels tienen) para construir sus rutas. La forma habitual de
conseguirlo es montando `/var/run/docker.sock` dentro del contenedor de Traefik.

**Eso equivale a darle root al host.** El socket de Docker permite, entre otras cosas,
crear contenedores con un bind-mount de `/` — es decir, escapar del contenedor y
tomar el control de la máquina. La propia documentación de Docker lo advierte:

> only **trusted** users should be allowed to control your Docker daemon

`socket-proxy` (basado en HAProxy) se interpone:

```
Traefik ──HTTP──▶ socket-proxy:2375 ──unix socket──▶ /var/run/docker.sock
```

El proxy aplica un `allow`/`deny` por prefijo de ruta de la API Docker según variables
de entorno. Con la configuración de este repo:

```yaml
environment:
  CONTAINERS: 1   # GET /containers/*  → listar e inspeccionar contenedores
  NETWORKS: 1     # GET /networks/*    → info de redes
  INFO: 1         # GET /info          → info del daemon
  POST: 0         # sin métodos de escritura
```

El resultado: Traefik puede **leer**, pero **no puede escribir**. No puede crear
contenedores, ejecutar comandos dentro de ellos, ni montar volúmenes. Si alguien
logra escapar de Traefik, lo que obtiene es un API de solo lectura en lugar de root.

> `POST: 0` es lo que realmente blinda el sistema. Sin él, `POST /containers/create`
> + `POST /containers/<id>/start` permitirían montar el host.

Además, la red `docker-api` es `internal: true`, así que **no tiene salida a
Internet** y el puerto 2375 nunca se publica al host. Solo Traefik puede hablar con él.

### 2. El descubrimiento de servicios

Traefik no lee un fichero de configuración: consulta la API Docker y construye
la configuración en caliente.

```
Traefik                     Docker API (vía proxy)
  │                                │
  │  GET /containers/json?all=1 ──▶│   lista todos los contenedores (running + stopped)
  │◀───────────────────────────────┤
  │                                │
  │  GET /containers/{id}/json ───▶│   inspecciona cada uno (labels, redes, puertos)
  │◀───────────────────────────────┤
  │  GET /events (stream) ────────▶│   escucha cambios en tiempo real
  │◀───────────────────────────────┤
  │                                │
  ▼  construye routers + services
```

Con `--providers.docker.exposedBydefault=false`, **solo** los contenedores que
tengan el label `traefik.enable=true` entran en la configuración. Cualquier otro
contenedor del host es invisible para Traefik.

El provider escucha los eventos del Docker (`start`, `stop`, `die`, `create`…), así
que cuando levantas un contenedor nuevo con sus labels, Traefik lo recoge en
segundos sin reiniciar nada. Cuando un contenedor se para, la ruta desaparece.

### 3. Las redes de Docker

Hay dos redes con propósitos distintos:

**`proxy` (external)** — la red "pública" de enrutamiento. Traefik está conectado y
los servicios que quieras publicar deben unirse también. Es `external: true` porque
es una red compartida entre stacks: la creas una vez y cada stack se engancha.

```bash
docker network create proxy
```

**`docker-api` (internal)** — red aislada, sin acceso a Internet, que conecta
únicamente Traefik con `socket-proxy`. Es la que implementa el aislamiento del
socket proxy.

> `docker-api` es `internal: true`, lo que significa que los contenedores en ella
> **no pueden salir a Internet**. Por eso `socket-proxy` no necesita estar en `proxy`.

### 4. Los entrypoints y la redirección HTTP→HTTPS

Traefik expone dos *entrypoints* (puntos de entrada):

```yaml
--entrypoints.web.address=:80
--entrypoints.websecure.address=:443
```

- **`web`** (`:80`) — HTTP plano. Redirige a HTTPS.
- **`websecure`** (`:443`) — HTTPS. Aquí van los routers reales y el dashboard.

```yaml
--entrypoints.web.http.redirections.entrypoint.to=websecure
--entrypoints.web.http.redirections.entrypoint.scheme=https
```

El mecanismo es una redirección a nivel de *entrypoint*: cualquier petición que
llegue por `web` recibe un `301` a `https://mismo-host/...` usando el `websecure`.
No necesitas definir un router para esto — es una política global del entrypoint.

Consecuencia: el puerto 80 solo sirve para redirigir. Todo el tráfico real va por 443.

### 5. TLS y Let's Encrypt con DNS-01

El "certificate resolver" llamado `cloudflare` está configurado así:

```yaml
--certificatesresolvers.cf.acme.email=${TRAEFIK_ACME_EMAIL}
--certificatesresolvers.cf.acme.storage=/letsencrypt/acme.json
--certificatesresolvers.cf.acme.dnschallenge.provider=cloudflare
--certificatesresolvers.cf.acme.dnschallenge.resolvers=1.1.1.1:53,1.0.0.1:53
```

**¿Por qué DNS-01 y no HTTP-01?** El challenge HTTP-01 requiere que Let's Encrypt
pueda alcanzar `http://tu-dominio/.well-known/acme-challenge/...` desde Internet.
Eso significa que el puerto 80 debe estar abierto y el DNS apuntar al host. En
muchos escenarios (Cloudflare Tunnel, NAT, IP dinámica) eso no es viable. El
challenge DNS-01 en cambio solo necesita que Traefik pueda **escribir un registro
TXT** en la zona DNS — no requiere puertos abiertos ni que el tráfico llegue al host.

**El flujo completo:**

```
Traefik                 Cloudflare API              Let's Encrypt
  │                          │                            │
  │ ¿necesita cert para       │                            │
  │ app.example.com?         │                            │
  │──────────────────────────▶│                            │
  │  crea TXT _acme-challenge│                            │
  │  app.example.com         │                            │
  │◀──────────────────────────│                            │
  │                          │                            │
  │  espera propagación ────────────────────────────────▶ │
  │                          │                            │
  │                          │        valida el TXT ──────▶│
  │                          │◀─────── TXT encontrado ────│
  │                          │                            │
  │◀─────── "desafío resuelto" ──────────────────────────│
  │                                                          │
  │ firma la CSR y solicita el certificado                  │
  │────────────────────────────────────────────────────────▶│
  │◀─────────────────── certificado X.509 ────────────────│
  │                                                          │
  │ lo guarda en /letsencrypt/acme.json                    │
```

**¿De dónde sale el token de Cloudflare?** Traefik usa internamente la librería
[Lego](https://go-acme.github.io/lego/), que espera la variable de entorno
`CF_DNS_API_TOKEN`. Como `env_file: .env` la inyecta en el contenedor, el token
llega a Traefik por ahí — no hace falta ningún flag extra. (Si el token no estuviera
en `.env`, el challenge fallaría con un error de autenticación de Cloudflare.)

**Los resolvers DNS** (`1.1.1.1:53`, `1.0.0.1:53`) son los servidores públicos de
Cloudflare que usa Traefik para comprobar que el TXT se ha propagado antes de
notificar a Let's Encrypt. Se pueden omitir para usar los del sistema.

**El almacenamiento** en `/letsencrypt/acme.json` es crítico: ahí viven las claves
privadas de los certificados. Es un fichero que **no debe borrarse ni compartirse**
(volumen bind-mount `./letsencrypt/acme.json`). Traefik renueva los certificados
automáticamente cuando faltan ~30 días para su expiración (validez de 90 días).

### 6. El dashboard

```yaml
--api.dashboard=true
- traefik.enable=true
- traefik.http.routers.traefik-dashboard.rule=Host(`traefik.${DOMAIN_NAME}`)
- traefik.http.routers.traefik-dashboard.entrypoints=websecure
- traefik.http.routers.traefik-dashboard.service=api@internal
```

Esto define un router estático (no viene de un contenedor, viene de los labels del
propio Traefik) que sirve el dashboard en `https://traefik.tudominio.com`, usando el
servicio interno `api@internal`.

Con `tls=true` y `certresolver=cloudflare`, ese host también obtiene su certificado
automáticamente vía DNS-01.

> ⚠️ **El dashboard está expuesto a Internet sin autenticación.** Cualquiera que
> conozca la URL ve la topología completa de tu infraestructura. Ver la sección
> [Autenticación del dashboard](#autenticación-del-dashboard).

---

## Autenticación del dashboard

El dashboard y la API de Traefik exponen información sensible: nombres de
contenedores, dominios internos, redes, rutas, certificados y configuración
completa. La propia documentación de Traefik es explícita:

> Enabling the API and the dashboard in production is not recommended, because it
> will expose all configuration elements [...] In production, it should be at
> least secured by authentication and authorizations.

Hay tres estrategias, de menor a mayor sofisticación:

| Estrategia | Contraseña | Suitable para | Coste |
|---|---|---|---|
| [BasicAuth](#opción-a-basicauth) | Sí | Un administrador, acceso ocasional | Mínimo |
| [Cloudflare Access](#opción-b-cloudflare-access-zero-trust) | No | Acceso por correo, varios usuarios, MFA | Medio |
| [Cloudflare Access + defence en profundidad](#opción-c-cloudflare-access--defensa-en-profundidad) | No | Producción seria | Medio |

**Recomendación:** si ya usas Cloudflare para los DNS (que es el caso de este
stack), la opción B es claramente superior a BasicAuth. Un usuario escribe su
correo y recibe un código de un solo uso. No hay contraseñas que gestionar ni
que filtrar.

### Antes de nada: qué hay que proteger

El dashboard no es solo la interfaz web. También expone la **API REST** en
`/api/*`, que devuelve lo mismo en JSON (y además permite modificar la
configuración dinámica en algunos endpoints). Por eso la regla del router debe
cubrir **ambos** prefijos de ruta:

```yaml
- traefik.http.routers.traefik-dashboard.rule=Host(`traefik.${DOMAIN_NAME}`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))
```

Con la regla por host completo (`Host(...)` a secas, que es la que tiene el
compose ahora mismo) también queda protegido —todo lo que llegue a ese host
pasa por el middleware—, pero la forma explícita es más robusta: si mañana
añades otro router al mismo host, la regla explícita sigue funcionando.

> La barra final en `/dashboard/` es obligatoria. Si alguien pide `/dashboard`
> sin barra, recibe un 404. Puedes redirigir con el middleware `RedirectRegex`.

### Opción A: BasicAuth

La opción más simple: usuario y contraseña gestionados por Traefik. El navegador
muestra el diálogo nativo de autenticación.

#### Generar el hash

Traefik acepta contraseñas hasheadas con **MD5, SHA1 o BCrypt**. Lo más
práctico es `htpasswd` de Apache:

```bash
# htpasswd está en el paquete apache2-utils (Debian/Ubuntu)
htpasswd -nB admin
# admin:$2y$05$...          (bcrypt, recomendado)

htpasswd -nB admin > auth/users.yml   # guarda usuario:hash en un fichero
```

Para Docker sin instalar nada:

```bash
docker run --rm -it --entrypoint htpasswd httpd:2 -Bbn admin 'mi-contraseña' > auth/users.yml
```

El formato del fichero `users.yml` es una lista de `usuario:hash`:

```yaml
# auth/users.yml
admin:$2y$05$aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789abcdefGHIJKLMNOPQRSTUV
```

#### Configurar Traefik

Los middlewares **no se pueden definir con labels de Docker** de forma práctica
(el valor sería una lista inline con caracteres especiales que hay que escapar). La forma limpia es
un fichero de configuración dinámica en YAML:

```yaml
# traefik.dynamic.yml
http:
  middlewares:
    dashboard-auth:
      basicAuth:
        usersFile: /etc/traefik/users.yml
        realm: "Traefik Dashboard"
        removeHeader: true
```

Y montarlo en el contenedor:

```yaml
# compose.yml
services:
  traefik:
    volumes:
      - ./letsencrypt/acme.json:/letsencrypt/acme.json:rw
      - ./auth/users.yml:/etc/traefik/users.yml:ro
      - ./traefik.dynamic.yml:/etc/traefik/dynamic.yml:ro
    command:
      # ...
      - --providers.file.filename=/etc/traefik/dynamic.yml
      # referenciar el middleware en el router
      - traefik.http.routers.traefik-dashboard.middlewares=dashboard-auth
```

Alternativa sin fichero de usuarios: `basicAuth.users` inline en el YAML
dinámico, con el hash como valor literal.

#### Ventajas y desventajas

✅ No requiere nada externo — un fichero y ya.
✅ Funciona aunque Cloudflare esté caído.
❌ Una contraseña compartida entre todos los administradores.
❌ Sin MFA, sin registro de auditoría, sin revocación inmediata.
❌ El hash vive en un fichero en el disco — hay que protegerlo (`chmod 600`).
❌ BasicAuth envía las credenciales en base64 por cada petición (solo TLS lo
salva de ser sniffed).

### Opción B: Cloudflare Access (Zero Trust)

Cloudflare Access pone una **capa de autenticación por delante de la
aplicación**. El usuario nunca ve Traefik directamente: va a Cloudflare, se
autentica allí, y solo si Access lo permite, la petición llega a tu origen.

```
Navegador ──▶ Cloudflare Edge ──▶ [Access: ¿sesión válida?] ──▶ Traefik ──▶ Dashboard
                    │
                    └─── Si no hay sesión → pantalla de login de Cloudflare
                         (correo + código OTP, o tu IdP corporativo)
```

Ventajas frente a BasicAuth:

- **El usuario se autentica con su correo.** Sin contraseñas que gestionar.
- **MFA y SSO** gratis (con el proveedor de identidad de Cloudflare o el tuyo).
- **Auditoría** de accesos en el panel de Zero Trust.
- **Revocación inmediata** de una sesión.
- **Las credenciales nunca tocan tu servidor.** Cloudflare las gestiona.
- Cloudflare Access es **gratuito** hasta 50 usuarios.

#### Paso 1 — Crear la aplicación en Access

En el panel de Cloudflare → **Zero Trust → Access controls → Applications**:

1. **Create new application** → **Self-hosted** → **Add public hostname**
2. Dominio: `traefik.tudominio.com` (debe pertenecer a una zona activa de tu cuenta)
3. Path: deja vacío (protege todo el host)
4. **Add a policy**:
   - Action: **Allow**
   - Include → **Emails**: `tu@email.com` (o el dominio entero, o un grupo)
5. **Identity providers** → selecciona el que quieras (ver siguiente paso)
6. **Session duration**: 24 h es un valor razonable
7. Guardar

> Todas las aplicaciones de Access son **deny by default**. Un usuario debe
> coincidir con una política `Allow` explícitamente; si no, se le bloquea.

#### Paso 2 — Habilitar el login por correo (One-Time PIN)

Esta es la parte que responde a "permitir el ingreso mediante correo".

Cloudflare Access puede enviar un **código de un solo uso (OTP)** al correo en
lugar de obligarte a integrar un IdP corporativo. Va en:

**Zero Trust → Integrations → Identity providers → Add new identity provider →
One-time PIN**

```bash
# Equivalente vía API
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/access/identity_providers" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{"name": "One-time PIN login", "type": "onetimepin", "config": {}}'
```

El flujo para el usuario:

```
1. Entra en https://traefik.tudominio.com
2. Access muestra "Introduce tu email" → "Enviar código"
3. Recibe un PIN por correo (expira en 10 minutos)
4. Lo introduce → accede al dashboard
```

Detalles importantes del OTP:

- Cloudflare **solo envía el email si el usuario está permitido por una política**.
  Si escribes un email que no está en la política `Allow`, no llega nada — y la
  pantalla dice igualmente "te hemos enviado un código". Esto evita enumerar
  usuarios válidos.
- Cada PIN es de un solo uso. Pedir uno nuevo invalida el anterior.
- El PIN expira a los 10 minutos.
- **Cloudflare no te manda el email si tu gateway de correo lo bloquea o filtra.**
  Si es tu caso, permite en la lista blanca el remitente
  `noreply@notify.cloudflare.com` y las IPs
  `104.30.16.2` – `104.30.16.7`. Un filtro que "consume" el enlace
  automáticamente hará que el PIN aparezca como ya usado.

Si además de correo quieres Google Workspace, GitHub, Azure AD, Okta o SAML
corporativo, añade esos proveedores como IdP adicionales en el mismo sitio: el
usuario podrá elegir método en la pantalla de login.

#### Paso 3 — Proteger el origen (lo que casi todo el mundo se salta)

Access autentica **en el borde de Cloudflare**. Eso significa que si alguien
consigue alcanzar tu servidor por una vía que no pasa por Cloudflare (tu IP
pública directo, otro túnel, un `curl` a la IP), **se salta la autenticación
entera**.

Por eso, si tu origen es público, hay que cerrar el acceso directo. Opciones:

**a) Cloudflare Tunnel (lo más limpio)** — eliminates la IP pública por completo:

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    restart: unless-stopped
    networks:
      - proxy
```

Con el túnel, Cloudflare es el **único** camino posible hasta Traefik. No hay
IP pública que atacar. Y en la config del túnel puedes activar **"Protect with
Access"**, que hace que `cloudflared` valide el token JWT de Access por ti.

**b) Firewall** — si no usas túnel, cierra los puertos 80/443 a todo el mundo
salvo las IPs de Cloudflare. Lo más simple es dejar que Cloudflare te lo
gestione: en **Settings → Security → Security Settings**, activa
*"Under Attack Mode"* y luego "*I am under attack*"*... o mejor, usa la opción
de **Authenticated Origin Pulls** con un mTLS entre Cloudflare y tu origen.

**c) `ipAllowList` en Traefik** — defensa en profundidad, pero frágil: las IPs de
Cloudflare cambian. Útil solo como capa extra, no como única protección.

```yaml
- traefik.http.routers.traefik-dashboard.middlewares=dashboard-auth,ipallowlist
```

> Con Cloudflare Access **no** añadas un BasicAuth delante: el usuario ya se
> autenticó en Cloudflare y el token va en la cookie `CF_Authorization`. Si
> además pides BasicAuth, el usuario tendrá que hacer login dos veces. Y si
> añades un BasicAuth, **recuerda que Cloudflare reenvía las cabeceras
> `X-Forwarded-*`**, así que no confíes en ellas para decisiones de seguridad.

### Opción C: Cloudflare Access + defensa en profundidad

La opción B resuelve la autenticación. La opción C añade una capa más para
proteger el origen, útil si no puedes cerrar la IP pública inmediatamente.

#### Validar el JWT de Access en Traefik

Cloudflare añade a cada petición que reenvía una cabecera con el token JWT de
la sesión:

```
Cf-Access-Jwt-Assertion: <JWT firmado por Cloudflare>
```

El token contiene la identidad del usuario, la política aplicada y la
expiración. Puedes validarlo en el origen.

**Opción 1 — `forwardAuth` con `cloudflared`** (lo hace el túnel por ti):

Si el túnel tiene **"Protect with Access"** activado, `cloudflared` ya rechaza
las peticiones sin token válido. No hace falta nada en Traefik.

**Opción 2 — middleware `forwardAuth` a un validador propio:**

```yaml
# traefik.dynamic.yml
http:
  middlewares:
    cf-access:
      forwardAuth:
        address: "http://cloudflared:8080/verify"   # tu servicio validador
        authResponseHeaders:
          - "X-Auth-User-Email"
        authRequestHeaders:
          - "Cf-Access-Jwt-Assertion"
          - "Cookie"
```

**Opción 3 — plugin de Traefik para validar el JWT:**

```yaml
# traefik.dynamic.yml
http:
  middlewares:
    cf-access:
      plugin:
        jwt:
          jwksUrl: "https://<tu-equipo>.cloudflareaccess.com/cdn-cgi/access/certs"
          issuer: "https://<tu-equipo>.cloudflareaccess.com"
          claims:
            - Equals(`email`, `tu@email.com`)
```

> ⚠️ **El middleware `jwt` de Traefik es una función de Traefik Hub**, no
> disponible en Traefik OSS v3.7. En OSS las opciones 2 y 3 requieren un
> servicio validador propio (o Traefik Hub). Lo mismo aplica al middleware
> `oidc`: es exclusivo de Hub. `basicAuth`, `forwardAuth`, `ipAllowList` y
> `headers` **sí** están disponibles en OSS.

Sobre el `jwksUrl` de Cloudflare: Access **rota la clave de firma cada 6
semanas** y mantiene la anterior válida 7 días. Por eso Cloudflare recomienda
consultar el endpoint en lugar de fijar la clave:

- Usa el endpoint `https://<equipo>.cloudflareaccess.com/cdn-cgi/access/certs`
- Empareja el `kid` del JWT con el certificado correspondiente en `public_certs`
- No uses `public_cert` a secas: tu origen podría estar leyendo una caché
  obsoleta

#### Cloudflare Access en la práctica vs. BasicAuth

| | BasicAuth | Cloudflare Access |
|---|---|---|
| Contraseña que gestionar | Sí, una por usuario | No |
| MFA | No | Sí (con el IdP de Cloudflare o el tuyo) |
| Auditoría de accesos | No | Sí (Access logs) |
| Revocación de un usuario | Cambiar el fichero y recargar | Un clic en el panel |
| Login por correo (OTP) | No | Sí |
| Funciona sin Cloudflare | Sí | No (pero el BasicAuth de respaldo sí) |
| SSO corporativo | No | Sí (SAML/OIDC) |
| Coste | 0 | 0 (hasta 50 usuarios) |

#### Recomendación final

1. **Instala BasicAuth** como base (aunque sea con un usuario de emergencia).
   Cuesta 5 minutos y te salva si algo falla con Cloudflare.
2. **Activa Cloudflare Access con OTP** como método principal de acceso.
3. **Cierra el acceso directo al origen** (túnel o firewall). Si no lo haces,
   Access solo protege a quien pase por Cloudflare.
4. **Valida el token en el origen** si puedes permitírtelo (C o forwardAuth).
5. **Restringe la política por IP** si tienes una IP fija o una VPN — un
   segundo filtro que no depende de la sesión del navegador.

### Verificar que funciona

```bash
# 1. Sin autenticar, debe devolver 302 (a Cloudflare) o 401 (BasicAuth)
curl -sI https://traefik.tudominio.com/dashboard/ | head -5

# Con BasicAuth, debe devolver 200
curl -sI -u admin:mi-contraseña https://traefik.tudominio.com/dashboard/ | head -5

# 2. Ver el JWT que Cloudflare inyecta (con sesión iniciada en el navegador)
#    DevTools → Network → petición a /dashboard/ → Request Headers
#    Cf-Access-Jwt-Assertion: eyJhbGciOiJSUzI1NiIsImtpZCI6...
```

Para decodificar el payload del JWT (no es la firma, solo el contenido legible):

```bash
echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

```json
{
  "email": "tu@email.com",
  "sub": "uuid-del-usuario",
  "aud": ["4714c1358e65fe4b408ad6d432a5f878f08194bdb4752441fd56faefa9b2b6f2"],
  "iss": "https://tu-equipo.cloudflareaccess.com",
  "exp": 1735689600
}
```

El campo `aud` es el **Application Audience Tag** — identifica la aplicación
concreta. Es el valor que usaría un validador para comprobar que el token es
para *esta* app y no para otra de tu cuenta. Lo encuentras en
**Access → Applications → (tu app) → Configure → Additional settings →
Application Audience (AUD) Tag**.

### Referencias

- [Traefik — API & Dashboard (Security)](https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/)
- [Traefik — BasicAuth middleware](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/)
- [Traefik — ForwardAuth middleware](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/forwardauth/)
- [Cloudflare — Publish a self-hosted app with Access](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)
- [Cloudflare — One-time PIN login](https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/one-time-pin/)
- [Cloudflare — Validate Access JWTs](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/authorization-cookie/validating-json/)
- [Cloudflare — Access policies](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/)

---

## Puesta en marcha

### Requisitos

- Docker Engine + Docker Compose v2
- Un dominio con los registros DNS apuntando a la IP pública del host
- Una zona gestionada en Cloudflare
- Un API Token de Cloudflare con permisos:
  - `Zone / Zone / Read`
  - `Zone / DNS / Edit`
  - Scoped a la zona concreta

### Pasos

**1. Crear las variables de entorno**

```bash
cp .env.example .env
```

```ini
# .env
TRAEFIK_ACME_EMAIL=tu@email.com
CF_DNS_API_TOKEN=cfat_XXXXXXXXXXXXXXXX
DOMAIN_NAME=tudominio.com
```

**2. Crear la red externa `proxy`**

```bash
docker network create proxy
```

(Si ya existe, `docker network create proxy` devuelve un error benigno; puedes
comprobarlo con `docker network ls | grep proxy`.)

**3. Preparar el almacenamiento de certificados**

```bash
mkdir -p letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

Los permisos `600` son importantes: el fichero contiene las claves privadas de tus
certificados.

**4. Levantar el stack**

```bash
docker compose up -d
```

**5. Verificar**

```bash
docker compose ps
docker compose logs -f traefik
```

Deberías ver en los logs:

```
level=info msg="Starting provider *docker.Provider"
level=info msg="Start ACME resolver" resolverName=cloudflare
level=info msg="Trying to challenge certificate" domains=[traefik.tudominio.com]
level=info msg="The certificate is valid" domain=traefik.tudominio.com
```

---

## Configuración

Todas las variables están en `.env`:

| Variable | Uso | Ejemplo |
|---|---|---|
| `TRAEFIK_ACME_EMAIL` | Email de registro en ACME (Let's Encrypt). Se avisa de caducidades y problemas de emisión. | `admin@tudominio.com` |
| `CF_DNS_API_TOKEN` | Token de API de Cloudflare. Permisos `Zone:Read` + `DNS:Edit`. | `cfat_...` |
| `DOMAIN_NAME` | Se usa en la regla del router del dashboard: `Host(\`traefik.${DOMAIN_NAME}\`)`. | `tudominio.com` |

### Endpoints del dashboard

| URL | Servicio |
|---|---|
| `https://tudominio.com` | Redirige a HTTPS → `https://tudominio.com/` (routers de tus apps) |
| `https://traefik.tudominio.com` | Dashboard de Traefik |

---

## Cómo publicar un servicio

Añade labels al servicio en tu propio `compose.yml` y únelo a la red `proxy`:

```yaml
services:
  mi-app:
    image: mi-app:latest
    networks:
      - proxy
    labels:
      - traefik.enable=true
      - traefik.http.routers.mi-app.rule=Host(`app.tudominio.com`)
      - traefik.http.routers.mi-app.entrypoints=websecure
      - traefik.http.routers.mi-app.tls=true
      - traefik.http.routers.mi-app.tls.certresolver=cloudflare
      - traefik.http.services.mi-app.loadbalancer.server.port=8080
```

### Labels más habituales

**Regla de enrutamiento:**

```yaml
- traefik.http.routers.mi-app.rule=Host(`app.tudominio.com`)
```

Admite sintaxis completa de Traefik v3:

```yaml
- traefik.http.routers.mi-app.rule=Host(`app.tudominio.com`) && PathPrefix(`/api`)
- traefik.http.routers.mi-app.rule=HostRegexp(`{sub:[a-z]+}.tudominio.com`)
```

**TLS con Let's Encrypt:**

```yaml
- traefik.http.routers.mi-app.tls=true
- traefik.http.routers.mi-app.tls.certresolver=cloudflare
```

Sin la segunda línea, Traefik emite un **certificado autogenerificado** (self-signed)
y el navegador mostrará un aviso.

**Puerto de destino (si el contenedor expone varios):**

```yaml
- traefik.http.services.mi-app.loadbalancer.server.port=8080
```

**Red específica** (si la app vive en otra red):

```yaml
- traefik.docker.network=mi-red
```

**MidDLEwares** (headers, autenticación, etc.):

```yaml
- traefik.http.routers.mi-app.middlewares=auth@file,headers
```

**Health check** — Traefik no enruta tráfico a contenedores que no estén sano
(solo si definiste `healthcheck` en el servicio):

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 10s
  timeout: 3s
  retries: 3
```

### Ejemplo con autenticación básica

```yaml
# docker compose del servicio
services:
  mi-app:
    image: mi-app:latest
    networks: [proxy]
    labels:
      - traefik.enable=true
      - traefik.http.routers.mi-app.rule=Host(`app.tudominio.com`)
      - traefik.http.routers.mi-app.entrypoints=websecure
      - traefik.http.routers.mi-app.tls=true
      - traefik.http.routers.mi-app.tls.certresolver=cloudflare
      - traefik.http.routers.mi-app.middlewares=basic-auth
      - traefik.http.routers.mi-app.service=mi-app
      - traefik.http.services.mi-app.loadbalancer.server.port=8080

  # Los middlewares complejos (basic-auth, rate-limit) se definen en un
  # fichero aparte: traefik.dynamic.yml montado en el contenedor.
```

Para middlewares complejos (basic-auth, rate-limit) lo más limpio es usar un
`traefik.dynamic.yml` montado en el contenedor, o el
[plugin de Traefik](https://doc.traefik.io/traefik/plugins/) correspondiente.

---

## Certificados wildcard

El DNS-01 es el único challenge que soporta comodines. Para un wildcard
`*.tudominio.com` necesitas una regla que lo mencione:

```yaml
- traefik.http.routers.mi-app.tls.domains[0].main=*.tudominio.com
- traefik.http.routers.mi-app.tls.domains[0].sans=tudominio.com
```

> Ten en cuenta: un wildcard y su dominio raíz comparten registro TXT, lo que puede
> causar fallos por caché de DNS en Cloudflare. Si ocurre, separa la emisión del
> wildcard y la del dominio raíz en timescales distintos.

---

## Operación y mantenimiento

### Comandos habituales

```bash
# Estado
docker compose ps
docker compose logs -f traefik

# Reiniciar solo Traefik (no afecta a los servicios)
docker compose restart traefik

# Reconstruir tras cambios en el compose
docker compose up -d

# Ver el certificado emitido
docker compose exec traefik cat /letsencrypt/acme.json | jq .

# Ver la configuración dinámica que ha construido Traefik
docker compose exec traefik traefik api get @docker 2>/dev/null || \
  curl -s https://traefik.tudominio.com/api/http/routers
```

### Certificados

- Validez: 90 días.
- Renovación: automática, cuando faltan ~30 días para expirar.
- Comprobar estado: `docker compose logs traefik | grep -i "certificate"`.

### Backups

Lo único que hay que respaldar es:

```bash
cp letsencrypt/acme.json /backup/acme.json
```

Si lo pierdes, Traefik simplemente emite certificados nuevos en el siguiente
arranque (no hay downtime, solo se repite el proceso de emisión).

### Logs

```bash
docker compose logs traefik --tail=100
docker compose logs socket-proxy --tail=100
```

En `socket-proxy`, los `403 Forbidden` son indicativos de que algo intentó una
operación de escritura — normalmente inofensivo (Docker internals), pero conviene
revisarlo si aparecen en ráfaga.

#### Rotación de logs

Ambos servicios usan el driver `json-file` con rotación automática configurada en
el `compose.yml`:

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

Esto significa que:

- Cada fichero de log alcanza como máximo **10 MB** y entonces se rota.
- Se conservan hasta **3 ficheros** por contenedor (el actual + 2 históricos).
- El consumo máximo por contenedor es de **~30 MB**.

Sin esta configuración, Docker deja crecer los logs indefinidamente
(`/var/lib/docker/containers/<id>/<id>-json.log`) hasta llenar el disco — un
problema habitual en servidores longevos, especialmente con Traefik, que registra
cada petición.

> La rotación **no aplica retroactivamente**: los contenedores que ya estaban
> corriendo mantienen su configuración anterior hasta que se recrean. `docker compose
> restart` no sirve aquí, porque reiniciar no recrea el contenedor y el driver se fija
> en su creación.

Para aplicar el cambio:

```bash
docker compose up -d
```

Como la definición de `logging` ha cambiado, Compose detecta que la configuración
difiere y **recrea** los contenedores. Si dudas, o si cambias los límites a mano,
fuerza el recreación:

```bash
docker compose up -d --force-recreate
```

Para consultar el tamaño actual de los logs (en el host):

```bash
sudo du -sh /var/lib/docker/containers/*/*-json.log 2>/dev/null | sort -h
```

---

## Problemas frecuentes

### El dashboard no carga / error de TLS

Comprueba el DNS:

```bash
dig +short traefik.tudominio.com
```

Debe devolver la IP pública del host. Si usas Cloudflare, recuerda que el proxy
está en modo "DNS only" (nube gris) para que Let's Encrypt pueda validar el
challenge DNS-01. **No** pongas el dominio en modo "Proxied" (nube naranja) si
quieres usar el challenge HTTP-01; con DNS-01 funciona en ambos modos.

### `unable to generate a certificate for the domains [...]: error: one or more domains had a problem`

Casi siempre es el token de Cloudflare. Verifica:

```bash
docker compose exec traefik env | grep CF_DNS
```

Debe mostrar `CF_DNS_API_TOKEN=...` con un valor no vacío. Si está vacío, el
fichero `.env` no se está cargando o le falta la variable.

Comprueba también los permisos del token: necesita `Zone:Read` + `DNS:Edit`.

### `403 Forbidden` en los logs del socket proxy

Significa que Traefik intentó una operación de escritura en la API Docker. Con la
configuración actual (`POST: 0`) eso está bloqueado a propósito. Si ves esto de
forma puntual, es normal. Si ves errores de lectura (`CONTAINERS`/`NETWORKS`
bloqueados), revisa las variables de entorno del socket-proxy.

### "No route found" / 404 en un subdominio

Casi siempre falta `traefik.enable=true` o la regla del router. Recuerda:

```bash
docker compose logs traefik | grep -i "router"
```

Y comprueba que el servicio está en la red `proxy` **y** conectado a ella:

```bash
docker inspect -f '{{json .NetworkSettings.Networks}}' mi-app
```

### Un contenedor nuevo no aparece

Comprueba que:
1. Tiene el label `traefik.enable=true`.
2. Está en la red `proxy`.
3. Su router declara `entrypoints=websecure`.
4. Su router tiene `tls=true` + `tls.certresolver=cloudflare`.

### `traefik.docker.network` no encuentra la red

El nombre de la red en el label debe coincidir **exactamente** con el nombre real.
Si usas Compose con `name:` en la red, el nombre real incluye el prefijo del
proyecto. Comprueba con `docker network ls`.

### Conflictos de puertos

Otro servicio puede estar ocupando los puertos 80/443 del host:

```bash
sudo lsof -i :80 -i :443
```

---

## Notas de la revisión del compose

Esta sección recoge el resultado de revisar `compose.yml`. Los puntos marcados con
⚠️ son cosas que **funcionan** pero conviene endurecer; los marcados con 🐛 son
bugs reales.

### 🐛 1. El dashboard no tiene autenticación

`--api.dashboard=true` combinado con un router público deja el panel de control
accesible a cualquiera que adivine `traefik.tudominio.com`. Expone nombres de
contenedores, redes, dominios internos y certificados.

**Mitigación** — ver la sección completa
[Autenticación del dashboard](#autenticación-del-dashboard), que cubre las tres
opciones: BasicAuth (Opción A), Cloudflare Access con login por correo (Opción B)
y Cloudflare Access con defensa en profundidad (Opción C).

Dado que este stack ya depende de Cloudflare para el DNS, la **Opción B** es la
más natural: el usuario accede con su correo y un código de un solo uso, sin
contraseñas que gestionar.

Resumen del cambio mínimo en el compose:

```yaml
services:
  traefik:
    volumes:
      - ./letsencrypt/acme.json:/letsencrypt/acme.json:rw
      - ./auth/users.yml:/etc/traefik/users.yml:ro
      - ./traefik.dynamic.yml:/etc/traefik/dynamic.yml:ro
    command:
      - --providers.file.filename=/etc/traefik/dynamic.yml
      - traefik.http.routers.traefik-dashboard.middlewares=dashboard-auth
```

> El fichero `auth/users.yml` debe existir **antes** de levantar el stack, o
> Docker lo creará como un directorio vacío y Traefik fallará al leerlo.

Alternativa aún más simple: **no publicar el dashboard** y acceder por SSH con
tunnel, o restringirlo por IP con `ipAllowList`.

### 🐛 2. `CF_DNS_API_TOKEN` sin usar explícitamente en el comando

Funciona por el mecanismo de `env_file`, pero es implícito y frágil. Si alguien
cambia a `environment:` en el futuro, o si Traefik deja de heredar el env file, el
challenge falla en silencio (con un error poco descriptivo). Alternativa más
explícita:

```yaml
command:
  - --certificatesresolvers.cf.acme.dnschallenge.provider=cloudflare
  # Alternativa usando providerToken (Traefik v3.2+):
  # --certificatesresolvers.cf.acme.dnschallenge.providerToken=${CF_DNS_API_TOKEN}
```

### ⚠️ 3. Permisos del socket proxy más amplios de lo necesario

`INFO: 1` no es imprescindible para el Docker provider de Traefik — este solo
necesita `CONTAINERS` (listar e inspeccionar), `NETWORKS` y `EVENTS` (este último
ya viene permitido por defecto). `INFO` expone metadatos del daemon del host.

Configuración mínima recomendada:

```yaml
environment:
  CONTAINERS: 1
  NETWORKS: 1
  EVENTS: 1
  POST: 0
  INFO: 0
```

Además, `NETWORKS` solo es necesario si usas `traefik.docker.network` para
seleccionar una red concreta; si todos tus servicios viven en `proxy`, puedes
dejarlo en `0` (Traefik lee las redes del contenedor al inspeccionarlo, no hace
falta listar redes de la API).

### ⚠️ 4. `docker.sock` montado en lectura (`ro`) — correcto, pero incompleto

El mount `:ro` está bien y evita que Traefik pueda escribir en el socket. Aun así,
la protección real la aporta el socket-proxy. Mantener ambas cosas.

### ⚠️ 5. `acme.json` sin permisos restrictivos explícitos

El compose no fija permisos ni usuario. El bind-mount crea el fichero con los
permisos del usuario que lanza Docker. Si el host es multiusuario, conviene:

```bash
chmod 600 letsencrypt/acme.json
```

Y en el compose, fijar el usuario del contenedor para que no corra como root:

```yaml
user: "65532:65532"   # nonroot
```

(comprobar que el usuario existe en la imagen `traefik:v3.7` y que tiene acceso de
escritura al volumen).

### ⚠️ 6. Ausencia de healthcheck

Si Traefik arranca antes de que `socket-proxy` esté listo, falla el primer refresh
y reintenta solo (con backoff), pero produce ruido en los logs. Un healthcheck
mejora el arranque:

```yaml
healthcheck:
  test: ["CMD", "traefik", "healthcheck", "--ping"]
  interval: 10s
  timeout: 5s
  retries: 3
```

Requiere `--ping=true` en los command.

### ⚠️ 7. Sin política de reinicio para fallos de arranque

`restart: unless-stopped` está bien. Si Traefik entra en crash-loop por una
configuración inválida, `unless-stopped` lo reintenta indefinidamente. Conviene
vigilar con `docker compose logs` tras cada cambio de configuración.

### ⚠️ 8. Permisos del usuario de la red `proxy`

Si la red `proxy` no existe cuando se levanta el stack, Compose falla con
`network proxy declared as external, but could not be found`. Créala antes
(ver [Puesta en marcha](#puesta-en-marcha)) o quítale `external: true`.

### ✅ Lo que está bien

- `docker.sock` montado en solo lectura.
- `POST: 0` — sin operaciones de escritura en la API Docker.
- `docker-api` como red `internal: true`, sin salida a Internet y sin puertos
  publicados.
- El socket-proxy no publica el puerto 2375 al host.
- `exposedbydefault=false` — solo se exponen los contenedores con `traefik.enable=true`.
- Redirección HTTP→HTTPS a nivel de entrypoint.
- `acme.json` persistido en disco (los certificados sobreviven a los reinicios).
- Cloudflare usado como DNS provider es compatible con DNS-01 y wildcard.

### ✅ Detalles menores que ya están correctos

- **El label `traefik.enable=true` en el propio Traefik** es necesario: es lo que
  hace que Traefik se exponga a sí mismo a través del Docker provider (y no como
  config estática).
- **`--providers.docker.network=proxy`** fija la red por defecto para enrutar a
  los contenedores. Se puede sobrescribir por contenedor con
  `traefik.docker.network`.
- **`hostname: traefik`** es inofensivo y aids en la legibilidad del dashboard.
- **Las versiones de imagen están pinneadas** (`v3.7`, `v0.5.0`) — buena práctica,
  evita upgrades no deseados. Recuerda actualizarlas de forma deliberada.

---

## Referencias

- [Traefik — Docker Provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/)
- [Traefik — ACME / Certificate Resolvers](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/)
- [Traefik — Routing con labels de Docker](https://doc.traefik.io/traefik/routing-configuration/other-providers/docker/)
- [docker-socket-proxy — Tecnativa](https://github.com/Tecnativa/docker-socket-proxy)
- [Lego — DNS providers (Cloudflare)](https://go-acme.github.io/lego/dns/)
- [Cloudflare — API Tokens](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
