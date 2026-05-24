# Portainer

## What It Does

Portainer is a web UI for managing Docker. It gives you a visual dashboard to monitor containers, view logs, restart services, and inspect volumes — without using the terminal.

---

## Compose File

See [`compose/portainer/docker-compose.yml`](../../compose/portainer/docker-compose.yml)

---

## Setup Steps

```bash
cd /opt/docker/portainer
docker compose up -d
```

Navigate to `http://portainer.home` immediately after starting — Portainer times out after a few minutes waiting for initial setup.

Create your admin account and save the credentials in a password manager.

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `portainer.home`  |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `9000`            |
| Block Exploits   | ✅ On             |

---

## Notes

- Portainer mounts `/var/run/docker.sock` to communicate with the Docker daemon directly
- If you get locked out, reset the admin password with the helper container:

```bash
docker stop portainer
docker run --rm -v /opt/docker/portainer/data:/data portainer/helper-reset-password
docker start portainer
```
