# OS Setup — Ubuntu Server 26.04 LTS

## Why Ubuntu Server 26.04 LTS

- Long Term Support release (supported until 2031)
- Largest homelab/Docker tutorial ecosystem — more answers when Googling errors
- Familiar apt/Debian package ecosystem (same as Linux Mint)
- Widely used as the base for Immich and Nextcloud self-hosting guides

---

## 1. Create a Bootable USB

**Download:** [Ubuntu Server 26.04 LTS](https://ubuntu.com/download/server)  
**Flash tool:** [Balena Etcher](https://etcher.balena.io) (free, cross-platform)

1. Open Etcher → Flash from file → select the `.iso`
2. Select your USB drive (4 GB minimum — it will be wiped)
3. Flash and wait

---

## 2. Boot the OptiPlex from USB

1. Insert USB into the OptiPlex
2. Power on and press **F12** repeatedly to open the one-time boot menu
3. Select your USB drive
4. The Ubuntu Server installer will load

---

## 3. Installation Steps

Walk through the installer with these choices:

### Language & Keyboard
- Language: English
- Keyboard: match your layout

### Network
- Let it auto-configure via DHCP for now
- You'll set a static IP (or DHCP reservation) after install

### Storage
- Use the 128 GB SSD as the install target
- Choose **default LVM setup** — fine for a single-drive setup
- Do **not** encrypt the disk (makes headless reboots annoying)

### Profile Setup
- Your name: up to you
- Server name (hostname): `homelab` (or whatever you chose — pick now, it's permanent)
- Username: your choice (e.g. `maxim`)
- Password: strong, you'll use this for `sudo`

### SSH
- ✅ Check **Install OpenSSH server** — this is essential for headless management

### Featured Snaps
- ✅ **Skip all of them** — don't install any snaps. Docker will be installed manually via apt.

### Finish
- Let it install and reboot
- Remove the USB when prompted

---

## 4. First Boot

You'll land at a login prompt. Log in with the username and password you set.

Verify you're online:

```bash
ping -c 4 google.com
```

Update the system immediately:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 5. Server Hardening

### 5a. Disable password SSH login (use keys only)

On your **laptop**, generate an SSH key pair if you don't have one:

```bash
ssh-keygen -t ed25519 -C "homelab"
```

Copy your public key to the server:

```bash
ssh-copy-id your-username@server-ip
```

Verify key login works:

```bash
ssh your-username@server-ip
```

Once confirmed, disable password auth:

```bash
sudo nano /etc/ssh/sshd_config
```

Set these values:
```
PasswordAuthentication no
PermitRootLogin no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

> ⚠️ Don't close your current SSH session until you've verified key login works in a new terminal window.

### 5b. Configure UFW firewall

UFW (Uncomplicated Firewall) is Ubuntu's built-in firewall wrapper.

```bash
# Allow SSH before enabling — or you'll lock yourself out
sudo ufw allow ssh

# Allow WireGuard (we'll need this later)
sudo ufw allow 51820/udp

# Enable the firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

### 5c. Install fail2ban

fail2ban watches log files and temporarily bans IPs that fail authentication repeatedly.

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Default config is sufficient for now — it protects SSH automatically.

---

## 6. Set a Static Local IP

Your server needs a stable local IP so other services can reliably reach it.

**Recommended method: DHCP reservation on your router**

1. Log into your router
2. Find the DHCP client list and identify your server by hostname or MAC address
3. Create a reservation (static lease) for that MAC address
4. Assign it a fixed IP (e.g. `192.168.0.215`)
5. Reboot the server: `sudo reboot`

Verify the IP stuck:

```bash
ip a
```

---

## 7. Post-Install Cleanup

Remove cloud-init (not needed on bare metal):

```bash
sudo apt purge cloud-init -y
sudo rm -rf /etc/cloud /var/lib/cloud
```

Disable snapd if you want a fully snap-free system:

```bash
sudo systemctl disable snapd
sudo apt purge snapd -y
```

---

## 8. Install Docker

Docker is not in Ubuntu's default repos — install it from Docker's official repo:

```bash
# Add Docker's GPG key
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Allow your user to run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker run hello-world
```

---

## Verification Checklist

- [ ] Can SSH in using key (no password prompt)
- [ ] Password SSH login is disabled
- [ ] UFW is active (`sudo ufw status`)
- [ ] fail2ban is running (`sudo systemctl status fail2ban`)
- [ ] Server has a stable local IP
- [ ] `docker run hello-world` succeeds
- [ ] System is fully updated (`sudo apt update && sudo apt list --upgradable`)
