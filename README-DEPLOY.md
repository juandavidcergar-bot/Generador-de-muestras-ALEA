# Despliegue de Alea en `/docker/alea`

Esta guia instala el frontend, backend y PostgreSQL sin afectar otros proyectos
del servidor. El frontend se publica en el puerto 8080; el backend queda ligado
a loopback en el puerto 3001 y PostgreSQL no publica ningun puerto.

Los archivos CSV/XLSX se procesan en el navegador y no se almacenan en el
servidor. La unica informacion funcional persistente es PostgreSQL. Los logs se
envian a stdout/stderr y se consultan con Docker Compose; no existe un directorio
persistente de uploads ni de logs.

## 1. Requisitos

- Linux con `/docker` ya creado.
- Docker Engine y el plugin Docker Compose v2.20 o posterior. La version
  minima tambien permite usar el alias `docker-compose.prod.yml`, basado en
  `include`; los comandos de esta guia usan siempre el archivo canonico.
- Git, Bash, `getent`, `readlink`, `sha256sum`, `install`, `mktemp`, `ln`,
  `flock` (normalmente provisto por `util-linux`), `curl` y `openssl`.
- Usuario autorizado para usar Docker. El grupo `docker` equivale practicamente
  a acceso root; apliquese segun la politica del servidor.
- Salida hacia el servidor SMTP configurado por la aplicacion.
- Inicialmente, una IP alcanzable por los usuarios.
- Posteriormente, DNS para `alea.sesna.gob.mx`, certificado TLS y Nginx del host.

Verificacion inicial:

```bash
docker --version
docker compose version
git --version
```

## 2. Estructura persistente

La estructura minima es:

```text
/docker/alea/
├── app/                       # checkout del repositorio
├── config/
│   └── alea.env              # configuracion y secretos, modo 0600
├── database/
│   └── postgres/             # bind mount de PostgreSQL
└── backups/
    └── postgres/             # archives .dump y checksums SHA-256
```

Todos los ejemplos usan la raiz predeterminada `/docker/alea`. Puede cambiarse,
pero solamente por un unico hijo directo de `/docker` cuyo nombre cumpla
`[A-Za-z0-9][A-Za-z0-9._-]*`; nunca use `/docker`, `..`, otra profundidad ni un
enlace simbolico. Mantenga el mismo valor en `ALEA_ROOT` dentro de `alea.env`.
Los scripts no ejecutan ese archivo como codigo, por lo que tambien debe exportar
la raiz elegida antes de invocarlos (y sustituir las rutas literales de esta
guia):

```bash
export ALEA_ROOT=/docker/alea-produccion
bash scripts/backup.sh
# Para prepare-server.sh, que normalmente se ejecuta mediante sudo:
sudo env ALEA_ROOT="${ALEA_ROOT}" ALEA_OWNER="$USER" ALEA_GROUP="$(id -gn)" \
  bash scripts/prepare-server.sh
```

Si el repositorio aun no esta instalado, cree solo la raiz y clone dentro de
`app`. Sustituya `<URL_DEL_REPOSITORIO>` por la URL real:

```bash
if sudo test -L /docker/alea || \
   { sudo test -e /docker/alea && ! sudo test -d /docker/alea; }; then
  echo 'ERROR: /docker/alea existe y no es un directorio regular.' >&2
  exit 1
elif ! sudo test -d /docker/alea; then
  sudo install -d -m 0750 -o "$USER" -g "$(id -gn)" /docker/alea
fi
git clone <URL_DEL_REPOSITORIO> /docker/alea/app
cd /docker/alea/app
sudo env ALEA_OWNER="$USER" ALEA_GROUP="$(id -gn)" \
  bash scripts/prepare-server.sh
```

Si el checkout ya esta en `/docker/alea/app`, ejecute solamente:

```bash
cd /docker/alea/app
sudo env ALEA_OWNER="$USER" ALEA_GROUP="$(id -gn)" \
  bash scripts/prepare-server.sh
```

