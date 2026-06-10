# WireGuard (wg-easy)

## What It Does

WireGuard is a modern VPN protocol. `wg-easy` is a Docker image that wraps WireGuard with a simple web UI for managing clients.

Instead of exposing your services to the internet via open ports, WireGuard lets you tunnel into your home network securely from anywhere. Once connected, your phone or laptop behaves as if it's sitting on your home LAN.

---

## Why VPN Over Reverse Proxy

|                   | VPN (WireGuard)  | Reverse Proxy |
|-------------------|------------------|---------------|
| Open ports        | 1 (UDP 51820)    | 2 (TCP 80, 443) |
| Attack surface    | Minimal          | Larger        |
| Domain required   | No               | Yes           |
| For single user   | ✅ Ideal         | Overkill      |

---

## Router Setup (Before Deploying)

You need to forward **UDP port 51820** from your router to the server.

1. Log into your router
2. Find **Port Forwarding** (may be under NAT, Firewall, or Advanced)
3. Add a rule:
   - External port: `51820`
   - Internal IP: `192.168.0.215`
   - Internal port: `51820`
   - Protocol: `UDP`

---

## Compose File

See [`compose/wireguard/docker-compose.yml`](../../compose/wireguard/docker-compose.yml)

Copy `.env.example` to `.env` and fill in your values before deploying.

Key settings in the compose file:

| Variable           | Value           | Purpose                                      |
|--------------------|-----------------|----------------------------------------------|
| `WG_HOST`          | your public IP  | Endpoint that clients connect to             |
| `WG_DEFAULT_DNS`   | 192.168.0.215   | AdGuard — gives VPN clients ad blocking and `.home` resolution |
| `WG_ALLOWED_IPS`   | 192.168.0.0/24, 10.8.0.0/24 | Split tunnel — only home network traffic goes through VPN |

---

## Setup Steps

```bash
cd /opt/docker/wireguard
cp .env.example .env
nano .env   # Fill in your public IP and a strong password
docker compose up -d
```

Navigate to `http://wireguard.home` (after NPM is configured) or `http://192.168.0.215:51821`.

Create a client for each device (phone, laptop). Download the config or scan the QR code.

---

## How DNS Works Through the VPN

This is the most non-obvious part of running WireGuard in a Docker container.

When a VPN client sends a DNS query, the traffic path is:

```
VPN client (10.8.0.2) → wg-easy container → Docker bridge (172.19.0.x) → AdGuard (port 53)
```

wg-easy NATs all traffic from VPN clients through its own Docker IP before it reaches the host. This means UFW sees DNS queries arriving from `172.19.0.x` — not from `10.8.0.x` as you might expect.

The consequence: the UFW rule allowing DNS must target `172.19.0.0/16` (the wg-easy Docker network), not `10.8.0.0/24` (the VPN subnet). A rule targeting the VPN subnet will never match and DNS will silently fail for all VPN clients.

```bash
# Correct rule — targets the Docker network, not the VPN subnet
sudo ufw allow from 172.19.0.0/16 to any port 53 comment "AdGuard DNS from wg-easy container"
```

---

## Split Tunnel vs Full Tunnel

**Split tunnel** (`WG_ALLOWED_IPS = 192.168.0.0/24, 10.8.0.0/24`): only traffic destined for the home network and VPN subnet goes through WireGuard. Regular internet (YouTube, etc.) goes directly through the client's local connection. Lower latency, current configuration.

**Full tunnel** (`WG_ALLOWED_IPS = 0.0.0.0/0`): all traffic goes through WireGuard. Useful if you want all internet traffic to appear to come from your home IP.

---

## Verification

On your phone: connect to mobile data (turn off WiFi), then enable WireGuard.

- Browse to `http://nextcloud.home` — it should load
- Browse to `http://adguard.home` — query log should show DNS queries coming through the VPN
- Check that `.home` domains resolve correctly (they won't if the UFW DNS rule is wrong)

---

## Troubleshooting

**Can't connect from outside:**
Verify the UDP 51820 port forward is correct in your router. Test with:
```bash
nmap -sU -p 51820 <your-public-ip>
```

**Connected but `.home` domains don't resolve:**
The most likely cause is the UFW DNS rule. Check that port 53 is allowed from `172.19.0.0/16`:
```bash
sudo ufw status numbered | grep 53
```
If only `192.168.0.0/24` is listed, add the Docker network rule (see above).

**Connected but no internet:**
Check `WG_ALLOWED_IPS` in the compose file. `0.0.0.0/0` routes all traffic through the VPN including internet. Use `192.168.0.0/24, 10.8.0.0/24` for split tunnel (home network only).

**WireGuard fails to start after system reboot:**
Check that `/etc/resolv.conf` is a valid symlink and that `systemd-resolved` is running. wg-quick uses systemd-resolved to apply the `DNS =` setting — if systemd-resolved is disabled, wg-quick silently skips DNS setup and may fail to start entirely.
```bash
systemctl status systemd-resolved
ls -la /etc/resolv.conf   # Should be a symlink, not a plain file
```
