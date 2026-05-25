# Uptime Kuma

## What It Does

Uptime Kuma is a self-hosted monitoring tool. It continuously checks whether your services are reachable and alerts you if something goes down.

Without it, a crashed container would go unnoticed until you manually tried to use that service. Uptime Kuma acts as a watchdog — it pings all your services on a schedule and fires a notification (via Ntfy) the moment one stops responding.

---

## Compose File

See [`compose/uptime-kuma/docker-compose.yml`](../../compose/uptime-kuma/docker-compose.yml)

---

## Setup Steps

### 1. Deploy the container

```bash
cd /opt/docker/uptime-kuma
docker compose up -d
```

### 2. Open the firewall port

```bash
sudo ufw allow 3001/tcp
```

### 3. Create admin account

Navigate to `http://192.168.0.215:3001` — you will be prompted to create an admin username and password on first visit.

### 4. Add DNS rewrite and NPM proxy host

In AdGuard: **Filters → DNS Rewrites** → add `uptime.home` → `192.168.0.215`

In NPM: add proxy host `uptime.home` → `192.168.0.215:3001`

### 5. Configure Ntfy notifications

In Uptime Kuma: **Settings → Notifications → Setup Notification**

| Field | Value |
|-------|-------|
| Notification Type | Ntfy |
| Name | Homelab Ntfy |
| ntfy Server URL | `http://192.168.0.215:5555` |
| Topic | `homelab-alerts` |
| Authentication Method | Username & Password |
| Username | `max` |
| Password | your Ntfy password |

Enable **Apply by default** and **Apply to all existing monitors**, then click Test and Save.

### 6. Add monitors

Add one HTTP monitor per service:

| Name | URL | Interval |
|------|-----|----------|
| AdGuard | `http://192.168.0.215:3000` | 1800s |
| Nginx Proxy Manager | `http://192.168.0.215:81` | 1800s |
| WireGuard | `http://192.168.0.215:51821` | 1800s |
| Nextcloud | `http://192.168.0.215:8080` | 1800s |
| Immich | `http://192.168.0.215:2283` | 1800s |
| Portainer | `http://192.168.0.215:9000` | 1800s |
| Homarr | `http://192.168.0.215:7575` | 1800s |
| Ntfy | `http://192.168.0.215:5555` | 1800s |
| Uptime Kuma | `http://192.168.0.215:3001` | 1800s |

> Interval is set to 1800 seconds (30 minutes) — appropriate for a homelab that isn't running 24/7.

---

## Verification

- All monitors should show green after the first check cycle
- Send a test notification from the Ntfy notification config to confirm alerts work

---

## Notes

- Uptime Kuma stores all data in a local SQLite database at `/opt/docker/uptime-kuma/data/`
- No external database needed — single container, very lightweight
- Monitoring only works while the server is running — expected for a homelab setup
