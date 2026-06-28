# Study Session — Chapter 2: The Operating System (Ubuntu Server)

> **Repo:** `homelab` (infrastructure — not the website repo)
> **Date:** June 28, 2026
> **Format:** Study-plan deep dive — explain the *why*, verify against the live box, quiz.
> **Verified against:** live `homelab` server over WireGuard + SSH, June 28, 2026.

The OS layer of the bottom-up study plan. Walks the machine from cold metal to running services: the boot chain, systemd, the filesystem & LVM, and users/permissions/apt. Goal: understand *why* each part of the OS exists and how one layer hands off to the next — and along the way, **fix two real bugs** and **activate a staged kernel** the safe way.

---

## TL;DR

- An OS is the **kernel** (the one piece allowed to touch hardware; everything else asks it via system calls) plus a distribution's userland. "Linux" = kernel; "Ubuntu" = distribution.
- The **boot chain** is a series of handoffs — UEFI → GRUB → kernel+initramfs → **systemd (PID 1)** — each stage existing only to load the next. The disk layout (`/boot` plain, `/` on LVM) is a direct fingerprint of this order.
- **systemd** models everything as a *unit* and never leaves — it supervises every service for the life of the box. Met all five unit types in the wild and fixed a real boot-ordering bug with a drop-in override.
- **Activated a staged kernel** (`7.0.0-22` → `7.0.0-27`) via an informed, pre-flighted reboot, then verified full recovery.
- Biggest *finding*: the LVM volume group is **fully allocated** (`VFree 0`) — which rewrites the future HDD-migration plan.

---

## 1. What the OS is — the kernel as referee

The **kernel** is the only software allowed to touch hardware directly. Everything else (shell, Docker, all ~21 containers) lives in **userspace** and must ask the kernel via a **system call** to read a file or send a packet. Two reasons it's built this way: **isolation** (a buggy program can't corrupt another's memory or the disk) and **sharing** (the kernel time-slices 4 CPU cores across dozens of processes — that's what "load average" measures).

**Key distinction:** `hostnamectl` prints `Operating System: Ubuntu` and `Kernel: Linux` on separate lines because they *are* separate. **Linux is only the kernel.** **Ubuntu is a distribution** = the kernel + GNU userland + systemd + apt + defaults. The kernel is the one shared ingredient under every distro.

**Forward hook:** every container shares this *one* kernel — which is exactly why containers are lighter than VMs (a VM boots its own kernel). [Ch 7.]

---

## 2. The boot chain — four handoffs from cold metal to a live site

Each stage exists only because the previous one couldn't do the next job (a chain of chicken-and-egg problems):

1. **Firmware (UEFI)** — Dell BIOS runs POST, then looks for the **EFI System Partition** (`sdb1`, the 1 GB `/boot/efi`, vfat — the only filesystem UEFI reads). Loads the bootloader, steps aside.
2. **Bootloader (GRUB)** — finds the **kernel** + **initramfs** on `/boot` (`sdb2`), loads both into RAM, passes the kernel its command line, jumps in.
3. **Kernel + initramfs** — the kernel can't see root yet, because **root is on LVM** and the LVM driver isn't loaded. Chicken-and-egg: needs root mounted to load drivers, needs a driver to mount root. The **initramfs** (a tiny RAM filesystem GRUB pre-loaded) carries the LVM/device-mapper driver, assembles the volume group, mounts the real `/`, pivots onto it, and is discarded.
4. **init = systemd (PID 1)** — the first userspace process. Mounts the rest of the filesystems (per `/etc/fstab` — how `/mnt/ssd2` returns), starts networking, starts the **Docker daemon**, which starts the containers. Site live.

**Why `/boot` is a plain partition but `/` is LVM:** the boot essentials must be readable *before LVM exists*, so they live on plain partitions early tools can read. LVM only comes alive *after* the initramfs sets it up. **The layout is the bootstrap order, frozen on disk.**

**Verified live:**
- `cat /proc/cmdline` → `root=/dev/mapper/ubuntu--vg-ubuntu--lv` — the literal instruction GRUB handed the kernel. (`/proc` is a *virtual* filesystem — that "file" is the kernel inventing output in RAM, nothing on disk.)
- `ps -p 1` → `systemd ... --switched-root` — PID 1 wearing a badge that says "the initramfs already pivoted to real root." The boot chain narrating itself.
- `crashkernel=...512M` in the cmdline reserves RAM for a crash-dump kernel → part of why `free` showed 15 GiB not 16.