El script no elimina contenido, no toca otros directorios de `/docker` y no
cambia permisos o propietarios de directorios que ya existan. Si encuentra una
ruta existente que no es un directorio regular, se detiene.

## 3. Configuracion central

La configuracion de servidor se guarda fuera del checkout:

```text
/docker/alea/config/alea.env
```

Copie la plantilla versionada y restrinja sus permisos:

```bash
cd /docker/alea/app
sudo cp --no-clobber -- \
  .env.server.example /docker/alea/config/alea.env
sudo chown "$USER":"$(id -gn)" /docker/alea/config/alea.env
sudo chmod 0600 /docker/alea/config/alea.env
nano /docker/alea/config/alea.env
```

`cp --no-clobber` conserva un `alea.env` existente y evita perder secretos al
repetir el procedimiento.

No copie secretos reales al repositorio. Antes de arrancar, sustituya todos los
placeholders y complete como minimo:

- `ALEA_ROOT=/docker/alea`
- `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD`
- `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`
- Credenciales SMTP y `SEED_ADMIN_*`
- `BASE_PATH`, `API_BASE_PATH`, `API_BASE_URL`
- `PUBLIC_FRONTEND_URL`, `PUBLIC_BACKEND_URL`, `CORS_ORIGINS`
- `TRUST_PROXY`, `COOKIE_SECURE`, `COOKIE_DOMAIN`, `COOKIE_NAME`
- `FRONTEND_BIND_ADDRESS`, `FRONTEND_PORT`
- `BACKEND_BIND_ADDRESS`, `BACKEND_PORT`

Mantenga `DB_RUN_MIGRATIONS=true` y `DB_SYNCHRONIZE=false`. El backend aplica
las migraciones versionadas antes de aceptar trafico. Cree siempre un respaldo
antes de actualizar codigo o imagenes.

Genere secretos independientes, largos y aleatorios. Por ejemplo:

```bash
openssl rand -base64 48
openssl rand -base64 48
```

Compruebe que el archivo no sea legible por otros usuarios:

```bash
chmod 0600 /docker/alea/config/alea.env
stat -c '%a %U:%G %n' /docker/alea/config/alea.env
```

## 4. Primer arranque mediante IP

Mientras no exista Nginx, configure valores equivalentes a los siguientes.
Sustituya `IP_DEL_SERVIDOR` por la IP real, sin escribirla en el codigo fuente:

```env
APP_PROTOCOL=http
APP_HOST=IP_DEL_SERVIDOR
FRONTEND_BIND_ADDRESS=0.0.0.0
FRONTEND_PORT=8080
BACKEND_BIND_ADDRESS=127.0.0.1
BACKEND_PORT=3001
PUBLIC_FRONTEND_URL=http://IP_DEL_SERVIDOR:8080
PUBLIC_BACKEND_URL=http://IP_DEL_SERVIDOR:8080/api
BASE_PATH=
API_BASE_PATH=/api
API_BASE_URL=/api
CORS_ORIGINS=http://IP_DEL_SERVIDOR:8080
TRUST_PROXY=false
COOKIE_SECURE=false
COOKIE_DOMAIN=
```

Valide Compose antes de construir:

```bash
cd /docker/alea/app
docker compose \
  --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml \
  config --quiet
```

Construya y levante los servicios:

```bash
cd /docker/alea/app
docker compose \
  --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml \
  build --pull
docker compose \
  --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml \
  up -d
```

Compruebe estado, healthchecks y acceso:

```bash
docker compose \
  --env-file /docker/alea/config/alea.env \
  -f /docker/alea/app/docker-compose.yml \
  ps
curl --fail --show-error --head http://127.0.0.1:8080/
curl --fail --show-error http://127.0.0.1:8080/_alea_health
curl --fail --show-error http://127.0.0.1:3001/health
```

Desde otro equipo abra:

```text
http://IP_DEL_SERVIDOR:8080/
```

