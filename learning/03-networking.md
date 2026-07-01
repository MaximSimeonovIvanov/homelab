# Study Session — Chapter 3: Networking Fundamentals

> **Repo:** `homelab` (infrastructure — not the website repo)
> **Date:** June 28, 2026
> **Format:** Study-plan deep dive — explain the *why*, verify against the live box, quiz.
> **Verified against:** live `homelab` server over WireGuard + SSH, June 28, 2026.
> **Status:** ✅ Complete (Parts 1–3). Part 3 doubled as the read-only exposure-audit verification pass for §3 of `PROJECT-STATE`.

The layer that ties the whole box together: everything on this server is ultimately *things talking to other things over a network*. Goal of this chapter: stop treating `192.168.x` / `172.x` / `10.8.x` as noise and understand each as a distinct "where," how a request actually travels between them, and — finally — what `172.19.0.2` has been the whole time.

---

## TL;DR

- An IP address is a **location** ("where"), and one machine has **many** of them because it's on several networks at once. This box has **16 IPs**.
- Three private ranges (RFC 1918) are non-routable on the internet, so every home reuses them: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. **All three appear on this box** — LAN, Docker, VPN.
- **CIDR** `/24` = 24 network bits + 8 host bits ≈ 254 usable addresses. Rule: *slash counts network bits; 32 − slash = host bits; 2^host bits = addresses.*
- **NAT is one concept at three scales** — the router, Docker, and the wg-easy VPN container each rewrite private addresses to cross a border. This single idea resolves the long-running `172.19.0.2` mystery.
- Finding: **~15 of 16 default Docker `/16` pools are in use** → address-pool exhaustion risk. (And a tempting-but-wrong "prune orphaned networks" was correctly rejected — see §3.)
- **Exposure audit passed:** `ss -tulnp` confirmed the databases (5432/3306/6379) are **not** host-exposed — verifying `PROJECT-STATE`'s security claim against reality. Everything else is published on `0.0.0.0` (LAN-reachable) via `docker-proxy`; only 4 ports are internet-forwarded.

---

## Part 1 — Addresses, subnets & the LAN

### An IP is a "where," and the box has many

IPv4 = a **32-bit** number written as four **octets** (0–255): `192.168.0.215`. The beginner trap is assuming a machine *has an IP* (singular). It has many, because it's plugged into several networks simultaneously — each a separate "where."

### Public vs private — why every home reuses the same addresses

`192.168.0.215` is **not globally unique** — millions of homes have it. Three ranges are reserved as **private** (RFC 1918) and **non-routable on the public internet** (routers drop packets aimed at them):

| Range | Common use |
|---|---|
| `10.0.0.0/8` | large private nets, VPNs |
| `172.16.0.0/12` | `172.16`–`172.31` — **Docker's default pool** |
| `192.168.0.0/16` | classic home LAN |

Because they never leave the local network, every home can reuse them without colliding. This is also the seed of NAT (Part 2).

### CIDR & subnets — decoding `/24`

The slash answers: *of the 32 bits, how many are the network part vs the host part?*
- `/24` → first 24 bits = network (`192.168.0`), last 8 = host → **256 values, ~254 usable** (`.1`–`.254`; `.0` = network, `.255` = broadcast). Same as netmask `255.255.255.0`.
- `/16` → 16 host bits → ~65,000 hosts. **Smaller slash = bigger network.**
- **The rule:** `32 − slash = host bits`; `2^host bits = addresses`. *(Corrected a misconception here: `/24` gives 256 addresses, not 8 — "8" is the number of host *bits*, not addresses.)*

The subnet defines **"who is local to me."** For every packet, the box asks *is the destination in one of my subnets?* → deliver directly; else → hand to the gateway. That per-packet decision is **routing**.

### LAN vs WAN — the router is the border post

- **LAN** (inside): `192.168.0.0/24` — laptop, phone, the Dell (`.215`). Router's inside face = **default gateway `192.168.0.1`**.
- **WAN** (outside/internet): one ISP-given **public IP `80.95.28.55`** represents the *whole house*.

The router has a foot in each world: a private LAN address inside, one public address outside.

