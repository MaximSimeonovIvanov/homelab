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

### 1. Disable systemd-resolved's stub listener (not the service itself)

Ubuntu runs `systemd-resolved` by default. It occupies port 53 on `127.0.0.53`, which conflicts with AdGuard. The fix is to disable only the stub listener — **not** the entire service. Disabling the service entirely breaks WireGuard DNS management (see `wireguard.md`).

Edit the resolved config:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d/
sudo nano /etc/systemd/resolved.conf.d/adguard.conf
```

Add:
```ini
[Resolve]
DNSStubListener=no
```

Then fix `/etc/resolv.conf` — it must remain a symlink to systemd-resolved's non-stub resolver:

```bash
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

Verify port 53 is now free:

```bash
sudo ss -tulpn | grep :53
```

Nothing should be listed on `127.0.0.53:53` — AdGuard is now free to bind to port 53.

### 2. Deploy the container

```bash
cd /opt/docker/adguard
docker compose up -d
```

### 3. Complete the web setup

Navigate to `http://192.168.0.215:3000` from any browser on your network.

The setup wizard will walk through:
- Admin username and password
- DNS listen port (use port 53)
- Web UI port (use 3000)

### 4. Configure upstream DNS

In AdGuard: **Settings → DNS settings → Upstream DNS servers**

Use Quad9 DNS over HTTPS for encrypted upstream resolution:
```
https://dns10.quad9.net/dns-query
```

### 5. Add DNS rewrites for internal services

In AdGuard Home: **Filters → DNS rewrites → Add DNS rewrite**

Add one entry per service:

| Domain            | Answer        |
|-------------------|---------------|
| `adguard.home`    | 192.168.0.215 |
| `npm.home`        | 192.168.0.215 |
| `nextcloud.home`  | 192.168.0.215 |
| `immich.home`     | 192.168.0.215 |
| `wireguard.home`  | 192.168.0.215 |
| `portainer.home`  | 192.168.0.215 |
| `homarr.home`     | 192.168.0.215 |
| `ntfy.home`       | 192.168.0.215 |
| `uptime.home`     | 192.168.0.215 |
| `music.home`      | 192.168.0.215 |
| `qbit.home`       | 192.168.0.215 |
| `prowlarr.home`   | 192.168.0.215 |
| `lidarr.home`     | 192.168.0.215 |
| `slskd.home`      | 192.168.0.215 |

### 6. Open the firewall

```bash
# Allow DNS from local network
sudo ufw allow from 192.168.0.0/24 to any port 53 comment "AdGuard DNS local network"

# Allow DNS from wg-easy Docker network (required for VPN clients — see networking.md)
sudo ufw allow from 172.19.0.0/16 to any port 53 comment "AdGuard DNS from wg-easy container"

# Allow AdGuard web UI (VPN only)
sudo ufw allow from 10.8.0.0/24 to any port 3000 comment "AdGuard VPN only"
```

### 7. Point your router DNS to AdGuard

Log into your router and change the DNS server (under DHCP settings) to `192.168.0.215`. This makes all devices use AdGuard automatically.

> **Current state:** The router is shared and still points to another DNS server. AdGuard is configured manually on individual devices. This will be fixed when on a dedicated router.

---

## Verification

- Browse to `http://adguard.home` — it should load the dashboard
- Check query log: you should see DNS queries from your devices
- Test an ad-heavy site — ads should be blocked
- Test a `.home` domain: `dig nextcloud.home @192.168.0.215` should return `192.168.0.215`

---

## Troubleshooting

**DNS not resolving after router change:**
Give devices a minute to pick up the new DNS server, or manually flush DNS:
```bash
# Linux
sudo resolvectl flush-caches

# Windows
ipconfig /flushdns
```

**Port 53 conflict with systemd-resolved:**
Do **not** disable `systemd-resolved` entirely — this will break WireGuard DNS management. Instead, disable only the stub listener as described in Step 1 above. The service must stay running.

**VPN clients can't resolve DNS:**
The UFW rule for port 53 must target `172.19.0.0/16` (the wg-easy Docker network), not `10.8.0.0/24` (the VPN subnet). wg-easy NATs all VPN traffic through its own Docker IP before it reaches the host, so UFW sees queries from the Docker network, not from the VPN client IPs. See `networking.md` for a full explanation.
