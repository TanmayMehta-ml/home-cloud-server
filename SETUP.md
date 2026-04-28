# Setup Guide — Self-Hosted Home Cloud Server

Complete implementation guide including every command, every issue encountered, and exact fixes. Written for Ubuntu Server 26.04 LTS on VirtualBox, with Nextcloud AIO behind Cloudflare Tunnel.

> **Environment:** Windows 11 host → VirtualBox 7.x → Ubuntu Server 26.04 LTS VM

---

## Prerequisites

### Accounts Required
- Cloudflare account (free) — cloudflare.com
- Domain name — purchased from GoDaddy, nameservers pointed to Cloudflare

### Downloads Required
- VirtualBox 7.x + Extension Pack — virtualbox.org
- Ubuntu Server 26.04 LTS ISO — ubuntu.com/download/server

---

## Phase 1 — VM Creation and Ubuntu Installation

### VirtualBox VM Settings
```
Name:        homecloud
RAM:         4096 MB (4GB)
CPUs:        4
Disk:        30 GB (VDI, Dynamically Allocated)
Network:     Bridged Adapter → select active WiFi adapter
EFI:         Disabled
```

### Ubuntu Installer Choices
```
Installation type:  Ubuntu Server (not minimized)
Storage:            Use entire disk
SSH:                Install OpenSSH server ✓
Snaps:              None
```

### After First Boot — Find VM IP
```bash
ip addr show
# Note the inet address under enp0s3, e.g. 192.168.1.10
# This is your VM's LAN IP — save it
```

### Connect via SSH from Windows PowerShell
```powershell
ssh yourusername@192.168.1.10
```

### System Updates
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### Install OpenSSH (if not installed)
```bash
sudo apt install openssh-server -y
sudo systemctl start ssh
sudo systemctl enable ssh
```

### UFW Firewall Setup
```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

---

## Phase 2 — Docker Installation

### Install Docker Engine (Official Method)
> ⚠️ Do NOT use `apt install docker.io` — it installs an outdated version.

```bash
sudo apt install ca-certificates curl -y

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin -y
```

### Add User to Docker Group
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Verify Docker Works
```bash
docker run hello-world
# Expected: "Hello from Docker!" message
```

### Fix Docker DNS Resolution
> ⚠️ Without this fix, `nextcloud-aio-notify-push` container will restart in a loop.
> The container cannot resolve external hostnames using Docker's default DNS.

```bash
sudo nano /etc/docker/daemon.json
```

Paste:
```json
{
  "dns": ["1.1.1.1", "8.8.8.8"]
}
```

```bash
sudo systemctl restart docker
```

---

## Phase 3 — Nextcloud AIO Deployment

### Open Required UFW Ports
```bash
sudo ufw allow 8080
```

> ⚠️ Do NOT open ports 80, 443, or 8443 — Cloudflare Tunnel handles external traffic.
> Port 8080 is for AIO management interface on LAN only.

### Deploy Nextcloud AIO in Reverse Proxy Mode
> ⚠️ Critical: Must use reverse proxy mode when running behind Cloudflare Tunnel.
> Standard deployment (without `APACHE_PORT` and `SKIP_DOMAIN_VALIDATION`) will fail.

```bash
sudo docker run \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=0.0.0.0 \
  --env SKIP_DOMAIN_VALIDATION=true \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  ghcr.io/nextcloud-releases/all-in-one:latest
```

**Why these flags:**
- `APACHE_PORT=11000` — AIO Apache listens on 11000, not 443, for reverse proxy compatibility
- `APACHE_IP_BINDING=0.0.0.0` — binds to all interfaces so cloudflared can reach it
- `SKIP_DOMAIN_VALIDATION=true` — bypasses AIO's domain check which fails due to Airtel NAT loopback

### Access AIO Interface
```
https://192.168.1.10:8080
```
Accept self-signed certificate warning. Save the AIO passphrase shown.

### AIO Setup Choices
```
Domain:           your-domain.in
Office Suite:     Nextcloud Office (Collabora)
Optional:         None (RAM constrained on 8GB host)
Timezone:         Asia/Kolkata
```

Click **Install** — downloads ~4–5 GB of Docker images. Takes 30–60 minutes on a home connection.

### Verify All Containers Running
```bash
sudo docker ps
# Should show 9-10 containers all with STATUS: healthy
```

---

## Phase 4 — Cloudflare Tunnel Setup

### Install cloudflared
```bash
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | \
  sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] \
  https://pkg.cloudflare.com/cloudflared any main' | \
  sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update && sudo apt install cloudflared -y