---

## 3. Reboot detour — activating the staged kernel (a self-contained reliability story)

**The catch caught red-handed:** `/boot` contained **both** `vmlinuz-7.0.0-22` and `vmlinuz-7.0.0-27`, with the `vmlinuz`/`initrd.img` symlinks pointing at **-27** and `.old` at **-22** — while `uname -r` reported **-22 running**. So the new kernel was installed and set as next-boot default, but the *running* kernel was still the old one. **"Installed" ≠ "running"** — proven by physical evidence in `/boot`. (Ubuntu keeps the old kernel as a GRUB fallback by design.)

**Pre-flight (all read-only) — every check green:**

| Check | Result | Why it matters |
|---|---|---|
| `docker inspect ... RestartPolicy` | all 21 = `unless-stopped` | containers self-restart after reboot — no manual `compose up` |
| `grep ssd2 /etc/fstab` | present, **by UUID** | `/mnt/ssd2` re-mounts on boot; UUID survives `sda`/`sdb` reshuffles |
| `systemctl is-enabled docker` | `enabled` | container engine starts at boot |
| `systemctl is-enabled ssh.socket` | `enabled` | **the save** — see below |

**The SSH red herring (a real lesson):** `ssh.service` read `disabled`, which *looks* like "SSH won't come back → locked out of a headless box." But `ssh.socket` was `enabled`. Modern Ubuntu uses **socket activation**: systemd itself listens on port 22 (`ssh.socket`) and only spawns `ssh.service` when a connection arrives. So "disabled service + enabled socket" is **correct and safe** — diagnosed from first principles before having the systemd vocabulary.

**Blast-radius honesty (the discipline):**
- *Recoverable remotely:* session drops on reboot (reconnect after); site offline ~1–3 min; containers self-return. → low drama.
- *NOT recoverable remotely:* booting a never-booted kernel — if `-27` failed to boot, the GRUB fallback to `-22` needs a **physical keyboard**, which a headless box 1,000 km away doesn't have. Probability low (same-series LTS point release), cost high (a trip home). Named explicitly so the reboot was an *informed* call, not a rubber stamp.

**Post-reboot verification — all confirmed:** `uname -r` = `7.0.0-27`; `/mnt/ssd2` re-mounted; **21/21 containers Up**; `curl -I https://sim-obleklo.bg` = `HTTP/2 200`. Bonus: the **utmp ghost session from Ch 1 was gone** — the reboot rebuilt logind's session list, exactly as predicted.

> **Key lesson:** a staged kernel is invisible until you read `/boot` vs `uname`. Activating it is a *reboot*, which on a live box has a real, partially-irreversible blast radius — so pre-flight (will everything come back?), name what you can't undo remotely, then verify recovery against the box, not the banner.

---

## 4. systemd — the one control panel behind the whole box

systemd is **PID 1** and never exits. It does four jobs continuously: **start** (in dependency order), **supervise** (restart what dies), **track** (group every process), **log** (one unified journal). The old init only did the first and walked away; systemd is a *service manager*.

**The core abstraction: everything is a "unit,"** controlled by the same verbs (`start/stop/status/enable/disable`). Met all five types in the wild:

| Unit type | What it is | Seen as |
|---|---|---|
| `.service` | a process systemd launches + supervises | `docker.service` |
| `.socket` | listens on a port, starts its service on demand | `ssh.socket` (the reboot save) |
| `.mount` | a mounted filesystem as a unit | `mnt-ssd2.mount` |
| `.device` | hardware represented so units can depend on it | `dev-ttyS0.device` (boot blame) |
| `.scope` | processes systemd **tracks but did NOT start** | `session-2068.scope` (the ghost) |

**Two distinctions that paid off:**
- **`.service` vs `.scope` = who started it.** A service is systemd's own child (docker); a scope is *someone else's* child that systemd only babysits (a login session created by sshd). That's why the Ch1 ghost was a `.scope`.
- **enabled vs active = two independent axes.** `enabled` = "starts at next boot" (mechanically: **a symlink exists** pointing the unit into a boot target). `active` = "running now." SSH was `active` but its service `disabled` — resolved because the *socket* carries the boot duty.

