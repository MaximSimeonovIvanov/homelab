# Nginx Proxy Manager

## What It Does

Nginx Proxy Manager (NPM) is an internal reverse proxy with a web UI. It runs on the server and routes traffic from clean `.home` hostnames to the correct Docker containers, adding HTTPS in the process.

Without NPM, you'd access services by IP and port (e.g. `http://192.168.0.215:8080`). With NPM, you get `https://nextcloud.home` — clean, secure, and easy to remember.

---

## How It Fits In

```
Browser: https://nextcloud.home
        │
   AdGuard DNS resolves nextcloud.home → 192.168.0.215
        │
   NPM (listening on port 443)
        │ matches hostname "nextcloud.home"
        │
   Nextcloud container (internal port 80)
```

NPM handles the SSL certificate, so your browser shows a valid HTTPS connection.

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

| Field            | Value                        |
|------------------|------------------------------|
| Domain Name      | `nextcloud.home`             |
| Scheme           | `http`                       |
| Forward Hostname | `nextcloud` (container name) |
| Forward Port     | `80`                         |

On the **SSL tab**: select "Request a new SSL Certificate" — for internal `.home` domains, use a self-signed cert. Your browser will warn once; you can add an exception.

Repeat for each service.

---

## Verification

- Browse to `https://nextcloud.home` — should load with a padlock (or a cert warning you can accept)
- Check NPM dashboard — host should show green/online status