```

### Authenticate with Cloudflare
```bash
cloudflared tunnel login
# Opens a URL — open in browser, select your domain, authorize
```

### Create Tunnel
```bash
cloudflared tunnel create homecloud
# Saves Tunnel ID and credentials JSON — note the Tunnel ID
```

### Create Tunnel Config
```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

Paste (replace YOUR_TUNNEL_ID and your domain):
```yaml
tunnel: YOUR_TUNNEL_ID
credentials-file: /home/yourusername/.cloudflared/YOUR_TUNNEL_ID.json

ingress:
  - hostname: your-domain.in
    service: http://localhost:11000
  - service: http_status:404
```

> ⚠️ Use `http://` not `https://` for localhost:11000.
> Port 11000 serves plain HTTP — using https:// causes `tls: first record does not look like a TLS handshake` error.

### Create DNS Record (Automatic)
```bash
cloudflared tunnel route dns homecloud your-domain.in
```

### Test Tunnel
```bash
cloudflared tunnel run homecloud
# Test from phone on mobile data (not WiFi) — open https://your-domain.in
```

### Install as System Service
```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
sudo cp ~/.cloudflared/YOUR_TUNNEL_ID.json /etc/cloudflared/YOUR_TUNNEL_ID.json
sudo cp ~/.cloudflared/cert.pem /etc/cloudflared/cert.pem

# Update credentials path in config
sudo nano /etc/cloudflared/config.yml
# Change credentials-file to: /etc/cloudflared/YOUR_TUNNEL_ID.json

sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

---

## Phase 5 — Security Hardening

### Install and Configure fail2ban
```bash
sudo apt install fail2ban -y

sudo nano /etc/fail2ban/jail.local
```

Paste:
```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
backend = systemd
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo systemctl status fail2ban
```

### Enable Automatic Security Updates
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
# Select Yes
```

### Fix Time Sync (Required for TOTP 2FA)
> ⚠️ If VM clock drifts, TOTP verification fails silently with "could not verify code".
> Fix this BEFORE enabling 2FA.

```bash
sudo apt install chrony -y
sudo systemctl enable chrony
sudo systemctl start chrony
sudo chronyc makestep
date
# Verify time matches current UTC time
```

### Enable 2FA on Admin Account
In Nextcloud browser:
```
Profile icon → Personal Settings → Security → TOTP → Enable
Scan QR with Google Authenticator or Aegis
Generate and save backup codes offline
```

### Final UFW State
```bash
sudo ufw status
# Should show only port 22/tcp allowed
# Ports 80, 443, 8443 should NOT be open — Cloudflare handles these
```

---

## Phase 6 — User Management

### Create Users via Nextcloud Admin
```
https://your-domain.in/index.php/settings/users
→ New account button
```

Recommended structure:
```
Username:     tanmay.mehta    Group: admin      Quota: Unlimited
Username:     ravi.mehta      Group: family     Quota: 200GB (production)
Username:     suchita.mehta   Group: family     Quota: 200GB (production)
Username:     shreya.mehta    Group: family     Quota: 150GB (production)
```

### Mobile App Setup (Android + iOS)
```
App: Nextcloud (official — Play Store / App Store)
Server: https://your-domain.in
Login: username + password
```

---

## Phase 7 — Backup Configuration

### Create Backup Directory
```bash
sudo mkdir -p /mnt/backup
sudo chmod 777 /mnt/backup
```

### Configure in AIO Interface
```
https://192.168.1.10:8080
→ Backup and restore section
→ Local backup location: /mnt/backup
→ Submit
→ Create backup (first manual backup)
→ Set daily backup time: 02:00
```

---

## Common Issues and Fixes

### Issue: SSH connection refused after Ubuntu install
```bash
# Inside VirtualBox window:
sudo apt install openssh-server -y
sudo systemctl start ssh
```

