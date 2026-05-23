# WireGuard (wg-easy)

## What It Does

WireGuard is a modern VPN protocol. `wg-easy` is a Docker image that wraps WireGuard with a simple web UI for managing clients.

Instead of exposing your services to the internet via open ports, WireGuard lets you tunnel into your home network securely from anywhere. Once connected, your phone or laptop behaves as if it's sitting on your home LAN.

---

## Why VPN Over Reverse Proxy

| | VPN (WireGuard) | Reverse Proxy |
|---|---|---|
| Open ports | 1 (UDP 51820) | 2 (TCP 80, 443) |
| Attack surface | Minimal | Larger |
| Domain required | No | Yes |
| For single user | ✅ Ideal | Overkill |

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

---

## Setup Steps

```bash
cd /opt/docker/wireguard
cp .env.example .env
nano .env   # Fill in your public IP and a strong password
docker compose up -d
```

Navigate to `https://wireguard.home` (after NPM is configured) or `http://192.168.0.215:51821`.

Create a client for each device (phone, laptop). Download the config or scan the QR code.

---

## Verification

On your phone: connect to mobile data (turn off WiFi), then enable WireGuard.

- Browse to `https://nextcloud.home` — it should load
- Check `http://adguard.home` → query log shows DNS queries coming through the VPN

---

## Troubleshooting

**Can't connect from outside:**  
Verify the UDP 51820 port forward is correct in your router. Test with `nmap -sU -p 51820 your-public-ip` from another network.

**Connected but no internet:**  
Check `ALLOWED_IPS` in the client config. `0.0.0.0/0` routes all traffic through the VPN. Use `192.168.0.0/24, 10.8.0.0/24` if you only want to access home network (split tunnel).
