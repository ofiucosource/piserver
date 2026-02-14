# Runbook completo: Raspberry Pi + Docker + Tailscale + FastAPI

Este documento resume todo lo que configuramos, los problemas que aparecieron y cómo se resolvieron.
La idea es que puedas **replicar el proceso de cero** y también entender el porqué de cada paso.

---

## 1) Objetivo final

Dejar una Raspberry Pi OS Lite arm64 con:

1. Docker Engine funcionando.
2. Tailscale conectado a tu tailnet.
3. Despliegue automático/manual de una app FastAPI desde GitHub.
4. App levantada en contenedor con Docker Compose.

Resultado final alcanzado:

- Repo público: `https://github.com/ofiucosource/piserver.git`
- App desplegada en Pi en `/opt/apps/piserver`
- Servicio publicado en puerto `8000`

---

## 2) Estructura creada del proyecto

En el repo `piserver`:

- `app/main.py`: API FastAPI con `/` y `/health`
- `app/templates/index.html`: home HTML simple
- `app/static/style.css`: estilos básicos
- `requirements.txt`: dependencias Python
- `Dockerfile`: build de la app
- `compose.yaml`: servicio `piserver`
- `.gitignore`: ignorados de Python/entorno
- `README.md`: uso rápido

Arquitectura de ejecución:

- `docker compose up -d --build` construye imagen y levanta contenedor.
- Uvicorn expone `0.0.0.0:8000`.
- Host Pi publica `8000:8000`.

---

## 3) Script de bootstrap de Raspberry Pi

Script usado: `Untitled-1.sh` (renombrable a `setup-rpi-docker-tailscale.sh`).

Qué hace:

1. Valida root y usuario (`PI_USER`).
2. Instala dependencias base (`curl`, `git`, etc.).
3. Instala Docker (script oficial) + plugin compose.
4. Habilita servicio Docker.
5. Instala Tailscale y habilita `tailscaled`.
6. Hace `tailscale up` (con `TS_AUTHKEY` o interactivo).
7. Clona/actualiza repo GitHub en `/opt/apps/<app>`.
8. Busca archivo compose y levanta stack.
9. Muestra verificación final (`docker version`, `tailscale status`, etc.).

Comando tipo:

```bash
sudo GIT_REPO_URL='https://github.com/ofiucosource/piserver.git' \
GIT_BRANCH='main' \
PI_USER='pibardo' \
bash Untitled-1.sh
```

Variables soportadas:

- `PI_USER` (default: `pi` o `SUDO_USER`)
- `TS_AUTHKEY` (opcional)
- `TS_HOSTNAME` (default: hostname de la Pi)
- `GIT_REPO_URL` (**obligatoria**)
- `GIT_BRANCH` (default: `main`)
- `APP_BASE_DIR` (default: `/opt/apps`)
- `APP_NAME` (default: nombre del repo)

---

## 4) Tailscale: configuración y autenticación

### 4.1 ¿De dónde sale `TS_AUTHKEY`?

Desde panel admin de Tailscale:

- `https://login.tailscale.com/admin/settings/keys`

Opciones típicas:

- `Reusable`: útil para aprovisionar varias veces.
- `Ephemeral`: nodos temporales.
- `Preauthorized`: evita aprobación manual.

### 4.2 Flujo usado en esta puesta en marcha

Se usó modo interactivo cuando no había key:

```bash
tailscale up --hostname=<host> --ssh
```

Luego se abrió el link de login en navegador y quedó la Pi unida a la red.

Estado validado:

- Pi visible en `tailscale status`
- Con IP del rango `100.x.x.x`

---

## 5) Problemas reales que aparecieron y resolución

### Problema A: Instalación Docker parecía “pegada”

Síntoma:

- Salida extensa durante instalación/descarga de paquetes/imágenes.
- Parecía congelado por tiempo.

Diagnóstico sugerido:

```bash
ps -eo pid,cmd | grep -E "apt|dpkg|docker-ce|containerd" | grep -v grep
sudo lsof /var/lib/dpkg/lock-frontend || true
df -h /
free -h
```

Resolución aplicada:

1. Reintento de instalación.
2. Confirmación de que paquetes ya estaban instalados.
3. Validación de servicio docker activo.

---

### Problema B: corte de sesión SSH durante `apt-get install`

Síntoma:

- `client_loop: send disconnect: Connection reset`

