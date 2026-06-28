# Study Session — Chapter 1: The Machine (Hardware, Firmware & Headless Operation)

> **Repo:** `homelab` (infrastructure — not the website repo)
> **Date:** June 27, 2026
> **Format:** Study-plan deep dive — explain the *why*, verify against the live box, quiz.
> **Verified against:** live `homelab` server over WireGuard + SSH, June 27, 2026.

The first chapter of a bottom-up study plan to understand the OptiPlex from bare metal up. Goal of this chapter: stop *operating* the hardware by rote and start being able to explain **why every piece is there and how one hardware fact ripples up into software behaviour.**

---

## TL;DR

- A "server" isn't special hardware — it's an ordinary machine doing the job of *staying on and being reached over the network*. The single defining fact of this box is that it is **headless** (no monitor/keyboard), and almost the entire networking stack exists as a downstream consequence of that.
- Verified the documented hardware against reality — **no drift found** this time.
- Biggest lesson came from an unexpected place: chasing a phantom **"2 users"** in `uptime` turned into a clean *"don't trust the ledger, count the wires"* investigation. The OS's own bookkeeping file (`utmp`) had drifted from reality — exactly the way markdown docs drift.

---

## 1. Verified hardware state — [verified June 27]

| Component | Detail | Notes |
|---|---|---|
| Machine | Dell OptiPlex 7040 SFF | `Chassis: desktop` — ex-office small-form-factor; ideal homelab box (low idle power, quiet, cheap secondhand). |
| CPU | Intel Core i5-6500 @ 3.20 GHz | **4 cores / 4 threads** confirmed (`Thread(s) per core: 1`) — **no** Hyper-Threading. Turbo to 3.6 GHz, idle 0.8 GHz. |
| GPU | None (integrated only) | No discrete GPU — directly explains slow Immich ML (see §3). |
| RAM | 16 GB DDR4 (15 GiB visible) | ~1 GiB shaved by integrated-graphics carve-out + kernel reservation. **`available` 11 GiB** — healthy. |
| Swap | 4 GiB | **0 B used** — healthy; never under enough pressure to spill to disk. |
| Virtualization | VT-x present, **unused** | Box runs *containers* (shared kernel), not VMs. VT-x just means VMs are possible if ever wanted. |
| Boot/OS disk | `sdb` ~120 GB SSD | 3 partitions: `sdb1` 1G `/boot/efi`, `sdb2` 2G `/boot`, `sdb3` 116G **LVM** → `ubuntu--vg/ubuntu--lv` → `/`. |
| Data disk | `sda` ~112 GB SSD → `/mnt/ssd2` | Single plain partition, **no LVM** — added by hand, formatted, mounted. Holds `music/` + `downloads/`. |
| Optical | `sr0` | CD/DVD drive, empty/unused — leftover from desktop life. |
| Firmware | Dell BIOS **1.24.0**, dated 2022-07-14 | ~4 years old. See §5. |
| OS / kernel | Ubuntu 26.04 LTS / `7.0.0-22-generic` | Pending kernel update takes effect only on reboot (Ch 17). |
| Uptime / load | 14 days / load ~0.12 on 4 cores | Saturation line ≈ 4.0; 0.12 = the box is loafing → "21 mostly-sleeping services" proven. |

