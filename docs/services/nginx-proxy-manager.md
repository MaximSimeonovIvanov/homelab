# Nginx Proxy Manager

## What It Does

Nginx Proxy Manager (NPM) is a reverse proxy with a web UI. It runs on the server and routes traffic from clean hostnames to the correct Docker containers.

For internal services, it routes `.home` domains to the right container ports. For the public-facing site, it routes `sim-obleklo.bg` traffic arriving from Cloudflare to the application containers and handles SSL termination.

---

## How It Fits In

### Internal services
```
Browser: http://nextcloud.home
        │
   AdGuard DNS resolves nextcloud.home → 192.168.0.215
        │
   NPM (listening on port 80)
        │ matches hostname "nextcloud.home"
        │
   Nextcloud container (internal port 80)
```

### Public site
```
Browser: https://sim-obleklo.bg
        │
   Cloudflare (proxies and terminates HTTPS externally)
        │
   NPM (listening on port 443, SSL cert from Let's Encrypt)
        │ matches hostname "sim-obleklo.bg"
        │
   Next.js frontend container (internal port 3000)
```

---

## Compose File

See [`compose/nginx-proxy-manager/docker-compose.yml`](../../compose/nginx-proxy-manager/docker-compose.yml)

---

## Setup Steps

### 1. Deploy

```bash
cd /opt/docker/nginx-proxy-manager
docker compose up -d
```

### 2. Initial login

Navigate to `http://192.168.0.215:81`

Default credentials:
- Email: `admin@example.com`
- Password: `changeme`

**Change these immediately after first login.**

### 3. Add a Proxy Host (example: Nextcloud)

Go to **Hosts → Proxy Hosts → Add Proxy Host**

| Field            | Value             |
|------------------|-------------------|
| Domain Name      | `nextcloud.home`  |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `8080`            |
| Websockets       | ✅ On             |
| Block Exploits   | ✅ On             |

Leave the SSL tab alone for `.home` domains — no SSL is configured for local services yet. See `known-limitations.md`.

Repeat for each service.

---

## Proxy Hosts (Current)

| Domain              | Destination              | Notes                    |
|---------------------|--------------------------|--------------------------|
| `adguard.home`      | `192.168.0.215:3000`     |                          |
| `npm.home`          | `192.168.0.215:81`       | Unreliable — use IP:port directly |
| `nextcloud.home`    | `192.168.0.215:8080`     |                          |
| `immich.home`       | `192.168.0.215:2283`     | Websockets on            |
| `wireguard.home`    | `192.168.0.215:51821`    |                          |
| `portainer.home`    | `192.168.0.215:9000`     |                          |
| `homarr.home`       | `192.168.0.215:7575`     |                          |
| `ntfy.home`         | `192.168.0.215:5555`     |                          |
| `uptime.home`       | `192.168.0.215:3001`     |                          |
| `music.home`        | `192.168.0.215:4533`     |                          |
| `qbit.home`         | `192.168.0.215:8090`     |                          |
| `prowlarr.home`     | `192.168.0.215:9696`     |                          |
| `lidarr.home`       | `192.168.0.215:8686`     |                          |
| `slskd.home`        | `192.168.0.215:5030`     |                          |
| `sim-obleklo.bg`    | frontend container:3000  | SSL, Cloudflare proxied  |
| `api.sim-obleklo.bg`| backend container:8000   | SSL, static files via alias |

---

## Known Limitation — Custom Location Blocks

NPM has a bug affecting proxy hosts with custom Nginx location blocks: every time you save a proxy host through the UI, NPM regenerates the Nginx config and injects a `proxy_pass` directive into every custom location block, even if you set up an `alias` directive for file serving. `proxy_pass` takes precedence over `alias` in Nginx, so your custom location stops working.

**Affected:** `api.sim-obleklo.bg` — the `/static/` location uses `alias` to serve Django static files directly from disk. NPM overwrites this on every save.

**Workaround:** After any save to this proxy host, manually fix the conf file and reload:

```bash
sudo nano /opt/docker/npm/data/nginx/proxy_host/12.conf
# Remove the injected proxy_pass and proxy headers from the /static/ location block
docker exec nginx-proxy-manager nginx -s reload
```

**Permanent fix:** Use the "Custom Nginx Configuration" field in the Advanced tab of the proxy host (the gear icon). NPM does not overwrite this field on save. Move the entire `/static/` location block there.

---

## Notes

- NPM admin UI is at `http://192.168.0.215:81` — always access via direct IP:port, not `npm.home` (NPM cannot reliably proxy its own admin interface)
- After any config change via the UI, wait a few seconds for NPM to reload Nginx automatically
- Nginx config files live at `/opt/docker/npm/data/nginx/proxy_host/`