**Unit file locations & the rule:**
- `/usr/lib/systemd/system/` — **vendor units, treat read-only** (a package update overwrites them; your edits vanish silently).
- `/etc/systemd/system/` — **yours, wins** over vendor.
- **Drop-ins** (`<unit>.d/override.conf`) — override *one setting* without copying the whole file. Always run `systemctl daemon-reload` after (systemd caches units in memory).

**`journalctl`** is the other half: every unit's stdout/stderr in one queryable journal. `-u <unit>` (one service), `-b` (this boot), `-f` (follow), `-p err` (errors only). The single most important debugging tool for the stack.

**Verified live — `docker.service`'s CGroup is a live network map:** the wall of `docker-proxy` processes *is* the port-publishing mechanism (one helper per published port, forwarding host-port → container-IP). Read straight from it: `443 → 172.26.0.5` (NPM), `2283 → 172.21.0.4` (Immich), `51820/udp → 172.19.0.2` (wg-easy). And the arr-stack proof: `slskd/qbittorrent/prowlarr/lidarr` all on **`172.31.0.x`** (shared `arrstack` network) while **navidrome on `172.27.0.2`** (different subnet → *not* on arrstack → couples by shared filesystem, exactly as documented). **The CGroup contains only `dockerd` + helpers, not the 21 containers** → proves the boundary: **systemd supervises `dockerd`; `dockerd` supervises the containers.** Also `ssh.socket` showed `ListenStream=0.0.0.0:22` — the one line behind the whole reboot save.

---

## 5. Two bugs found & fixed (the applied core of the chapter)