Inicie sesion con el administrador inicial, cambie inmediatamente su
contrasena desde la aplicacion y verifique el acceso. Despues puede dejar
`SEED_ADMIN_EMAIL=` y `SEED_ADMIN_PASSWORD=` vacios en `alea.env` y recrear el
backend; con una base que ya contiene usuarios esas variables no se vuelven a
usar:

```bash
docker compose --env-file /docker/alea/config/alea.env \
  -f /docker/alea/app/docker-compose.yml up -d --force-recreate backend
```

## 5. Operacion diaria

Todos los comandos siguientes usan el Compose canonico:

```bash
cd /docker/alea/app
```

Estado:

```bash
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml ps
```

Logs en stdout/stderr, sin archivos persistentes de aplicacion:

```bash
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml logs --tail=200
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml logs --follow --tail=200 backend
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml logs --follow --tail=200 frontend
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml logs --follow --tail=200 db
```

Reinicio de un servicio:

```bash
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml restart backend
```

Reconciliar los tres servicios con la configuracion actual:

```bash
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml up -d
```

## 6. Actualizacion

1. Cree un respaldo antes de cambiar imagenes o codigo:

   ```bash
   cd /docker/alea/app
   bash scripts/backup.sh
   ```

2. Verifique que no existan cambios locales inesperados y actualice solo por
   avance directo:

   ```bash
   git status --short
   git fetch origin
   git pull --ff-only
   ```

3. Valide, reconstruya y aplique:

   ```bash
   docker compose --env-file /docker/alea/config/alea.env \
     -f docker-compose.yml config --quiet
   docker compose --env-file /docker/alea/config/alea.env \
     -f docker-compose.yml build --pull
   docker compose --env-file /docker/alea/config/alea.env \
     -f docker-compose.yml up -d
   docker compose --env-file /docker/alea/config/alea.env \
     -f docker-compose.yml ps
   ```

4. Revise logs y las funciones principales. No use `docker system prune` en un
   servidor compartido.

## 7. Respaldos

El script crea un archive PostgreSQL custom-format con permisos 0600, lo valida
con `pg_restore --list` y genera un `.sha256`. Publica primero el checksum y al
final el dump mediante enlaces atomicos que fallan si el destino ya existe; no
sobrescribe ni elimina respaldos anteriores:

```bash
cd /docker/alea/app
bash scripts/backup.sh
```

Los resultados se guardan exclusivamente en:

```text
/docker/alea/backups/postgres/alea-postgres-AAAAMMDDTHHMMSSZ.dump
/docker/alea/backups/postgres/alea-postgres-AAAAMMDDTHHMMSSZ.dump.sha256
```

Si el nombre del mismo segundo ya existe, se agrega un sufijo numerico. Tanto
`backup.sh` como `restore.sh` mantienen un `flock` exclusivo y no bloqueante
sobre `/docker/alea/backups/postgres`: una segunda operacion concurrente falla de
forma inmediata en vez de esperar o interferir con la primera.

Liste y verifique un respaldo sin mostrar secretos:

```bash
ls -lh /docker/alea/backups/postgres/
cd /docker/alea/backups/postgres
sha256sum --check alea-postgres-AAAAMMDDTHHMMSSZ.dump.sha256
```

Defina fuera de estos scripts una politica de retencion y una copia cifrada en
otro equipo o almacenamiento. Nunca automatice el borrado de otros proyectos de
`/docker`.

## 8. Restauracion

La restauracion acepta solamente archivos `.dump` cuya ruta canonica permanezca
dentro de `/docker/alea/backups/postgres`. Cuando existe el checksum, exige una
sola entrada cuyo nombre coincida exactamente con el dump seleccionado y compara
su SHA-256. Despues valida el archive, detiene `backend`, crea primero un respaldo
preventivo y usa una sola transaccion. Un trap intenta reiniciar y esperar el
healthcheck del backend tanto si `pg_restore` termina bien como si falla. Durante
la operacion el frontend puede responder temporalmente 502/503.

