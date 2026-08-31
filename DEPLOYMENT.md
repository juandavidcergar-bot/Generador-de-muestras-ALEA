# Despliegue

La documentacion de despliegue vigente se encuentra en
[`README-DEPLOY.md`](README-DEPLOY.md).

La configuracion se centraliza en `/docker/alea/config/alea.env`; no se usan
archivos `.env.production` separados por servicio ni volumenes nombrados.

Comprobacion y arranque, una vez completada la preparacion descrita en la guia:

```bash
cd /docker/alea/app
docker compose --env-file /docker/alea/config/alea.env config --quiet
docker compose --env-file /docker/alea/config/alea.env up -d --build
docker compose --env-file /docker/alea/config/alea.env ps
```

No ejecutes `docker system prune`, eliminaciones globales ni comandos contra
otros directorios de `/docker`; este proyecto solo administra `/docker/alea`.
