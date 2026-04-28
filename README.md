# 🏠 Self-Hosted Home Cloud Server

A fully functional, secure, private home cloud server built for personal and family use — accessible from anywhere in the world, running on a laptop VM as proof of concept before migrating to Raspberry Pi 5.

---

## Project Overview

This project builds a self-hosted alternative to Google Drive or iCloud using **Nextcloud AIO** deployed on **Ubuntu Server** inside a **VirtualBox VM**, exposed to the internet via **Cloudflare Tunnel** — solving the CGNAT problem common with Indian ISPs like Airtel.

The server supports multiple family users with isolated storage, strong security controls, automated backups, and global HTTPS access — all without a static IP or open router ports.

---

## Architecture

```
[Phones / Laptops / Browsers]
            │
            ▼
  [Cloudflare Edge — DNS + SSL]
            │  (outbound encrypted tunnel)
            ▼
  [cloudflared daemon — Ubuntu Server]
            │
            ▼
  [Nextcloud AIO — Docker Containers]
            │
     ┌──────┴───────┐
     │              │
[PostgreSQL]    [Redis Cache]
     │
[External SSD / VM Disk — User Storage]
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Hypervisor | VirtualBox 7.x |
| Server OS | Ubuntu Server 26.04 LTS |
| Cloud Platform | Nextcloud AIO v12.9.2 (Docker) |
| Database | PostgreSQL (via AIO) |
| Cache | Redis (via AIO) |
| Reverse Tunnel | Cloudflare Tunnel (cloudflared 2026.3.0) |
| Container Runtime | Docker Engine 29.4.1 |
| Firewall | UFW |
| Brute Force Protection | fail2ban 1.1.0 |
| Backup | Nextcloud AIO BorgBackup |
| SSL | Cloudflare (automatic) |
| 2FA | Nextcloud TOTP |
| Time Sync | Chrony |

---

## Key Problems Solved

### 1. CGNAT (Carrier Grade NAT) — Airtel India
Airtel uses CGNAT, meaning no public IPv4 address is assigned to the home router. Port forwarding does not work. **Solution:** Cloudflare Tunnel creates an outbound encrypted connection from the server to Cloudflare's edge — no open ports, no public IP required.

### 2. Nextcloud AIO Reverse Proxy Mode
AIO by default tries to handle its own TLS. Behind Cloudflare Tunnel, it must run in reverse proxy mode with `APACHE_PORT=11000` and `SKIP_DOMAIN_VALIDATION=true`. Standard AIO deployment fails behind any reverse proxy without these flags.

### 3. Docker DNS Resolution Failure
The `nextcloud-aio-notify-push` container repeatedly restarted because Docker's default DNS could not resolve external hostnames inside containers. **Solution:** Added Cloudflare DNS (`1.1.1.1`) and Google DNS (`8.8.8.8`) to `/etc/docker/daemon.json`.

### 4. NAT Loopback (Hairpinning) Failure
AIO's domain validation tries to reach the domain from inside the container. Airtel routers do not support NAT loopback — local containers cannot resolve the public domain back to the local server. **Solution:** `SKIP_DOMAIN_VALIDATION=true` environment variable.

### 5. TOTP Verification Failure
TOTP codes are time-based. If the VM clock drifts even 30 seconds, verification fails silently. **Solution:** Install and force-sync `chrony` before enabling 2FA.

---

## Features

- Global HTTPS access via custom domain (`mehtacloud.in`)
- 4 isolated user accounts with individual storage quotas
- Nextcloud Office (Collabora) for document editing
- Nextcloud Talk for family messaging
- File versioning and recovery
- Automated daily backups at 2 AM via BorgBackup
- Brute force protection (fail2ban + Nextcloud built-in)
- 2FA (TOTP) on admin account
- Automatic OS security updates (unattended-upgrades)
- UFW firewall — only SSH port open

---

## Network Flow

```
User Device
    │
    │ HTTPS (port 443)
    ▼
Cloudflare Edge (mehtacloud.in)
    │
    │ Encrypted QUIC tunnel
    ▼
cloudflared service (Ubuntu VM)
    │
    │ HTTP (localhost:11000)
    ▼
nextcloud-aio-apache (Caddy reverse proxy)
    │
    ▼
nextcloud-aio-nextcloud (PHP-FPM)
```

---

## User Setup

| User | Role | Quota |
|---|---|---|
| admin | System Admin | Unlimited |
| tanmay.mehta | Personal Admin | Unlimited |
| ravi.mehta | Family | 10 GB (POC) |
| suchita.mehta | Family | 10 GB (POC) |
| shreya.mehta | Family | 10 GB (POC) |

---

## POC vs Production Hardware

| Component | POC (Current) | Production (Planned) |
|---|---|---|
| Server | Windows 11 Laptop + VirtualBox VM | Raspberry Pi 5 8GB |
| Storage | 30GB VM virtual disk | 1TB Seagate Portable SSD |
| OS Drive | VM disk | 32GB SanDisk microSD |
| Connection | WiFi | Ethernet (router LAN port) |
| Power | Laptop power | Official Pi 5 27W USB-C PSU |

---

## Migration Path

POC → Pi (WiFi) → Pi (Ethernet) — all three stages require zero Cloudflare reconfiguration. The tunnel reconnects automatically regardless of network interface change.

---

## Security Checklist

- [x] UFW enabled — only port 22 open
- [x] fail2ban — SSH brute force protection
- [x] Nextcloud built-in brute force protection
- [x] HTTPS enforced
- [x] 2FA (TOTP) on admin account
- [x] Strong passwords — 20+ character passphrases
- [x] Automatic unattended security updates
- [x] Docker DNS hardened
- [x] No open router ports
- [x] Cloudflare proxy enabled (origin IP hidden)

---

## Domain & Networking Cost

| Item | Cost |
|---|---|
| `mehtacloud.in` domain (GoDaddy) | ₹599 first year / ₹899 renewal |
| Cloudflare Tunnel | Free |
| Cloudflare DNS + Proxy | Free |

**Total annual cost: ₹899/year**

---

## Author

**Tanmay Mehta**  
Home Lab / Self-Hosted Infrastructure  
[mehtacloud.in](https://mehtacloud.in)
