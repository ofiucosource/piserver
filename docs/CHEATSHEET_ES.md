# Chuleta rápida: PiServer en Raspberry Pi

## 1) Conectar a la Pi

```bash
ssh pibardo@192.168.137.139
# o por Tailscale IP
ssh pibardo@100.116.118.76
```

## 2) Ver estado general

```bash
tailscale status
docker version
docker compose version
sudo systemctl status docker --no-pager
```

## 3) Desplegar desde GitHub (flujo principal)

```bash
sudo GIT_REPO_URL='https://github.com/ofiucosource/piserver.git' \
GIT_BRANCH='main' \
PI_USER='pibardo' \
bash ~/setup-rpi.sh
```

## 4) Redeploy rápido (manual)

```bash
cd /opt/apps/piserver
git pull
docker compose up -d --build
docker compose ps
```

## 5) Logs y diagnóstico

```bash
cd /opt/apps/piserver
docker compose logs -f --tail=100
```

## 6) Reiniciar servicio

```bash
cd /opt/apps/piserver
docker compose down
docker compose up -d --build
```

## 7) Validar app

```bash
curl -s http://127.0.0.1:8000/health
```

URLs:

- `http://192.168.137.139:8000`
- `http://100.116.118.76:8000`

## 8) Error típico: permisos Docker

Si aparece `permission denied ... /var/run/docker.sock`:

```bash
sudo usermod -aG docker pibardo
newgrp docker
docker version
```

## 9) Reparación APT/DPKG (si algo se rompe)

```bash
sudo bash rpi-apt-repair.sh
# opcional forzar fsck en próximo reinicio
sudo bash rpi-apt-repair.sh --schedule-fsck
sudo reboot
```

## 10) Flujo diario recomendado

1. Desarrollar en PC y `git push`.
2. Entrar a la Pi.
3. `cd /opt/apps/piserver && git pull`.
4. `docker compose up -d --build`.
5. Revisar `docker compose ps` y `/health`.