Use este procedimiento solamente con un dump compatible con la version de la
aplicacion, PostgreSQL y las migraciones desplegadas. `pg_restore --clean` elimina
los objetos incluidos en el archive, pero no objetos posteriores que no figuren
en el dump; por ello no representa un downgrade ni una reconstruccion exacta de
la base. Para recuperacion exacta, aprovisione una base o instancia vacia y
separada con versiones compatibles, restaure y valide alli, y haga despues un
cambio controlado. No elimine manualmente el directorio activo de PostgreSQL.

La frase literal de confirmacion evita ejecuciones accidentales:

```bash
cd /docker/alea/app
bash scripts/restore.sh \
  --backup /docker/alea/backups/postgres/alea-postgres-AAAAMMDDTHHMMSSZ.dump \
  --confirm RESTORE_ALEA
```

Despues compruebe:

```bash
docker compose --env-file /docker/alea/config/alea.env \
  -f /docker/alea/app/docker-compose.yml ps
docker compose --env-file /docker/alea/config/alea.env \
  -f /docker/alea/app/docker-compose.yml logs --tail=200 backend db
```

## 9. Prueba de persistencia

1. Cree o identifique un registro reconocible desde la aplicacion.
2. Cree un respaldo con `bash scripts/backup.sh`.
3. Anote el estado de los servicios y detengalos sin `-v`:

   ```bash
   cd /docker/alea/app
   docker compose --env-file /docker/alea/config/alea.env \
     -f docker-compose.yml down
   ```

4. Confirme que el bind mount sigue teniendo contenido y vuelva a levantar:

   ```bash
   sudo find /docker/alea/database/postgres \
     -mindepth 1 -maxdepth 1 -print -quit | grep -q .
   docker compose --env-file /docker/alea/config/alea.env \
     -f docker-compose.yml up -d
   docker compose --env-file /docker/alea/config/alea.env \
     -f docker-compose.yml ps
   ```

5. Entre nuevamente y compruebe el registro identificado en el paso 1.

No use `docker compose down -v`. Aunque el almacenamiento objetivo es un bind
mount, omitir `-v` evita afectar por error otros recursos declarados en futuras
versiones.

## 10. Firewall

Acceso provisional por IP:

- Abrir TCP 8080 solo desde las redes o IPs usuarias necesarias.
- Mantener TCP 3001 ligado a `127.0.0.1`; no abrirlo en el firewall.
- No abrir TCP 5432. PostgreSQL es exclusivamente interno.
- Mantener el puerto administrativo SSH limitado conforme a la politica local.

Con Nginx y HTTPS:

- Abrir TCP 80 y 443.
- Cambiar `FRONTEND_BIND_ADDRESS=127.0.0.1` y cerrar 8080 externamente.
- Mantener backend 3001 en loopback y PostgreSQL 5432 sin publicar.

## 11. Dominio y dos paths de Nginx

Cuando se conozcan las rutas, elija dos paths distintos/no iguales y sin slash
final. Por ejemplo, `/alea` y `/alea-api`. Actualice
`/docker/alea/config/alea.env`:

```env
APP_PROTOCOL=https
APP_HOST=alea.sesna.gob.mx
FRONTEND_BIND_ADDRESS=127.0.0.1
FRONTEND_PORT=8080
BACKEND_BIND_ADDRESS=127.0.0.1
BACKEND_PORT=3001
PUBLIC_FRONTEND_URL=https://alea.sesna.gob.mx/RUTA_FRONTEND
PUBLIC_BACKEND_URL=https://alea.sesna.gob.mx/RUTA_BACKEND
BASE_PATH=/RUTA_FRONTEND
API_BASE_PATH=/RUTA_BACKEND
API_BASE_URL=/RUTA_BACKEND
CORS_ORIGINS=https://alea.sesna.gob.mx
TRUST_PROXY=true
COOKIE_SECURE=true
COOKIE_DOMAIN=alea.sesna.gob.mx
```

