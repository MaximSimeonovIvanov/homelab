# Ntfy

## What It Does

Ntfy is a self-hosted push notification service. It lets other services (like Uptime Kuma) send notifications directly to your phone without going through any third-party server like Telegram or Pushover.

It works on a pub/sub model — services **publish** messages to a topic, and your phone **subscribes** to that topic and receives them instantly.

In this homelab, Ntfy is used exclusively to receive alerts from Uptime Kuma when a service goes down.

---

## Why Ntfy over Alternatives

- **Fully self-hosted** — notifications never touch an external server
- **F-Droid app** — no Google Play required
- **Simple** — no complex setup, just topics and credentials
- **Actively maintained** — more active development than Gotify (the main alternative)

---

## Compose File

See [`compose/ntfy/docker-compose.yml`](../../compose/ntfy/docker-compose.yml)

---

## Configuration

Key environment variables in the compose file:

| Variable | Value | Purpose |
|----------|-------|---------|
| `NTFY_BASE_URL` | `http://192.168.0.215:5555` | Public address of the Ntfy server |
| `NTFY_LISTEN_HTTP` | `:80` | Port Ntfy listens on inside the container |
| `NTFY_AUTH_DEFAULT_ACCESS` | `deny-all` | Blocks unauthenticated access — required for security |
| `NTFY_AUTH_FILE` | `/etc/ntfy/user.db` | Where user credentials are stored |

---

## Setup Steps

### 1. Deploy the container

```bash
cd /opt/docker/ntfy
docker compose up -d
```

### 2. Create an admin user

```bash
docker exec -it ntfy ntfy user add --role=admin max
```

You will be prompted for a password.

### 3. Open the firewall port

```bash
sudo ufw allow 5555/tcp
```

### 4. Add DNS rewrite and NPM proxy host

In AdGuard: **Filters → DNS Rewrites** → add `ntfy.home` → `192.168.0.215`

In NPM: add proxy host `ntfy.home` → `192.168.0.215:5555`

### 5. Set up the Android app

- Install **Ntfy** from F-Droid
- Add server: `http://192.168.0.215:5555`
- Log in with your credentials
- Subscribe to topic: `homelab-alerts`
- **Requires WireGuard to be connected when away from home**

---

## Useful Commands

```bash
# Add a user
docker exec -it ntfy ntfy user add --role=admin <username>

# List users
docker exec -it ntfy ntfy user list

# Send a test notification manually
curl -u 'max:PASSWORD' -d "Test message" http://192.168.0.215:5555/homelab-alerts
```

---

## Verification

Send a test notification via curl (above) and confirm it arrives on your phone.

---

## Known Limitations

**Web UI shows "Notifications not supported" warning** — browser push notifications require HTTPS. This warning only affects the web UI. The Android app works fine over HTTP. Will be resolved after SSL is configured post-move.

**Requires WireGuard when remote** — since Ntfy is not exposed to the internet, your phone needs to be on the VPN to receive notifications when away from home.
