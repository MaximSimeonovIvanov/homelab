# Known Limitations & Current Issues

This document tracks known issues, active workarounds, and things blocked on future hardware or infrastructure changes.

---

## SSL/HTTPS — Local Services

**Status:** Not configured. All `.home` services run on HTTP only.

**Why skipped:** Self-signed certs for `.home` domains require installing a root CA certificate on every device. Not worth the complexity at this stage.

**Fix (when moving out):** Set up a local CA with `mkcert`, install the root cert on all devices once, generate certs for all `.home` domains.

**Note:** The public-facing site `sim-obleklo.bg` has proper Let's Encrypt SSL via Cloudflare DNS challenge. This limitation only applies to internal `.home` services.

---

## Router DNS — Not Pointing to AdGuard

**Status:** The router is shared. Its DNS still points to another device on the network, not to AdGuard. AdGuard is only used by devices configured manually (laptop and phone).

**Fix (when moving out):** Point the router's DHCP DNS setting to `192.168.0.215`. This gives every device on the network ad blocking and `.home` resolution automatically, without manual configuration per device.

---

## NPM — Custom Location Blocks Overwritten on Save

**Status:** Active issue affecting `api.sim-obleklo.bg` static file serving.

**What happens:** Every time a proxy host is saved through the NPM web UI, NPM regenerates the Nginx config file and injects `proxy_pass` into every custom location block, overriding any `alias` directive. This breaks static file serving for the Django admin.

**Workaround:** After any NPM proxy host save, manually edit the conf file and reload Nginx:
```bash
sudo nano /opt/docker/npm/data/nginx/proxy_host/12.conf
# Remove the injected proxy_pass from the /static/ location block
docker exec nginx-proxy-manager nginx -s reload
```

**Permanent fix:** Move the `/static/` location block to NPM's server-level "Custom Nginx Configuration" field (gear icon on the proxy host) — NPM does not overwrite this field on save.

---

## Port 2222 Open to Internet

**Status:** Port 2222 is open to the internet for GitHub Actions SSH deployments (workwear catalog CI/CD). fail2ban protects it, but it is an additional attack surface.

**Review when moving out:** Consider restricting to a known IP range if GitHub Actions publishes its IP ranges, or switching to a different deployment method.

---

## Port 22 Open to Internet

**Status:** SSH on port 22 is intentionally left open to the internet as a safety net — if WireGuard breaks while remote, SSH is the fallback to recover the server.

**Review when moving out:** Once on own router with reliable network, consider restricting port 22 to VPN only (`10.8.0.0/24`). This eliminates the last internet-facing management port.

---

## Pending — Requires HDD

The 4TB HDD has not been purchased yet. Until it is, all persistent data lives on the 128GB boot SSD.

- [ ] Install HDD, mount at `/mnt/data/`
- [ ] Move Immich uploads: `/opt/docker/immich/uploads` → `/mnt/data/immich/`
- [ ] Move Nextcloud data: `/opt/docker/nextcloud/data` → `/mnt/data/nextcloud/`
- [ ] Move music library to HDD mount, update Navidrome and Lidarr volume paths
- [ ] Enable Immich auto-backup on Pixel 9
- [ ] Enable Nextcloud sync on phone and laptop
