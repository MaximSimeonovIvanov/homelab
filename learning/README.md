# 📚 Learning — Understanding the Box, Layer by Layer

> Bottom-up study notes that explain **why** each layer of this server exists and **how it was proven against the live machine** — not just how to operate it.

This folder is the *understanding* companion to the rest of the repo. Where [`docs/`](../docs/) answers **"what is the current state and how do I reproduce it?"**, the notes here answer **"why does this work the way it does, and how did I verify it?"** They are written as a self-taught, bottom-up course through the OptiPlex 7040 homelab — from bare metal up to the running application.

Each note follows the same method:

1. **Explain the *why*** — the reasoning, tradeoffs, and failure modes behind a layer, not just commands.
2. **Verify against reality** — read-only checks run on the live server, because documentation drifts. Findings are recorded even when they *contradict the docs* (that's the point).
3. **Quiz** — close the loop by reasoning it back in plain language.

> **Guiding principle:** don't trust documentation over reality. When it matters, verify the actual state of the server with read-only checks before recording anything as "true." More than once the *machine* — or even the OS's own bookkeeping — has corrected the notes.

---

## Curriculum

Two mental models run through every chapter: the **layer cake** (each layer stands on the one below — the spine of this index) and the **request's journey** (follow one packet from a customer's browser down through the stack and back — the connective tissue).

### Part I — Foundations
| Ch | Title | Status |
|---|---|---|
| 01 | [The Machine: Hardware, Firmware & Headless Operation](./01-the-machine.md) | ✅ |
| 02 | The Operating System: Ubuntu Server (boot chain, systemd, filesystem & LVM, users, apt) | ⏳ planned |

### Part II — Reaching the Box
| Ch | Title | Status |
|---|---|---|
| 03 | Networking Fundamentals (IP, subnets, ports, LAN/WAN, NAT) | ⏳ planned |
| 04 | DNS: How Names Become Addresses (AdGuard, Cloudflare, `.home`) | ⏳ planned |
| 05 | Remote Access: WireGuard + SSH (tunnels, keys, split-tunnel, VS Code Remote) | ⏳ planned |
| 06 | The Host Firewall: UFW & Fail2ban (deny-by-default, VPN-only scoping) | ⏳ planned |

### Part III — Running Software
| Ch | Title | Status |
|---|---|---|
| 07 | Containers & Docker (images vs containers, the daemon, namespaces & cgroups) | ⏳ planned |
| 08 | Docker Compose & Inter-Container Networking (bridge networks, volumes, the arr-stack) | ⏳ planned |
| 09 | The Reverse Proxy: Nginx Proxy Manager (host headers, one door in) | ⏳ planned |
| 10 | TLS/SSL & Cloudflare (certificates, DNS-01 vs HTTP-01, CDN/cache, IP hiding) | ⏳ planned |

### Part IV — The Services
| Ch | Title | Status |
|---|---|---|
| 11 | Tour of the Self-Hosted Stack (Nextcloud, Immich, Portainer, Homarr, Ntfy, Uptime Kuma) | ⏳ planned |
| 12 | The Media Pipeline as a System (network coupling vs shared-filesystem coupling) | ⏳ planned |

### Part V — The Application
| Ch | Title | Status |
|---|---|---|
| 13 | The Website: Django + Next.js + Postgres | ⏳ planned |
| 14 | CI/CD & Deployment (GitHub Actions, deploy key on 2222, the full request path) | ⏳ planned |

### Part VI — Keeping It Alive
| Ch | Title | Status |
|---|---|---|
| 15 | Observability & Alerting (Uptime Kuma, Ntfy, cron alert scripts, reading logs) | ⏳ planned |
| 16 | Storage, Backups & Data Safety (disk layout, the planned HDD, 3-2-1) | ⏳ planned |
| 17 | Maintenance & Change Management on a Live Box (apt, image hygiene, `tmux`, blast radius) | ⏳ planned |

### Part VII — The Whole Picture
| Ch | Title | Status |
|---|---|---|
| 18 | Security as Defense-in-Depth (revisits Ch 5, 6, 10 as one threat model) | ⏳ planned |
| 19 | Documentation & Reproducibility (two-repo discipline, source-of-truth vs history, docs drift) | ⏳ planned |

---

## How this folder relates to the rest of the repo

| Folder / file | Genre | Answers |
|---|---|---|
| [`docs/`](../docs/) | Reference | *What is the current state, and how do I reproduce it?* |
| [`compose/`](../compose/) | Configuration | *What exactly is running, declaratively?* |
| **`learning/`** (here) | Understanding | *Why does it work this way, and how was it proven?* |

When a learning note surfaces a fact that belongs in the canonical reference (e.g. a hardware detail or a new known limitation), it is **promoted** into `docs/` rather than left only here — so `docs/` stays the single source of reproducible truth and `learning/` stays the place for the reasoning behind it.

---

*A self-hosted homelab on a Dell OptiPlex 7040 running Ubuntu Server 26.04 LTS — built and understood by doing.*
