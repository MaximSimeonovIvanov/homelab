# Known Limitations & Future Improvements

This document tracks known issues, workarounds applied, and planned improvements.

---

## WireGuard — Remote DNS

**Issue:** When connecting remotely via WireGuard, `.home` hostnames don't resolve because the VPN DNS is set to `1.1.1.1` (Cloudflare) instead of AdGuard.

**Why:** Setting AdGuard (`192.168.0.215`) as VPN DNS caused Android to use it even when the tunnel was inactive, breaking all internet connectivity.

**Workaround:** Use direct IP:port to access services remotely (see [Remote Access Reference](#remote-access-reference) below).

**Fix (when moving out):** Configure AdGuard to be reachable through the VPN tunnel and switch `WG_DEFAULT_DNS` back to `192.168.0.215`.

---

## WireGuard — Android Split Tunnel

**Issue:** Android ignores simple `WG_ALLOWED_IPS` ranges and routes all traffic through the VPN.

**Workaround:** Explicit IP range list in `WG_ALLOWED_IPS` that covers all public IP space except the home network ranges.

---

## Nextcloud — Trusted Domain

**Issue:** Nextcloud blocks access via raw IP address by default.

**Fix applied:**
```bash
docker exec -it nextcloud php occ config:system:set trusted_domains 2 --value=192.168.0.215
```

**Future:** When `.home` DNS works remotely, access will be via hostname and this won't be needed.

---

## SSL/HTTPS

**Status:** Not configured. All services run on HTTP only.

**Why skipped:** Self-signed certs for `.home` domains require installing a root CA certificate on every device. Not worth the complexity at this stage.

**Fix (when moving out):** Set up a local CA with `mkcert`, install the root cert on all devices once, generate certs for all `.home` domains.

---

## Remote Access Reference

When connected via WireGuard on mobile data:

| Service | Remote URL |
|---------|-----------|
| Nextcloud | `http://192.168.0.215:8080` |
| Immich | `http://192.168.0.215:2283` |
| AdGuard | `http://192.168.0.215:3000` |
| WireGuard UI | `http://192.168.0.215:51821` |
| Portainer | `http://192.168.0.215:9000` |
| NPM | `http://192.168.0.215:81` |

---

## Pending — Requires HDD

- Move Immich uploads from SSD to HDD (`/mnt/data/immich`)
- Move Nextcloud data from SSD to HDD (`/mnt/data/nextcloud`)
- Enable Immich auto backup on mobile
- Enable Nextcloud sync on mobile and laptop
