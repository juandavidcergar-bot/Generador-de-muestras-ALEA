# Insaculación - Muestreo Aleatorio Simple

## 📚 Documentación

[![Manual Técnico y de Instalación](https://img.shields.io/badge/Manual-Técnico%20y%20de%20Instalación-1565C0?style=for-the-badge&logo=readthedocs&logoColor=white)](https://docs.google.com/document/d/1LZ--V0NFSgH-KGyda4ie8BgJ9pP9p_xe/edit) 👈 Click aquí!

Sistema web para gestion de usuarios auditores y administradores, carga de padrones de empleados, generacion de muestras aleatorias auditables y verificacion posterior de integridad.

Este repositorio contiene:
- `frontend/`: aplicacion React + Vite
- `backend/`: API NestJS + PostgreSQL
- `docker-compose.yml`: despliegue de servidor por Docker Compose
- `README-DEPLOY.md`: instalacion, Nginx, respaldos y restauracion

> La guia operativa vigente es `README-DEPLOY.md`. No uses instrucciones de
> despliegue copiadas de versiones anteriores del proyecto.

---

## 1) Alcance funcional

### Modulo de administracion
- Alta de usuarios por invitacion (correo)
- Reenvio de invitacion
- Edicion y baja de usuarios
- Roles: `admin` y `auditor`

### Modulo de auditoria
- Carga de archivo de empleados (`.csv`, `.xlsx`, `.xls`)
- Generacion de muestra aleatoria con semilla
- Registro de bitacora de muestreos en base de datos por usuario
- Verificacion de muestra usando semilla + hash resultado
- Exportacion de resultados a Excel

### Trazabilidad
Cada muestreo guarda:
- fecha/hora
- tamano de muestra
- seed
- hash de archivo
- hash de resultado
- metodo utilizado

---

## 2) Arquitectura

- Frontend: React 19 + Vite + MUI + Tailwind
- Backend: NestJS + TypeORM
- DB: PostgreSQL 16
- Auth: JWT Bearer
- Email: SMTP para invitaciones y recuperacion

Comunicacion:
- El Nginx interno sirve el frontend y enruta `API_BASE_PATH` al backend.
- `BASE_PATH`, `API_BASE_PATH` y `API_BASE_URL` se aplican al arrancar el
  contenedor; cambiar IP, dominio o subrutas no requiere recompilar la imagen.

---

## 3) Requisitos

### Opcion Docker (recomendada)
- Docker Desktop reciente

### Opcion local sin Docker
- Node.js 22+
- pnpm (via Corepack)
- PostgreSQL 16+

---

## 4) Despliegue con Docker

El despliegue soportado usa `/docker/alea`, bind mounts y un unico archivo de
configuracion en `/docker/alea/config/alea.env`. La configuracion inicial por
IP y la migracion posterior a `alea.sesna.gob.mx` estan documentadas paso a
paso en `README-DEPLOY.md`.

Resumen del arranque, despues de preparar directorios y secretos:

```bash
cd /docker/alea/app
docker compose --env-file /docker/alea/config/alea.env config --quiet
docker compose --env-file /docker/alea/config/alea.env up -d --build
docker compose --env-file /docker/alea/config/alea.env ps
```

PostgreSQL no publica ningun puerto. El backend se liga solo a `127.0.0.1` y
el frontend se publica provisionalmente en el valor de `FRONTEND_PORT`.

---

## 6) Instalacion local (sin Docker)

### 6.1 Backend

```bash
cd backend
corepack enable
pnpm install
pnpm run start:dev
```

### 6.2 Frontend

```bash
cd frontend
corepack enable
pnpm install
pnpm run dev
```

### 6.3 Base de datos
Configura PostgreSQL y variables de entorno del backend (`backend/.env`).

---

## 7) Manual de usuario

### 7.1 Usuario administrador
1. Inicia sesion con cuenta `admin`.
2. Usa el panel para:
   - invitar usuario
   - reenviar invitacion
   - editar usuario
   - eliminar usuario

### 7.2 Usuario auditor
1. Inicia sesion con cuenta `auditor`.
2. Carga archivo de empleados.
3. En `Generar`, define tamano y ejecuta muestreo.
4. Revisa datos de auditoria y descarga Excel.
5. En `Historial`, usa `Ver` para cargar un muestreo previo en `Verificar`.
6. En `Verificar`, valida semilla/hash contra los datos cargados.

---

## 8) Estructura del repositorio

```text
.
|-- backend/
|   |-- src/
|   |   |-- auth/
|   |   |-- users/
|   |   `-- sampling-history/
|   |-- Dockerfile
|   `-- README.md
|-- frontend/
|   |-- components/
|   |-- src/
|   |-- Dockerfile
|   |-- Dockerfile.prod
|   `-- README.md
|-- docker-compose.yml
|-- docker-compose.prod.yml
|-- deploy/
|-- scripts/
|-- README-DEPLOY.md
`-- README.md
```

---

## 9) Operacion y mantenimiento

### Reinicio
```bash
docker compose --env-file /docker/alea/config/alea.env restart
```

### Parar contenedores
```bash
docker compose --env-file /docker/alea/config/alea.env down
```

La base de datos permanece en `/docker/alea/database/postgres` porque se usa un
bind mount. El procedimiento de comprobacion de persistencia esta en
`README-DEPLOY.md`.

### Ver logs
```bash
docker compose --env-file /docker/alea/config/alea.env logs -f
docker compose --env-file /docker/alea/config/alea.env logs -f backend
docker compose --env-file /docker/alea/config/alea.env logs -f frontend
```

---

## 10) Solucion de problemas

### Error de puerto ocupado
Modifica `FRONTEND_PORT` o `BACKEND_PORT` en
`/docker/alea/config/alea.env`. PostgreSQL no se publica al host.

### Frontend no conecta a API
- Revisa `API_BASE_PATH`, `API_BASE_URL` y `BACKEND_INTERNAL_URL`.
- Comprueba `docker compose ... ps` y el healthcheck del backend.

### No llegan correos
- Revisa las variables `MAIL_*` del archivo central de servidor.
- Revisa logs del backend

No se documentan comandos de limpieza global: podrían afectar otros proyectos
del servidor.

---

## 11) Documentacion adicional

- Despliegue: `README-DEPLOY.md`
- Frontend: `frontend/README.md`
- Backend: `backend/README.md`
