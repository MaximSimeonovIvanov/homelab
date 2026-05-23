# AdGuard Home

## What It Does

AdGuard Home is a network-wide DNS resolver and ad/tracker blocker. It replaces your router's default DNS server, meaning every device on your network benefits automatically — no per-device configuration needed.

In this homelab it also serves as the internal DNS, resolving `.home` hostnames to the server's local IP.

---

## Why AdGuard over Pi-hole

Both do the same job. AdGuard Home has a cleaner modern UI, built-in HTTPS for the admin interface, and slightly easier DNS rewrite configuration. Either works fine.

---

## Compose File

See [`compose/adguard/docker-compose.yml`](../../compose/adguard/docker-compose.yml)

---

## Setup Steps

### 1. Deploy the container

```bash
cd /opt/docker/adguard
docker compose up -d
```

### 2. Complete the web setup

Navigate to `http://192.168.0.215:3000` from any browser on your network.

The setup wizard will walk through:
- Admin username and password
- DNS listen port (use port 53)
- Web UI port (use 3000 or 80)

### 3. Point your router DNS to AdGuard

Log into your router and change the DNS server (under DHCP settings) from your ISP's default to `192.168.0.215`. This makes all devices use AdGuard automatically.

> Some routers call this "Primary DNS" under LAN or DHCP settings.

### 4. Add DNS rewrites for internal services

In AdGuard Home: **Filters → DNS rewrites → Add DNS rewrite**

Add one entry per service:

| Domain            | Answer        |
|-------------------|---------------|
| `nextcloud.home`  | 192.168.0.215 |
| `immich.home`     | 192.168.0.215 |
| `npm.home`        | 192.168.0.215 |
| `wireguard.home`  | 192.168.0.215 |
| `adguard.home`    | 192.168.0.215 |

---

## Verification

- Browse to `http://adguard.home` — it should load the dashboard
- Check query log: you should see DNS queries from devices on your network
- Test an ad-heavy site — ads should be blocked

---

## Troubleshooting

**DNS not resolving after router change:**  
Give devices a minute to pick up the new DNS server, or manually flush DNS on your laptop (`sudo systemd-resolve --flush-caches` on Linux, `ipconfig /flushdns` on Windows).

**Port 53 conflict:**  
Ubuntu may have `systemd-resolved` using port 53. Fix:
```bash
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```
Then redeploy AdGuard.