### Finding 2 (trivial) — wrong default target on a headless box
`systemctl get-default` → `graphical.target` (aims for a GUI that isn't installed). Notably, `/etc/systemd/system/default.target` **didn't exist** — so `graphical` was the **vendor default showing through**, not a choice in `/etc`. *(Teacher correction by reality: the claim "set-default rewrites an existing symlink" was wrong; with nothing in `/etc`, systemd falls back to `/usr/lib/.../default.target`.)*
**Fix:** `sudo systemctl set-default multi-user.target` — **created** the `/etc` override symlink. Reversible (`set-default graphical.target`). Takes effect next boot.

### Finding 1 (the real one) — boot-alert raced its own containerized dependency
`boot-alert.service` (a self-monitoring Ntfy ping on boot) **failed** every reboot. Journal smoking gun:
```
curl: (7) Failed to connect to 192.168.0.215 port 5555 ... Could not connect to server
```
**Diagnosis:** the script `curl`s the **Ntfy container** (`:5555`), but the unit only ordered itself `After=network-online.target`. Its *real* dependency is a **Docker-started container**, which `network-online` knows nothing about — so it fired at `10:27:40`, **6 seconds before Docker finished at `10:27:46`.** A boot-ordering race: the alert built to catch reboots was the thing that broke *during* reboots.

**Why `After=docker.service` alone is insufficient:** dockerd reporting "started" ≠ the Ntfy *container* being ready (dockerd then spends time launching 21 containers). Need belt **and** braces: order after Docker **and** make the alert tolerant of a not-yet-ready Ntfy.

**The thorough fix (rotate → env file → retry → drop-in → live test):**
1. **Rotated the leaked Ntfy password** (`docker exec -it ntfy ntfy user change-pass max`) — interactive, so the new secret never touched shell history or notes. *(The old one had been exposed by `cat`-ing the script.)*
2. **Verified persistence:** `docker inspect ntfy` → `/opt/docker/ntfy/data -> /etc/ntfy` (bind mount on SSD) — why the user/password survives container recreation. [Ch 8 preview: bind-mount-for-persistence.]
3. **Secret → locked env file:** `/etc/boot-alert.env`, `chmod 600`, `root:root` (`-rw-------` — root-only). Secret reduced from world-readable to root-only.
4. **Self-healing script:** `source /etc/boot-alert.env` + a retry loop (`curl -fsS`, 30 attempts × 2 s). Better than a fixed `sleep` — succeeds the instant Ntfy answers, copes whether that's 4 s or 40 s.
5. **Ordering drop-in:** `/etc/systemd/system/boot-alert.service.d/override.conf` with `After=docker.service` + `Requires=docker.service`. First real systemd override — *merges* onto the vendor unit, never edits it.
6. **`daemon-reload` → live test** (`systemctl start boot-alert.service`, no reboot): exit `0/SUCCESS`, log `notification sent (attempt 1)`, and **Ntfy returned a JSON receipt** confirming storage. `Type=oneshot` ends `inactive (dead)` = correct success.

**Fault isolation (the skill):** phone didn't buzz, but the receipt proved `script → Ntfy server` works → failure is isolated to the **last hop (server → phone)**, a *separate* subsystem the boot fix never touched. Polling `…/json?poll=1` showed the message stored server-side **and** incidentally revealed **Uptime-Kuma → Ntfy** firing correctly around the reboot (Immich `down` → `up`) — i.e. the *server-side* observability pipeline genuinely works; only phone delivery (VPN-up requirement / subscription auth after rotation) is outstanding.

> **Key lesson:** a service whose dependency is a *container* must order after Docker **and** tolerate the container's startup lag. Diagnose with `journalctl`, fix with an `/etc` drop-in (never edit the vendor unit), and isolate failures to a single hop with evidence before "fixing" the wrong layer.

---

## 6. Filesystem & LVM

**One tree, not lettered drives.** Everything hangs off a single root `/`. A new disk isn't "D:" — it's **mounted onto a directory** (`/mnt/ssd2` is a *directory* the second SSD is grafted onto; `cd` there transparently reads `/dev/sda1`). This decouples *where a file is* (path) from *which disk it's on* (hardware) — move data to a new disk, paths don't break.

**FHS map (the top-level tree has meaning):** `/etc` config · `/var` runtime-varying data (`/var/log`, **`/var/lib/docker`** = images) · `/home` personal · `/usr` installed programs (`/usr/bin`, `/usr/lib/systemd/system` = vendor units) · **`/usr/local/bin`** = *your* scripts (`boot-alert.sh`) · **`/opt`** = self-contained apps (`/opt/docker/<service>`, `/opt/workwear-catalog`) · `/mnt` extra mounts · `/boot` kernel+GRUB · `/proc` & `/sys` = **virtual** (kernel-generated, not on disk).

**LVM — the indirection under `/`:** `sdb3` (**PV**, physical volume) → `ubuntu-vg` (**VG**, a pool) → `ubuntu-lv` (**LV**, a slice) → ext4 → `/`. The `--` in `ubuntu--vg-ubuntu--lv` is LVM doubling literal hyphens in `VG-LV`. **Why bother vs plain partition:** live resize, span multiple disks into one pool, snapshots. **Why the asymmetry** (`/` on LVM, `/mnt/ssd2` plain): the installer gives root the flexible treatment by default; the hand-mounted bolt-on disk didn't need it. *Caveat: LVM is not RAID — spanning two disks means either disk dying kills the LV (no redundancy).*

**Verified live + the big finding:**
- `findmnt` → full tree: `/` on LVM, `/boot` on `sdb2`, `/boot/efi` vfat on `sdb1`, `/mnt/ssd2` on `sda1`.
- `df -h /` → ~28–30% (banner said 25.9% same morning; Ch1 doc said 31%) → **usage is volatile; record what consumes it, not the snapshot %.**
- **`vgs` → `VFree 0`, `pvs` → `PFree 0`: the volume group is FULLY ALLOCATED.** The installer gave the entire 116 GB partition to root, no reserve. → The "just `lvextend` into free space" move **does not apply here.** Growing `/` *requires adding a physical disk first.*
- `ls /` → `swap.img` is swap as a *file* (not a partition — why `lsblk` showed no swap partition); `cdrom`/`media` are empty desktop-heritage stubs.

> **Key lesson (HDD plan, corrected by a read-only check):** because `VFree = 0`, the future 4 TB migration is **`vgextend` (add the new PV to the pool) → `lvextend` (grow root into it) → `resize2fs` (grow the filesystem)** — bottom-up through the same PV→VG/LV→FS stack. Don't trust the plan over the disk.

---

## 7. Users, permissions & apt

**Privilege separation** = every process runs *as* a user, and a user can only touch what its permissions allow → a compromised service is *contained* to its user. Three tiers: **root (UID 0)** omnipotent; **`max` (UID 1000)** normal human; **service users (UID < 1000)** like `www-data` (UID 33) — no login, exist only to run daemons with minimal rights (confirmed isolated: in its own group only).

**`sudo`** = operate unprivileged by default, elevate per-command. Three wins: **least privilege** (typos as a normal user fail instead of executing), **auditing** (every `sudo` logged), **intent** (a deliberate speed bump). Granted by **`sudo` group** membership (`getent group sudo` → `sudo:x:27:max`). Caches auth ~15 min **per session** → why fresh SSH sessions re-prompt.

**Permission bits** — `ls -l` first ten chars: `[type][owner rwx][group rwx][others rwx]`, `r=4 w=2 x=1`:
- `/etc/boot-alert.env` = `-rw-------` = **600** → root (owner) read/write, no one else anything → the secret is locked.
- `boot-alert.sh` = `-rwxr-xr-x` = **755** → owner rwx, group/others r-x → runnable + readable + shareable (no secret in it). **Permissions encode each file's role**; the `x` bit is why `chmod +x` was needed to execute the script.

**apt vs dpkg:** **`dpkg`** = low-level engine (installs one `.deb`, no dependency resolution, no downloading). **`apt`** = high-level manager (reads repos, resolves the dependency tree, downloads, calls dpkg). The kernel arrived via apt fetching the `.deb`s and dpkg unpacking them into `/boot`.

**`update` vs `upgrade` (the classic trap):** `apt update` refreshes the **catalog** (your *knowledge*, installs nothing); `apt upgrade` **installs** the newer versions. Ritual: `update` then `upgrade`.

**Verified live:**
- `id` → groups include **`sudo` AND `docker`**. **`docker` group ≈ root** (can `docker run -v /:/host` to edit the host as root, no password) → a real exception to the "deliberately type sudo" safety story. [Hardening, Ch 18.]
- `dpkg -l | grep -c '^ii'` → **739 packages** installed for a "minimal" server — the dependency tree apt manages on your behalf.
- `apt list --upgradable` → the **3 updates = `sg3-utils` family** (`libsgutils2`, `sg3-utils`, `sg3-utils-udev`), a `…ubuntu3 → …ubuntu3.1` point release of SCSI disk utilities. **Trivial, no service restart** → unlike June 20's Docker upgrade, this one won't sever the session.
- `sources.list.d/` → `ubuntu.sources` (base) · `docker.list` (official Docker repo → upstream `29.6.1`) · `nodesource.sources` (Node.js for the website build) · `ubuntu.sources.curtin.orig` (**inert installer leftover** — apt ignores `.orig`; candidate to remove).

> **Key lesson (blast radius, learned June 20):** `apt upgrade` can restart Docker → kill the wg-easy container carrying your SSH session → strand a half-finished apt on a `needrestart` prompt to a dead TTY. **Run upgrades in `tmux`/`screen`** (survives a dropped tunnel) and make `needrestart` non-interactive on a headless box.

---

## 8. Parked items — require physical presence (carried from Ch 1)

| Item | Why parked | If done wrong remotely |
|---|---|---|
| BIOS **AC Power Recovery** — confirm enabled | a power blip otherwise leaves the site down until someone presses the button | can't be set remotely at all |
| BIOS `1.24.0` → newer / **GDS microcode** flag | ~4-yr-old firmware; `lscpu` shows `Gather data sampling: Vulnerable: No microcode` | highest blast radius in the stack — a blip mid-flash can brick the board. **Reversible Linux alternative:** `intel-microcode` package (loads microcode at boot, independent of BIOS) — Ch 17, run in `tmux`. |

---

## 9. Open to-dos — `homelab` repo

- [ ] (Optional) Apply the 3 `sg3-utils` updates via `tmux` → `apt update && apt upgrade` (low-risk; buys a clean banner).
- [ ] Phone not receiving Ntfy pushes — re-auth subscription with the **new** password; confirm VPN-up requirement (Ntfy is `:5555`, VPN-only). [Ch 15.]
- [ ] Immich `down → up` alert spanned ~1 h post-reboot vs 7-min container start — check Uptime-Kuma interval / Immich true readiness. [Ch 15.]
- [ ] Remove inert `ubuntu.sources.curtin.orig` from `sources.list.d/` (cosmetic).
- [ ] (Optional, cosmetic) Evict any future utmp ghost with `loginctl terminate-session <id>` — or let reboots clear it.
- [ ] **Next physical visit:** verify BIOS AC Power Recovery; consider BIOS/microcode.

## 10. State to promote into canonical docs (`docs/`)

- **Kernel `7.0.0-27-generic` running** [verified June 28] — rebooted to activate; `-22` retained as GRUB fallback. *(Supersedes "kernel update pending".)*
- **`ubuntu-vg` fully allocated (`VFree 0`)** — root spans the entire 116 GB PV; growing `/` requires **adding a disk** (`vgextend` → `lvextend` → `resize2fs`), not just a command. *(hardware.md / HDD plan.)*
- **Root usage volatile ~26–31%** — record consumers (`/var/lib/docker`, `/opt/docker`), not the %.
- **`max` ∈ `docker` group = root-equivalent** — note in `known-limitations.md` (hardening).
- **default target now `multi-user.target`** (was vendor-default `graphical.target`).
- **`boot-alert.service` fixed** — drop-in (`After/Requires=docker.service`) + self-healing retry + secret in `chmod 600` env file + rotated credential. Add the sanitized script + `.env.example` (placeholder only) to the repo.

---

## 11. Portfolio / learning angles

- **Boot-ordering race, end to end.** "My server's boot-notification raced its own containerized dependency — it `curl`ed Ntfy 6 seconds before Docker started it. Diagnosed via journald (curl exit 7, connection refused), fixed with a systemd drop-in (`After`/`Requires=docker.service`) plus a self-healing retry loop, and hardened it by moving the secret into a `chmod 600` env file and rotating the exposed credential." Diagnosis + systemd internals + a security fix in one tidy story.
- **Informed reboot under irreversibility.** Spotted a staged kernel in `/boot`, ran a real pre-flight that flagged `ssh.service` as `disabled`, *correctly diagnosed it as harmless socket-activation* rather than blindly "fixing" it, weighed the headless-reboot blast radius honestly, and verified full recovery. Shows judgment, not just assembly.
- **Full-VG discovery.** A read-only `vgs` (`VFree 0`) corrected the HDD-migration plan *before* it bit. "Don't trust the plan over the disk."
- **Fault isolation by evidence.** Polled Ntfy's stored messages to prove the alert reached the server, isolating the failure to the phone hop — and incidentally proved the Uptime-Kuma alerting pipeline fires correctly.
- **Teacher-corrected-by-reality moments** (the ethos turned on the explanations themselves): the `default.target` symlink that didn't exist (vendor-default fallback), and `sshd:` vs `sshd-session`. Verify reality — even against the teacher.

---

## Appendix — read-only commands used (blast radius: zero)

```bash
# Boot chain
uname -a ; uname -r                 # running kernel
cat /proc/cmdline                   # kernel command line (root=, crashkernel=)
ls -lh /boot                        # kernels + initramfs; spot a staged update
ps -p 1 -o pid,comm,args            # PID 1 = systemd, --switched-root

# systemd
systemctl status <unit> --no-pager  # Loaded/Active/Main PID/CGroup
systemctl cat <unit>                 # unit file + any drop-ins (merge proof)
systemctl is-enabled / is-active <unit>
systemctl list-units --type=mount    # filesystems as units
systemctl get-default                # boot target
systemd-analyze blame                # per-unit startup time
journalctl -u <unit> -b --no-pager   # one unit's logs, this boot
journalctl -b -p err --no-pager      # errors since boot

# Filesystem & LVM
findmnt -t ext4,vfat                 # mount tree
df -h /                              # root usage (a snapshot)
ls /                                 # FHS map
sudo pvs ; sudo vgs ; sudo lvs       # LVM stack — watch vgs VFree

# Users, permissions, apt
id ; getent group sudo ; id www-data # identity, elevation, a service user
ls -l <file>                         # decode permission bits
apt list --upgradable                # pending updates, named
dpkg -l | grep -c '^ii'              # installed package count
ls /etc/apt/sources.list.d/          # configured repositories

# Reboot verification (post-activation)
docker ps --format '{{.Names}}: {{.Status}}'
curl -I https://sim-obleklo.bg
```

---

*Next: Chapter 3 — Networking Fundamentals (IP addresses, subnets, ports, LAN/WAN, NAT) — where the `172.19.0.2` wg-easy address finally gets fully explained.*
