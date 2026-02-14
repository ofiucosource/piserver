# PiServer

Aplicación mínima con FastAPI para correr en Raspberry Pi usando Docker Compose.

## Endpoints

- `/` Home HTML
- `/health` Estado del servicio

## Ejecutar local con Docker

```bash
docker compose up -d --build
```

Abrir: `http://localhost:8000`

## Desplegar en Raspberry Pi

Este repo está listo para usarse con tu script de despliegue:

```bash
sudo GIT_REPO_URL='https://github.com/ofiucosource/piserver.git' GIT_BRANCH='main' PI_USER='pibardo' bash Untitled-1.sh
```