### The three private worlds on this one box

| Network | Address | Purpose |
|---|---|---|
| Home LAN | `enp0s31f6` = `192.168.0.215/24` | reaches the router, LAN, internet |
| Docker internals | `172.x` (many `/16`s) | containers talking to each other **inside the host** |
| WireGuard VPN | `10.8.0.0/24` | remote clients' tunnel addresses |

One machine, three "wheres," because it's on three networks. **This is why `172.19.0.2` looked foreign** — it's a Docker-internal address, not a LAN address.

### Routing decision (corrected)

For `192.168.0.50`: the box checks its own routing table, sees it's inside a **directly-connected** network (`192.168.0.0/24 ... scope link`), and delivers it **directly on the LAN — the router is NOT involved.** For `80.95.28.55`: not local → **default route** → hand to gateway `192.168.0.1`. *The decision happens on the box, before the router ever sees a local packet.*

### Verified live

- `ip -br addr`: `lo` (`127.0.0.1/8`), `enp0s31f6` = **`192.168.0.215/24`** ✓ (matches `PROJECT-STATE` — no drift), `docker0` (`172.17.0.1/16`, DOWN), ~15 `br-*` bridges (`172.17`–`172.31`), ~20 `veth*` pairs.
- `hostname -I`: **16 addresses** — `192.168.0.215` + fifteen `172.x.0.1`. Each `172.x.0.1` is **the box acting as the gateway (`.1`) for that Docker network.**
- `ip route`: `default via 192.168.0.1 dev enp0s31f6`; one directly-connected route per subnet; several marked `linkdown`.
- **Reading the interfaces:** `br-*` = Docker networks (virtual switches); `veth*@if2` = virtual cables into containers (`master br-*` says which network). Located the ghost: `br-ac1783304957` = `172.19.0.1/16` = `wireguard_default`; the wg-easy container sits on it at `172.19.0.2`.

### Finding: Docker address-pool pressure + a corrected "orphan" call

`ip -br addr` showed several bridges **DOWN** (`172.18/28/29/30`), and `docker network ls --filter dangling=true` flagged `lidarr_default`, `npm_default`, `prowlarr_default`, `qbittorrent_default`. Initial (wrong) read: "orphaned, prune them." **Domain knowledge corrected it:** those services *are* running — they just run on a **shared** network (arrstack `172.31.x`, or the website network `172.26.x`) instead of their own auto-created `<project>_default`, leaving the `_default` bridge empty (DOWN). Those `_default` networks are **recreated on every `compose up`** — not true orphans. **Do not prune** (cosmetic + regrows).

The *real* issue is capacity: **~15 of 16 default `172.x/16` blocks are in use**, so adding more multi-network stacks (Vaultwarden, Jellyfin…) risks a "no available address pool" error. The lever is **network consolidation**, not pruning.

> **Key lesson:** most services run isolated on their own single-container network; only `arrstack` (4 services) and the website network are shared. That's a reasonable *isolation-by-default* security posture — with an address-pool cost worth watching.

---

## Part 2 — Ports & NAT

### The half of an address that was missing: the port

An IP finds the *machine*; a **port** (16-bit, 0–65535) identifies *which program* on it. The real coordinate is an **IP : port** pair (a **socket**): `192.168.0.215:22` = SSH, `:2283` = Immich. Ports **< 1024 are privileged** (need root to bind) — which is *why* containers publishing 80/443 (NPM) involve root and higher ones (3001, 8090) don't. A TCP connection is identified by the **four-tuple** (src IP, src port, dst IP, dst port) — the thing that makes NAT possible.

### NAT — one public IP for a whole house

The house has one public IP but many devices online at once, each with a non-routable private address. **NAT (Network Address Translation)** on the router rewrites packets at the border:

1. Laptop → source `192.168.0.50:54321`, dest `93.184.x.x:443`.
2. Router rewrites **source** to `80.95.28.55:<tracking-port>` and records the swap in its **NAT table** (a ledger).
3. Reply comes back to `80.95.28.55:<tracking-port>`.
4. Router looks up the ledger, rewrites **dest** back to `192.168.0.50:54321`, delivers.

