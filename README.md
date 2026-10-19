# crossfit-deploy

Configuración de ejecución del stack CrossFit (PWA + API + PostgreSQL + Caddy) para la Raspberry Pi.

Contiene los ficheros Compose y la configuración de Caddy de cada entorno. No contiene código de aplicación ni secretos.

## Estructura

```text
environments/
  preproduction/
    docker-compose.yml
    caddy/Caddyfile
    .env.example
  production/
    docker-compose.yml
    caddy/Caddyfile
    .env.example
```

- `preproduction`: expone `8080` (HTTP) y `8443` (HTTPS). Nombre local `preprod.crossfit.home.arpa`. PostgreSQL accesible solo en `127.0.0.1:5433` del host (para herramientas como DBeaver mediante túnel SSH).
- `production`: expone `80` (HTTP) y `443` (HTTPS). Nombre local `crossfit.home.arpa`.

Cada entorno es un proyecto Compose aislado (`crossfit-pre`, `crossfit-prod`) con su propia red, volúmenes y base de datos.

## Imágenes

Las imágenes se publican en GitHub Container Registry y la Pi solo las descarga:

- `ghcr.io/quiquesancho/crossfit-api`
- `ghcr.io/quiquesancho/crossfit-web`

Etiquetas:

| Etiqueta | Uso |
| --- | --- |
| `preview-<sha>` | candidata desplegada manualmente en preproducción |
| `sha-<commit>` | imagen publicada desde `main`, sin despliegue automático |
| `vX.Y.Z` | versión estable |
| `latest` | última versión estable |

## Configuración en la Raspberry Pi

Directorios recomendados:

```text
/opt/crossfit/crossfit-deploy      # este repositorio (git clone)
/opt/crossfit/preproduction/.env   # secretos de preproducción
/opt/crossfit/production/.env      # secretos de producción
```

Crear los ficheros `.env` a partir de `.env.example` y completar los valores reales. Los ficheros `.env` nunca se versionan.

Variables:

| Variable | Descripción |
| --- | --- |
| `API_IMAGE` | imagen de la API con su etiqueta |
| `WEB_IMAGE` | imagen de la PWA con su etiqueta |
| `SITE_ADDRESS` | nombre local del entorno |
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | credenciales de la base de datos |
| `ATHLETE_DISPLAY_NAME` | nombre visible del perfil local inicial |
| `ATHLETE_TIMEZONE` | zona horaria deportiva, por defecto `Europe/Madrid` |

## Runner autohospedado

El despliegue se ejecuta con un runner de GitHub Actions instalado en la Pi. El runner abre conexión saliente con GitHub; no se expone SSH.

Se registra un runner por repositorio que despliega (`crossfit-api` y `crossfit-web`), en directorios separados.

```bash
# En la Raspberry Pi (usuario con acceso a Docker)
mkdir -p /opt/actions-runner/crossfit-api && cd /opt/actions-runner/crossfit-api

# Descargar la última versión linux-arm64 desde:
# https://github.com/actions/runner/releases
curl -o actions-runner.tar.gz -L <URL_DEL_TAR_ARM64>
tar xzf ./actions-runner.tar.gz

# El token se obtiene en:
#   https://github.com/quiquesancho/crossfit-api/settings/actions/runners/new
./config.sh --url https://github.com/quiquesancho/crossfit-api --token <TOKEN> \
  --labels raspberrypi,preproduction,production

sudo ./svc.sh install
sudo ./svc.sh start
```

Repetir en `/opt/actions-runner/crossfit-web` con la URL y el token de `crossfit-web`.

Tras instalar el runner, activar los despliegues creando la variable de repositorio `DEPLOY_ENABLED=true` en **Settings → Secrets and variables → Actions → Variables** de cada repositorio. Mientras no exista, los jobs de despliegue se omiten.

## Despliegue

Los workflows de cada repositorio de servicio ejecutan en la Pi:

```bash
git -C /opt/crossfit/crossfit-deploy pull --ff-only
cd /opt/crossfit/crossfit-deploy
API_IMAGE="ghcr.io/quiquesancho/crossfit-api:<tag>" \
docker compose --env-file /opt/crossfit/<entorno>/.env \
  -f environments/<entorno>/docker-compose.yml up -d --pull always
```

- **Preproducción**: manual mediante GitHub Actions, indicando una rama, tag o SHA; fusionar en `main` no modifica PRE.
- **Producción**: tras un release SemVer, con aprobación manual del entorno `production`, con etiqueta `vX.Y.Z`.

### Desplegar una referencia en PRE

Cada repositorio de servicio incluye el workflow **Deploy ref to preproduction**. En GitHub:

1. Abrir **Actions** en `crossfit-api` o `crossfit-web`.
2. Seleccionar **Deploy ref to preproduction**.
3. Pulsar **Run workflow** e indicar el nombre de la rama, un tag o un SHA. Para validar lo último de `main`, indicar `main` explícitamente.

El workflow construye una imagen ARM64 con la etiqueta inmutable `preview-<sha>` y actualiza solo ese servicio en PRE. No fusiona código ni crea una release. Ninguna rama, incluida `main`, se despliega automáticamente.

## Rollback

```bash
cd /opt/crossfit/crossfit-deploy
API_IMAGE="ghcr.io/quiquesancho/crossfit-api:vX.Y.Z" \
WEB_IMAGE="ghcr.io/quiquesancho/crossfit-web:vX.Y.Z" \
docker compose --env-file /opt/crossfit/production/.env \
  -f environments/production/docker-compose.yml up -d --pull always
```

## Confianza en la CA interna

Caddy emite los certificados con su CA interna. Para evitar advertencias, instalar el certificado raíz en cada dispositivo cliente:

```bash
docker compose --env-file /opt/crossfit/production/.env \
  -f environments/production/docker-compose.yml \
  exec caddy cat /data/caddy/pki/authorities/local/root.crt > crossfit-root.crt
```

Instalar `crossfit-root.crt` y marcarlo como de confianza en cada dispositivo, incluido el iPhone.
