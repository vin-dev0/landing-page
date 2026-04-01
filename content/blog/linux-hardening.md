+++
title = "Quick Guide for Linux Infrastructure Hardening"
date = "2026-03-29"
draft = false
description = "A deep dive into securing your Linux infrastructure, from VPS hardening with SSH and Fail2Ban to advanced container security and auditing tools."
+++

Securing your infrastructure doesn't always require a six-figure enterprise suite. Whether you are running a single VPS, a fleet of VMs, or a cluster of containers, the most effective defense often comes from mastering the "basics."

In 2026, the threat landscape is automated, making "security by obscurity" a dangerous myth. Here is how to harden your Linux environment using standard, open-source tools.

---

## 1. Hardening the Gateway: VPS & VM Security
Before you touch a container, your host (the VPS or VM) must be a fortress. If the host is compromised, every container on it is effectively compromised too.

### SSH: Kill the Passwords
Bots crawl the internet 24/7 looking for SSH ports. The first step is to disable password authentication entirely and use **Ed25519** keys.

```bash
# Generate a modern, secure key on your LOCAL machine
ssh-keygen -t ed25519 -C "admin@yourdomain.com"

# Copy it to your server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@your-vps-ip
```

Then, edit `/etc/ssh/sshd_config` on the server:
```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222 # Changing the default port 22 reduces log noise significantly
```
Don't forget to restart the service: `sudo systemctl restart ssh`.

### The "Auto-Bouncer": Fail2Ban
Fail2Ban watches your logs for repeated failed login attempts and tells the firewall to drop that IP.

```bash
sudo apt install fail2ban -y
```
Create a local config (`/etc/fail2ban/jail.local`) to protect SSH:
```ini
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 1h
```

### The Shield: UFW (Uncomplicated Firewall)
Deny everything by default and only open what you absolutely need.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp  # Your new SSH port
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

---

## 2. Securing the Core: Containers (Docker/Podman)
Containers share the host's kernel. A "container escape" allows an attacker to hop from a web app straight into your host's root shell.

### Rule #1: Use a Non-Root User
By default, many Dockerfiles run as root. If an attacker exploits your app, they are root inside the container.

**Bad Dockerfile:**
```dockerfile
FROM node:20
COPY . .
CMD ["node", "app.js"] # Runs as root!
```

**Secure Dockerfile:**
```dockerfile
FROM node:20-alpine
# Create a system user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --chown=appuser:appgroup . .
CMD ["node", "app.js"]
```

### Rule #2: Limit Capabilities
Linux has "Capabilities" that break root's power into smaller pieces. Most containers don't need things like `NET_ADMIN` or `SYS_ADMIN`.

```bash
# Drop ALL privileges, then add back only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE my-app
```

### Rule #3: Read-Only Filesystem
Prevent attackers from downloading and running malware scripts by making the container's root filesystem read-only.

```bash
docker run --read-only --tmpfs /tmp my-app
```

---

## 3. Auditing & Scanning Tools
You can't fix what you don't see. Use these basic tools to find "holes" in your setup.

### For the Host: Lynis
Lynis is a battle-tested security auditing tool for Linux. It scans your OS and gives you a "Hardening Index" score.

```bash
sudo apt install lynis
sudo lynis audit system
```

### For Containers: Trivy
Trivy is the gold standard for scanning container images for known vulnerabilities (CVEs).

```bash
# Scan a local image before you deploy it
trivy image my-custom-app:latest
```

## Conclusion
Security is a continuous cycle of auditing, patching, and hardening. By following these steps, you build a layered defense that keeps your infrastructure resilient.