Causa probable:

- Corte de red/SSH, no necesariamente error de Docker.

Resolución:

1. Reconectar por SSH.
2. Repetir comando de instalación.
3. Confirmar estado con `already newest version`.

---

### Problema C: `permission denied` al usar Docker sin sudo

Síntoma:

- `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`

Causa:

- Usuario aún no tenía grupo `docker` aplicado en la sesión actual.

Resolución:

```bash
sudo usermod -aG docker pibardo
newgrp docker
docker version
```

Con eso quedó cliente/servidor Docker funcionando sin sudo.

---

### Problema D: error `^M` al ejecutar script remoto

Síntoma:

- `/home/pibardo/setup-rpi.sh: line 5: $'\r': command not found`

Causa:

- Script copiado con saltos de línea Windows (`CRLF`) a Linux.

Resolución:

- Convertir archivo a `LF` antes de copiar/ejecutar.

Recomendación permanente:

- En VS Code: configurar `End of Line = LF` para scripts `.sh`.

---

## 6) Estado final validado

Validaciones ejecutadas exitosamente:

```bash
docker version
docker compose version
sudo systemctl status docker --no-pager
tailscale status
docker compose ps
```

Resultados clave:

- Docker Engine activo (`active (running)`).
- Compose disponible.
- Tailscale conectado (Pi visible en tailnet).
- Contenedor `piserver` levantado con puerto `8000`.

---

## 7) Cómo redeplegar cuando cambies código

En tu PC:

```bash
git add .
git commit -m "feat: cambios"
git push origin main
```

En la Pi:

```bash
sudo GIT_REPO_URL='https://github.com/ofiucosource/piserver.git' \
GIT_BRANCH='main' \
PI_USER='pibardo' \
bash ~/setup-rpi.sh
```

Alternativa manual rápida:

```bash
cd /opt/apps/piserver
git pull
docker compose up -d --build
```

---

## 8) Checklist de verificación rápida (1 minuto)

```bash
# Red privada overlay
 tailscale status

# Docker
 docker version
 docker compose version

# App
 docker compose -f /opt/apps/piserver/compose.yaml ps
 curl -s http://127.0.0.1:8000/health
```

Esperado:

- `status: ok` en `/health`
- contenedor `Up`

---

## 9) Buenas prácticas para continuar

1. Cambiar nombre del script `Untitled-1.sh` a `setup-rpi-docker-tailscale.sh`.
2. Mantener scripts bash en `LF`.
3. Si el repo pasa a privado, configurar Deploy Key SSH.
4. No guardar secretos en Git; usar `.env` local en la Pi.
5. Evaluar agregar workflow CI/CD para redeploy automático.

---

## 10) Comandos de recuperación rápida

### Reparar APT/DPKG

Script disponible fuera del repo app:

```bash
sudo bash rpi-apt-repair.sh
```

Con fsck programado:

```bash
sudo bash rpi-apt-repair.sh --schedule-fsck
sudo reboot
```

### Ver logs de contenedor

```bash
cd /opt/apps/piserver
docker compose logs -f --tail=100
```

### Reiniciar stack

```bash
cd /opt/apps/piserver
docker compose down
docker compose up -d --build
```

---

## 11) Línea de tiempo de esta implementación

1. Se evaluó objetivo: instalar Docker + Tailscale + despliegue desde GitHub.
2. Se detectó script base de reparación APT (`rpi-apt-repair.sh`).
3. Se construyó script de bootstrap end-to-end (`Untitled-1.sh`).
4. Se creó repo `piserver` con app FastAPI mínima y Docker Compose.
5. Se inicializó git local y push a GitHub (`main`).
6. Se ejecutó bootstrap remoto en Pi por SSH.
7. Se resolvió error CRLF (`^M`) y se reintentó.
8. Se desplegó contenedor y se verificó estado final.

---

## 12) Próximos pasos recomendados

1. Añadir GitHub Actions para redeploy en cada push a `main`.
2. Añadir HTTPS/reverse proxy (Caddy o Nginx) si expones fuera de LAN/Tailscale.
3. Agregar endpoint `/version` para trazabilidad de despliegues.
4. Implementar backup de datos si agregas base de datos.

---

Si quieres, el siguiente documento que podemos crear es un **playbook de incidentes** (qué hacer ante caída de red, SD corrupta, o contenedor en crashloop).
