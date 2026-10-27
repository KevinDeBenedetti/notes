# Debian

*Configuration and hardening guide for a Debian 13 (Trixie) VPS.*

## Automated Bootstrap (Recommended)

For VPS nodes used with the private `infra` repository, the full provisioning is automated via the dotfiles setup script. This is the standard way to prepare a fresh node:

```bash
# From your local machine (infra repo):
make setup-master    # bootstrap VPS1
make setup-worker    # bootstrap VPS2
```

Equivalent manual command on each VPS (as root):

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/KevinDeBenedetti/dotfiles/main/os/debian/init.sh) -a
```

What this installs automatically:
- Base packages (curl, git, vim, htop, unzip, …)
- Non-root sudo user with SSH key and NOPASSWD sudo
- SSH hardening (no root login, key-only auth, AllowUsers)
- UFW firewall (deny-in, allow SSH/HTTP/HTTPS)
- Fail2Ban (SSH brute-force protection)
- AppArmor (mandatory access control)
- Kernel sysctl hardening (`/etc/sysctl.d/99-security.conf`)
- Docker CE (optional, `-a` flag)

> The sections below document the manual steps for reference or when not using dotfiles.

---

## Initial Setup

```bash
# Change the root password
passwd

# Update the package lists and upgrade
sudo apt update && sudo apt upgrade -y

# Clean up obsolete packages
sudo apt autoremove -y && sudo apt autoclean

# Reboot to apply kernel updates
sudo reboot
```

> **Best practice**: Enable automatic security updates with `unattended-upgrades`.

---

## Create a Non-root User

Always operate as a non-root user with sudo privileges. The automated bootstrap creates this user for you, but here is the manual procedure:

```bash
# Create user
adduser kevin

# Add to sudo group
usermod -aG sudo kevin

# Copy root SSH key to the new user
mkdir -p /home/kevin/.ssh
cp /root/.ssh/authorized_keys /home/kevin/.ssh/authorized_keys
chown -R kevin:kevin /home/kevin/.ssh
chmod 700 /home/kevin/.ssh
chmod 600 /home/kevin/.ssh/authorized_keys
```

For non-interactive automated operations (k3s install scripts, CI), grant passwordless sudo:

```bash
echo "kevin ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/kevin
chmod 440 /etc/sudoers.d/kevin
```

---

## SSH Hardening

Edit `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

```ini
# /etc/ssh/sshd_config

# Change the default port (optional — reduces automated scan noise)
Port 22

# Disable root login
PermitRootLogin no

# Key-based auth only
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no

# Restrict users allowed to SSH in
AllowUsers kevin

# Limit brute-force surface
MaxAuthTries 3
MaxSessions 5

# Idle session timeout (5 min)
ClientAliveInterval 300
ClientAliveCountMax 2

# Protocol 2 only
Protocol 2
```

```bash
# Validate config before restarting
sudo sshd -t

# Restart SSH
sudo systemctl restart sshd

# ⚠️ Test connection in a NEW terminal BEFORE closing your current session
ssh kevin@<VPS_IP> -p 22
```

---

## Kernel Parameters (sysctl)

The dotfiles setup writes security defaults to `/etc/sysctl.d/99-security.conf`. For k3s nodes, an additional file overrides `ip_forward`:

```bash
# /etc/sysctl.d/99-security.conf (set by dotfiles)
net.ipv4.ip_forward = 0

# /etc/sysctl.d/99-z-k3s.conf (set by k3s install scripts — MUST sort after)
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

The `99-z-` prefix ensures the k3s file is applied **after** the security baseline.

Apply without rebooting:

```bash
sudo sysctl --system
sudo sysctl net.ipv4.ip_forward    # should print 1 on k3s nodes
```

---

## Firewall (UFW)

See [ufw.md](ufw.md) for detailed rules, including k3s-specific configuration.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw status verbose
```

---

## Fail2Ban

See [fail2ban.md](fail2ban.md) for configuration details.

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Minimal SSH jail (`/etc/fail2ban/jail.local`):

```ini
[sshd]
enabled  = true
port     = ssh
filter   = sshd
maxretry = 3
findtime = 5m
bantime  = 30m
backend  = systemd
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

---

## Timezone & Locale

```bash
# Set timezone
sudo timedatectl set-timezone Europe/Paris
timedatectl

# Reconfigure locales
sudo dpkg-reconfigure locales

# Reconfigure keyboard
sudo dpkg-reconfigure keyboard-configuration
```

---

## Useful Diagnostic Commands

```bash
# System resources
htop
free -h
df -h

# Network
ip addr
ip route
ss -tulnp

# Logs
journalctl -xe
journalctl -u sshd -f
journalctl -u ufw --since "1 hour ago"

# Failed login attempts
sudo grep "Failed password" /var/log/auth.log | tail -20
sudo fail2ban-client status sshd
```

---

## Resources

- [Debian Administrator's Handbook — Security](https://debian-handbook.info/browse/stable/security.html)
- [OVH VPS Security Guide](https://help.ovhcloud.com/csm/fr-vps-security-tips?id=kb_article_view&sysparm_article=KB0047708)
- [dotfiles — debian init.sh](https://github.com/KevinDeBenedetti/dotfiles/blob/main/os/debian/init.sh)