The **port** is what keeps many simultaneous conversations distinct. (This flavour = PAT / "NAT overload" / **masquerading**.)

### Why outbound "just works" but inbound needs port-forwarding

Outbound creates a ledger entry *because the inside started it*. An **unsolicited inbound** packet (someone loading the website) hits the router with **no matching ledger entry** → the router doesn't know which inside device to send it to → **drops it by default.** So each **port-forward rule** ("unsolicited `:443` → `192.168.0.215`") is a deliberate hole punched inward. Forwards in use: **443, 80, 51820, 2222.**

> **Key lesson:** NAT is an *accidental firewall* — inbound is denied unless explicitly forwarded. **Every port-forward is attack surface you chose to open.**

### Docker repeats NAT *inside* the host

Docker does to containers what the router does to the house. Each container has a non-routable `172.x` address; publishing a port (`-p 2283:2283`) creates a NAT rule + a **`docker-proxy`** helper that forwards `host:2283 → container 172.21.0.4:2283`. **That entire `docker-proxy` wall (seen in `docker.service`'s cgroup) is Docker's NAT layer** — one helper per published port.

So the box has **two stacked NAT layers**: router (LAN ↔ internet) and Docker (host ↔ containers). An inbound web request crosses **both**.

### The full inbound journey (website)

1. DNS resolves `sim-obleklo.bg` → Cloudflare (hides `80.95.28.55`).
2. Cloudflare → **`80.95.28.55:443`** hits the router (WAN).
3. **Port-forward** 443 → `192.168.0.215` *(NAT layer 1 crossed)*.
4. Host `:443` → **`docker-proxy`** → **NPM container `172.26.0.5:443`** *(NAT layer 2 crossed)*.
5. **NPM** reads the `Host:` header, matches its rule, forwards to the website frontend.
6. frontend → backend → Postgres → response retraces the path out.

### RESOLVED: why remote SSH shows `FROM 172.19.0.2`

Remote SSH travels: laptop → **WireGuard tunnel** → **wg-easy container (`172.19.0.2`)**, which **decrypts** it and forwards to the host's `sshd` — **but masquerades (source-NATs) it, rewriting the source to its own address `172.19.0.2`.** So `sshd` sees the connection arriving `FROM 172.19.0.2` — the container's address, not the laptop's. **That is exactly the `who` / `ss` output from Chapter 1.** Same masquerading mechanism as the router and Docker — just performed by the VPN container.

**Consequence (logged since Ch 1):** because *all* VPN clients are masqueraded behind `172.19.0.2`, the host can't tell them apart by source IP. Not a quirk — NAT doing what NAT does.

> **Key lesson — the whole chapter in one line:** **NAT is a single concept at three scales.** Router (house ↔ internet), Docker (host ↔ containers), wg-easy (VPN client ↔ host). Each is a border that rewrites private addresses to cross into where they aren't valid. `172.19.0.2` was never "you" — it's the return address the VPN stamped on your traffic.

---

## Part 3 — Ports, sockets & "who's listening" (the exposure audit)

### A listening socket = a service with its door open

A **listening socket** is a *(protocol, IP, port)* a program has claimed and is waiting on. "Is my service reachable?" reduces to: **is something listening on the right IP:port, and can the client's packets get there?** Most "down"/"refused" errors (e.g. `boot-alert`'s `curl (7)`, Immich `ECONNREFUSED` at boot) are simply *nothing is listening at that socket yet*. Reading listening sockets diagnoses half of all homelab problems.

### The security lever: `0.0.0.0` vs `127.0.0.1`

When a program binds a port it also picks *which interface*, which decides **who can reach it**:

| Bind address | Reachable by |
|---|---|
| `127.0.0.1` (loopback) | **only other processes on this host** — never the LAN or internet |
| `0.0.0.0` (all interfaces) | anyone who can route a packet to *any* of the box's addresses (LAN, Docker, forwarded internet) |
| specific IP (e.g. `192.168.0.215`) | only that one interface |