**Verify-reality result:** docs matched the machine on every line checked. Nothing to correct. (Recording the *non-finding* matters too — it's evidence the source-of-truth is currently trustworthy.)

---

## 2. The throughline — headless → safe remote access

The one idea to carry into Part II:

> **Nobody sits in front of this machine.** If you can't sit in front of it, you must reach it over the network. That single fact *forces* SSH, *forces* a VPN to make SSH reachable safely from anywhere, and *forces* a firewall to expose only what's safe.

So none of the upcoming networking chapters (DNS, WireGuard, SSH, UFW) are arbitrary — they are all the consequence of the headless decision made at the hardware layer. **Hardware choice in Ch 1 → entire security posture in Part II.**

---

## 3. Concept notes that bit us (the *why* behind the *what*)

**`free` vs `available` — the RAM trap.** Linux borrows otherwise-idle RAM as a file cache (`buff/cache`), so naïve "used" looks low and "free" looks alarming. The number that matters is **`available`**: what the kernel can hand a new program *right now* by reclaiming cache. Watch `available`, not `free`.

**No GPU → slow Immich ML (the cross-layer chain).** ML inference is millions of tiny matrix multiply-and-adds. A CPU has 4 generalist lanes and does them largely in sequence; a GPU has thousands of dumb lanes doing them at once — exactly what the math wants. **No GPU → ML falls back to CPU → photo indexing is seconds-per-image not ms → whole-library indexing is slow.** One hardware absence surfaces three layers up as a service behaviour.

**Containers, not VMs.** VT-x is present but unused because Docker containers *share* the host's single Ubuntu kernel and isolate only the process — far lighter than a VM emulating a whole second computer. That lightness is *why ~21 of them fit on this box*. (Full treatment Ch 7.)

**LVM-on-root vs plain second disk.** The Ubuntu installer wraps `/` in LVM (a flexibility layer for resize/snapshot later); the hand-added second SSD is just a plain formatted partition. The OS disk got the fancy treatment; the bolt-on disk didn't. (Storage detail Ch 16.)

---

## 4. Centrepiece — the "2 users" investigation: `utmp` vs reality

**The symptom.** `uptime` reported **2 users**, but the tools disagreed with each other:

- `uptime` → "2 users"
- `w` → header "2 users" but body listed only **1** session
- `who` → listed **0**

Three tools, three stories, while clearly logged in. When a record disagrees with itself, the record is wrong — not reality.

**The cause.** `uptime`/`who`/`w` don't *measure* logins; they read a bookkeeping ledger called **`utmp`** (`/run/utmp`). A clean logout removes its entry; an **unclean** death (sshd killed, or a **WireGuard tunnel blip dropping a session mid-flight**) can leave a "ghost" entry the ledger never erases. On Ubuntu 26.04, `utmp` is also mid-deprecation, so the legacy tools read it inconsistently — hence `who` saying 0 while `uptime` says 2. (Honest gap: I don't know the exact internal reason `who` returned 0 on this build, and didn't invent one.)

**Ground truth — tools that don't trust the ledger:**

```bash
ps aux | grep '[s]shd-session'   # one process per LIVE ssh session (note: sshd-session, see below)
loginctl list-sessions           # systemd-logind's own tracker, separate from utmp
sudo ss -tnp | grep ':22'        # actual established TCP connections on the SSH port
```

**The verdict — exactly one live session.**
- `ss` showed a single established connection: `192.168.0.215:22 ← 172.19.0.2:56994`, held by `sshd-session`. One wire = one human.
- `loginctl` listed 3, but the **CLASS** column reconciled it: one was `manager` (the systemd *user manager*, infrastructure, not a login), and of the two `user`-class sessions only one (leader PID matching the live `ss` connection) was real.
- The ghost was **session 2068**, `State: closing`. `session-status` revealed why it never cleared: its SSH login disconnected **cleanly a week earlier** (Jun 19), but a leftover **`.vscode-server ... agent host` process** stayed alive inside the session scope, keeping it half-closed. Its leader PID returned nothing from `ps -p` → **dead**. → A zombie VS Code remote-server, orphaned when the tunnel dropped. Cosmetic; clears on reboot or via `loginctl terminate-session 2068`.

**NAT footnote (connects to a prior lesson).** The live session's source showed as `FROM 172.19.0.2` — the **wg-easy container**, not the laptop. The container **masquerades (NATs)** VPN traffic, rewriting the source to its own address. Consequence: **VPN clients can't be told apart by source IP at the host level** — they all look like `172.19.0.2`. Same NAT phenomenon previously diagnosed for DNS; here it reappears for SSH.

**Teacher's-own-stale-command beat (kept on purpose).** The first grep pattern tried (`sshd:`) returned nothing, because modern OpenSSH split the daemon into per-connection **`sshd-session`** processes and dropped the old `sshd: user@pts/N` label. The instruction was built on older behaviour — a stale assumption the live box corrected. The course's whole thesis, demonstrated against the course itself.

> **Key lesson:** Count the wires, not the ledger. `ss`/`ps`/`loginctl` are load-bearing truth; `who`/`w`/`uptime` are convenience notes that drift exactly like documentation. The day `who` shows a session from an IP you don't recognise, confirm against `ss` before panicking — or before *not* panicking.

---

## 5. Parked items — require physical presence (blast-radius notes)

These are deliberately **not** to be done remotely. Recording them so they're not forgotten on the next physical visit to Gabrovo.

| Item | Why parked | Blast radius if done wrong remotely |
|---|---|---|
| **AC Power Recovery** (BIOS) — confirm set to power-on after power loss | A 30-sec outage otherwise leaves the live site down until someone presses the button | Can't be set remotely at all (BIOS screen needs keyboard+monitor). Failure mode = indefinite downtime. |
| **BIOS 1.24.0 → newer** (optional, low priority) | Carries CPU microcode + fixes | **Highest blast radius in the whole stack:** a blip mid-flash can brick the board; recovery needs physical presence. **Never remote.** |
| **GDS / "No microcode" flag** | `lscpu` shows `Gather data sampling: Vulnerable: No microcode` | Low priority for a single-user box behind VPN+firewall. **Reversible Linux path exists:** `intel-microcode` package loads microcode at boot independent of BIOS — run in `tmux`, reboot, re-check `lscpu`. Far safer than a flash. (Ch 17.) |

---

## 6. Open to-dos — `homelab` repo

- [ ] **Next physical visit:** verify BIOS *AC Power Recovery* enabled.
- [ ] (Optional, Ch 17) Try `intel-microcode` to clear the GDS flag — reversible, run in `tmux`, then `lscpu` to confirm.
- [ ] (Optional, cosmetic) Clear the zombie session: `loginctl terminate-session 2068` — or just let the next reboot do it.
- [ ] Note for Ch 17: pending kernel `7.0.0-22` only applies on reboot — "fully patched" and "14-day uptime" are in mild tension.
- [ ] BIOS flash → only ever at the physical machine; low priority.

---

## 7. Portfolio / learning angles

- **"utmp vs reality: counting login sessions the honest way."** A tidy verify-reality story in the same vein as the NAT-diagnosis nugget: three OS tools disagreed, traced it to a drifting bookkeeping file, and proved the real count with `ss`/`ps`/`loginctl` — then identified the ghost as an orphaned VS Code server still propping a week-old session open. Demonstrates debugging discipline, knowing where ground truth lives, and not trusting a record over the wires.
- **NAT/masquerade recognised across two services.** Spotting that SSH (like DNS before it) arrives as the wg-easy container IP shows pattern transfer, not one-off luck.
- **Blast-radius discipline.** Explicitly parking firmware/BIOS work for physical presence, and preferring the reversible `intel-microcode` path over a risky remote flash, is exactly the change-management judgement that separates a homelabber from someone who bricks a board 1,000 km away.

---

## Appendix — read-only commands used (blast radius: zero)

```bash
hostnamectl                       # OS, kernel, vendor/model, firmware, virtualization
lscpu                             # cores/threads, model, caches, vuln flags
free -h                           # RAM — watch the 'available' column
lsblk                             # block devices, partitions, mount points
sudo dmidecode -t bios            # BIOS version/date (reads firmware table)
uptime                            # uptime, session count (utmp), load average
who ; w                           # session ledgers (utmp) — convenience, can drift
ps aux | grep '[s]shd-session'    # live ssh sessions (ground truth)
loginctl list-sessions            # logind session tracker
loginctl session-status <id>      # a session's leader PID, service, scope tree
ps -p <pid>                       # is a given PID alive? (empty output = dead)
sudo ss -tnp | grep ':22'         # established TCP connections on SSH port (ground truth)
```

---

*Next: Chapter 2 — The Operating System (Ubuntu Server): the boot chain, systemd, the filesystem & LVM, users/permissions, and apt.*
