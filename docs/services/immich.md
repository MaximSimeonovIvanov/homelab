# Immich

## What It Does

Immich is a self-hosted photo and video management platform — a replacement for Google Photos. It features automatic mobile backup, facial recognition, object detection, smart search, and a timeline view.

In this homelab, Immich is the **primary photo service**. Nextcloud handles general files; Immich handles photos exclusively.

---

## Stack

Immich runs as four containers:

- `immich-server` — main application server
- `immich-ml` — machine learning container (facial recognition, object detection, CLIP search)
- `immich-db` — PostgreSQL with pgvecto-rs extension (vector search for ML features)
- `immich-redis` — Redis cache for job queues

---

## Hardware Notes

The i5-6500 CPU has no GPU. All ML inference runs on CPU via ONNX Runtime. Expect:
- Initial ML processing of a large library to take hours
- Ongoing processing of new uploads to be slow but functional
- Smart search and face recognition will work, just not instantly

---

## Compose File

See [`compose/immich/docker-compose.yml`](../../compose/immich/docker-compose.yml)

Copy `.env.example` to `.env` and fill in your values before deploying.

---

## Setup Steps

```bash
cd /opt/docker/immich
cp .env.example .env
nano .env   # Fill in DB password
docker compose up -d
```

Navigate to `http://immich.home` — complete the registration wizard to create your admin account.

---

## Nginx Proxy Manager Config

| Field            | Value           |
|------------------|-----------------|
| Domain           | `immich.home`   |
| Scheme           | `http`          |
| Forward Hostname | `192.168.0.215` |
| Forward Port     | `2283`          |
| Websockets       | ✅ On           |
| Block Exploits   | ✅ On           |

---

## Mobile App Setup

1. Install Immich app on your phone (iOS / Android)
2. Connect to WireGuard VPN
3. Server URL: `http://immich.home`
4. Log in with your admin credentials
5. Enable auto-backup in app settings

---

## Notes

- Data stored at `/opt/docker/immich/uploads` — move to HDD mount when drive is added
- ML model cache at `/opt/docker/immich/ml-cache`
- Storage template disabled by default — enable later if desired