The difference between a DB on `127.0.0.1:5432` and `0.0.0.0:5432` is "only the local app can reach it" vs "it's listening on the LAN." Same service, wildly different exposure — decided by one address.

### TCP vs UDP

**TCP** = connection-oriented, reliable, ordered (handshake + retransmits) → websites, SSH, most services. **UDP** = connectionless, fire-and-forget, low-overhead → **DNS** (AdGuard :53) and **WireGuard** (:51820). *(Correction from quiz: TCP is more* reliable*, not more* secure *— encryption is a separate layer, TLS/WireGuard.)* This is why the firewall rule is `51820/udp` specifically.

### The tool: `ss`

`ss` ("socket statistics"; `netstat`'s modern replacement). `sudo ss -tulnp` = **t**cp + **u**dp, **l**istening only, **n**umeric, **p**rocess. The single best "what's exposed?" command. *Reading subtlety that already bit us:* `ss` matches text, so `grep :22` also catches `:2283` — filter with a trailing space or read the full list.

### Host `ss` shows *published* ports, not container listeners

A published port (`-p`) appears in the host's `ss` as **`docker-proxy`** holding the door — the *real* service listens inside its container's namespace (e.g. Immich on `172.21.0.4:2283`, while the host shows `docker-proxy` on `0.0.0.0:2283`). A merely **`expose`d** container port (internal-only) does **not** appear as a host listener at all — which is the desired locked-down state for internal services.

### Verified live — the audit result

**✅ Database claim CONFIRMED (the headline).** `ss -tulnp | grep -E '5432|3306|6379'` returned **nothing** — Postgres, MariaDB, and Redis are **not** host-exposed; they bind container-internal only, exactly as `PROJECT-STATE` claimed. Doc said it, reality confirmed it → upgraded from `[per docs]` to **[verified June 28]**.

**The listener inventory:**
- Nearly every published port is owned by **`docker-proxy`** on `0.0.0.0` (and `[::]` for IPv6) — the NAT front-desk, not the real services.
- **`sshd` on `0.0.0.0:22`** — real SSH daemon, directly on the host (not containerized). Correct.
- **`AdGuardHome` on `*:53` and `*:3000`** — bound directly, no `docker-proxy`: the visible fingerprint of **`network_mode: host`** (skips Docker NAT). Concrete proof of "host networking looks different."
- **`chronyd` on `127.0.0.1:323`** — the *only* loopback-only service (NTP time sync). Correctly scoped host-internal; textbook example.

**Posture note (honest):** every `docker-proxy` line is `0.0.0.0` = **LAN-reachable**, including admin UIs (Portainer `9000`, NPM admin `81`, qBittorrent `8090`, Prowlarr `9696`, Lidarr `8686`, Immich `2283`). Only **4 ports are internet-forwarded** (`80/443/51820/2222`) — the rest are LAN-only, so not internet-exposed. But *anything on the LAN* (a guest, a compromised IoT device) can reach every admin panel directly. This is precisely why **UFW** matters: it can scope these `0.0.0.0`-published ports down to "VPN only" *after* Docker publishes them. **Docker publishes to `0.0.0.0`; UFW claws it back** — a real interaction (with a known "Docker bypasses UFW" gotcha, hinted at by the `WG_IPTABLES_BACKEND=nftables` note) to dig into in the firewall chapter.

> **Key lesson:** the bind address is a security control. Audit exposure with `ss -tulnp`, not the compose files — verify what's *actually* listening, on *which* interface. "Absent from host `ss`" = "not directly reachable" = the correct state for internal services behind a reverse proxy.

---

## Diagrams produced (for reference / portfolio)

1. **NAT border** — three LAN devices sharing one public IP through the router's NAT ledger.
2. **The nesting** — home (LAN) › the Dell (host) › Docker network › container, each with its own address scope. Answers "which is which / what lives inside what."
3. **Interactive journey stepper** — the 5 hops of a remote SSH login, culminating in step 4 where wg-easy rewrites the source to `172.19.0.2`.

---

## State to promote into `PROJECT-STATE` §3 / `networking.md`

- **NIC `enp0s31f6` = `192.168.0.215/24`, gateway `192.168.0.1`** [verified June 28] — matches docs, no drift.
- **Docker networking model:** most services isolated on their own `<project>_default` bridge; only `arrstack` (4 svcs, `172.31.x`) and the website network (`172.26.x`) are shared. `dangling`/DOWN `_default` nets (lidarr/prowlarr/qbittorrent/npm) are **auto-recreated on `compose up`, not true orphans** — do not prune. **~15 of 16 default `172.x/16` pools in use → watch for pool exhaustion; consolidate rather than prune.**
- **Exposure audit [verified June 28] (`ss -tulnp`):** databases (5432/3306/6379) confirmed **not** host-exposed — container-internal only ✓ (upgrade this line from `[per docs]` to `[verified]`). `sshd` and AdGuard (`network_mode: host`) bind the host directly; all other services published via `docker-proxy` on `0.0.0.0` (LAN-reachable). Only 4 ports internet-forwarded (80/443/51820/2222); rest LAN-only. `chronyd` correctly loopback-only. **Website containers use `expose` (not `publish`) → absent from host listeners → reachable only via NPM over the Docker network** (corrects the "port-forwarding" assumption — it's publish-vs-expose, not a router forward).

## Open items

- [ ] Docker `/16` pool pressure — plan network consolidation before adding Vaultwarden / Jellyfin / etc.
- [ ] **Part 3 (next):** `ss` / `ip` reading skills; the `0.0.0.0` (all interfaces) vs `127.0.0.1` (localhost-only) **binding distinction** as a security lever; audit which services are exposed where. Doubles as the §3 verification pass for `PROJECT-STATE`.

---

## Portfolio / learning angles

- **"Why do my SSH sessions look like they come from a container?"** Traced a remote login through WireGuard → wg-easy → sshd and found the VPN container **masquerades** the source to `172.19.0.2`. Demonstrates reading `who`/`ss`, understanding source-NAT, and recognising NAT at three scales.
- **Read the whole box's network identity from `ip addr` / `ip route`** — 16 IPs, distinguishing bridges (`br-*`) from container cables (`veth*`), and mapping each to a Docker network.
- **Caught Docker address-pool pressure** (15/16 `/16` blocks) — and *correctly rejected a tempting prune*, overruling both a tool's `dangling` label **and** the teacher's recommendation using ground truth ("I use those services"). The reflex to verify a claim against what you actually know is the transferable skill.
- **Isolation-by-default networking posture** framed as a deliberate security choice with a capacity tradeoff.
- **Exposure audit that verified a security claim.** Ran `ss -tulnp` and *confirmed against reality* that the databases weren't host-exposed, that internal services are reachable only through the reverse proxy (not published to the host), and that only 4 ports are internet-forwarded. Turning a documented claim into a verified one is exactly the doc-discipline story.

---

## Appendix — read-only commands used (blast radius: zero)

```bash
# Addresses & routing
ip -br addr                 # brief: one line per interface
ip addr                     # full detail (long — br-* = Docker nets, veth* = container cables)
ip route                    # routing table: default gateway + directly-connected nets
hostname -I                 # all the box's IPs on one line

# Docker networking
docker network ls                          # all networks
docker network ls --filter dangling=true   # Docker's view of "unused" networks (read with care!)
docker inspect <ctr> --format '{{range .Mounts}}...'  # (mounts/subnets per container)

# Ports & exposure audit
sudo ss -tulnp                             # every TCP+UDP listener, numeric, with process
sudo ss -tulnp | grep -E '0\.0\.0\.0|\[::\]'  # bound to all interfaces (off-box reachable)
sudo ss -tulnp | grep '127.0.0.1'          # loopback-only (host-internal, safe)
sudo ss -tulnp | grep -E '5432|3306|6379'  # DB exposure check (expect: nothing)
```

---

*Chapter 3 complete. Next: Part II of the curriculum breaks into its own chapters — DNS (Ch 4), Remote Access / WireGuard + SSH (Ch 5), and the Firewall / UFW (Ch 6), where the "Docker publishes to 0.0.0.0, UFW claws it back" interaction gets its proper treatment.*