### Issue: notify-push container restarting in loop
```
Error: "The notify_push binary was not found"
Cause: Docker cannot resolve external DNS to download binary
Fix: Add DNS servers to /etc/docker/daemon.json (see Phase 2)
```

### Issue: 502 Bad Gateway on domain
```
Cause: Cloudflare tunnel pointing to wrong port or protocol
Fix: Ensure config.yml uses http://localhost:11000 (not https, not 443, not 80)
```

### Issue: AIO domain validation fails
```
Error: "Domain does not point to this server"
Cause 1: Cloudflare proxy (orange cloud) intercepting validation
Cause 2: NAT loopback not supported by Airtel router
Fix: Use SKIP_DOMAIN_VALIDATION=true in docker run command
```

### Issue: TOTP "could not verify code"
```
Cause: VM clock out of sync
Fix: sudo chronyc makestep
```

### Issue: cloudflared service install fails — "no config file found"
```
Cause: Service installer looks in /etc/cloudflared/, not ~/.cloudflared/
Fix: Copy all cloudflared files to /etc/cloudflared/ and update credentials path
```

### Issue: tls handshake error in cloudflared logs
```
Error: "tls: first record does not look like a TLS handshake"
Cause: config.yml using https:// for localhost:11000 which serves plain HTTP
Fix: Change to http://localhost:11000
```

### Issue: "Client sent HTTP request to HTTPS server"
```
Cause: config.yml using http:// but AIO interface requires https
Fix: AIO management (8080) is always HTTPS. Nextcloud traffic (11000) is HTTP.
     Use http://localhost:11000 in tunnel config — Cloudflare adds HTTPS externally.
```

---

## Useful Diagnostic Commands

```bash
# Check all containers status
sudo docker ps

# Check container logs
sudo docker logs nextcloud-aio-nextcloud --tail 50
sudo docker logs nextcloud-aio-notify-push --tail 20

# Check resource usage
sudo docker stats --no-stream

# Check disk usage
df -h

# Check tunnel status
sudo systemctl status cloudflared

# Check fail2ban status
sudo fail2ban-client status sshd

# Check UFW rules
sudo ufw status verbose

# Restart Docker (and all containers)
sudo systemctl restart docker

# Force time sync
sudo chronyc makestep
```

---

## Production Migration (Raspberry Pi 5)

### Hardware Required
- Raspberry Pi 5 8GB
- Official Pi 5 27W USB-C power supply
- Official Pi 5 case with fan
- 32GB SanDisk microSD (OS only)
- 1TB Seagate/Samsung portable SSD (data storage)
- Ethernet cable (optional but recommended)

### Migration Steps
1. Flash Ubuntu Server 26.04 LTS to microSD via Raspberry Pi Imager
2. Enable SSH, set credentials in Imager before flashing
3. Boot Pi, SSH in, run all phases above
4. Mount SSD: `sudo mkfs.ext4 /dev/sda1 && sudo mount /dev/sda1 /mnt/ssd`
5. Add to `/etc/fstab` for auto-mount on boot
6. Run final backup on VM via AIO interface
7. Copy backup to Pi: `scp -r /mnt/backup user@pi-ip:/mnt/ssd/backup`
8. Restore via AIO on Pi
9. Stop cloudflared on VM: `sudo systemctl stop cloudflared`
10. Start cloudflared on Pi — same tunnel ID, same domain, zero Cloudflare changes needed

### Switch WiFi to Ethernet on Pi
```bash
# Plug Ethernet into router LAN port
ip addr show eth0
# Confirm IP assigned

# Optionally disable WiFi
sudo nano /etc/netplan/50-cloud-init.yaml
# Remove wifis section
sudo netplan apply
```
Cloudflare Tunnel reconnects automatically. No other changes needed.

---

## Production Storage Layout (1TB SSD)

| Purpose | Size |
|---|---|
| Docker images + Nextcloud app | ~15 GB |
| Database + configs | ~5 GB |
| BorgBackup (local) | ~100 GB reserved |
| User storage | ~880 GB |

Recommended user quotas:
```
tanmay.mehta:   300 GB
user.one:       200 GB
user.two:       200 GB
user.three:     150 GB
Buffer:          30 GB
```