Las variables que dependen de los paths pendientes son `PUBLIC_FRONTEND_URL`,
`PUBLIC_BACKEND_URL`, `BASE_PATH`, `API_BASE_PATH` y `API_BASE_URL`. Si la ruta
de API cambia, el backend deriva automaticamente el path de la cookie de refresh
a partir de `API_BASE_PATH`.

Recree los servicios para aplicar configuracion runtime; no use `build` solo por
un cambio de dominio o paths:

```bash
cd /docker/alea/app
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml config --quiet
docker compose --env-file /docker/alea/config/alea.env \
  -f docker-compose.yml up -d --force-recreate frontend backend
```

En Debian o Ubuntu con Nginx administrado por `systemd`, instale la plantilla del
host sin sobrescribir una configuracion ya existente:

```bash
sudo cp --no-clobber -- \
  /docker/alea/app/deploy/nginx/alea.conf.example \
  /etc/nginx/sites-available/alea.conf
sudoedit /etc/nginx/sites-available/alea.conf
```

Sustituya `__FRONTEND_PATH__`, `__BACKEND_PATH__`, `__TLS_CERTIFICATE__` y
`__TLS_CERTIFICATE_KEY__`. La plantilla usa `proxy_pass` sin URI final para
preservar la ruta completa hacia frontend `127.0.0.1:8080` y backend
`127.0.0.1:3001`; no agregue slash al `proxy_pass`.

Habilite y valide Nginx solo despues de reemplazar todos los tokens. La prueba
de sintaxis se ejecuta despues de crear el enlace para que Nginx cargue realmente
el sitio nuevo antes del `reload`:

```bash
sudo grep -n '__[A-Z_]*__' /etc/nginx/sites-available/alea.conf
sudo ln -s /etc/nginx/sites-available/alea.conf \
  /etc/nginx/sites-enabled/alea.conf
sudo nginx -t
sudo systemctl reload nginx
```

El primer `grep` no debe devolver coincidencias. Si el sitio ya esta habilitado,
no vuelva a crear el enlace; ejecute solamente `nginx -t` y el reload.

En una distribucion cuyo `nginx.conf` incluya `/etc/nginx/conf.d/*.conf`, use en
su lugar el siguiente flujo; no cree enlaces en `sites-enabled` y no combine
ambos metodos:

```bash
sudo cp --no-clobber -- \
  /docker/alea/app/deploy/nginx/alea.conf.example \
  /etc/nginx/conf.d/alea.conf
sudoedit /etc/nginx/conf.d/alea.conf
sudo grep -n '__[A-Z_]*__' /etc/nginx/conf.d/alea.conf
sudo nginx -t
sudo nginx -s reload
```

Tambien aqui el `grep` no debe devolver coincidencias. Si la distribucion usa
otro gestor de servicios, sustituya solamente el comando de reload por el
procedimiento documentado por esa distribucion.

Pruebas finales:

```bash
curl --fail --show-error --head \
  https://alea.sesna.gob.mx/RUTA_FRONTEND/
curl --fail --show-error http://127.0.0.1:3001/health
```

No afirme el despliegue como concluido hasta probar login, renovacion de sesion,
enlaces enviados por correo, CORS, persistencia y restauracion con los dos paths
reales.

## 12. Cookies, CORS y CSRF

El refresh token se guarda en una cookie `HttpOnly` con `SameSite=Lax`; su flag
`Secure`, dominio, nombre y path efectivo dependen de la configuracion de
servidor. `CORS_ORIGINS` debe enumerar origenes exactos y nunca usar `*` junto
con credenciales. Bajo el diseno previsto, frontend y API comparten origen y
`SameSite=Lax` aporta la barrera CSRF, por lo que no se incorpora un token CSRF
separado. Si se habilitan origenes cruzados o se cambia `SameSite`, hay que
reevaluar y probar explicitamente la proteccion CSRF antes de publicar.
