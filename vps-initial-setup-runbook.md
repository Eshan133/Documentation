# Contabo VPS Initial Setup Runbook

A step-by-step guide for hardening and configuring a fresh Ubuntu VPS (tested against Ubuntu 22.04/24.04). Run sections in order — don't skip the SSH verification step in Section 2, it's the one that saves you from locking yourself out.

This version includes real gotchas hit during an actual setup (Ubuntu socket-activated SSH, multiple local keys, pasting multi-line commands, etc.) — marked with ⚠️ **Gotcha** callouts so you don't lose time on the same issues.

**Placeholders used throughout:**
- `newuser` — your new sudo username
- `2222` — your custom SSH port (pick any unused port 1024–65535)
- `yourdomain.com` — your actual domain (e.g. `admin.gymgrips.com.np`)

**General tip:** when running multiple commands, paste/run them **one at a time**, not as one multi-line block. Pasting several commands together into some terminals can cause them to be misread as arguments to the first command (this bit the Nginx symlink step below — see Section 4).

---

## 1. First Login & Base System

```bash
# SSH in as root using the password Contabo emailed you
ssh root@your_server_ip

# Update everything
apt update && apt upgrade -y

# Set hostname (optional but tidy)
hostnamectl set-hostname myserver

# Set timezone
timedatectl set-timezone Asia/Kathmandu

# Create a non-root user
adduser newuser
```

`adduser` will prompt for **Full Name, Room Number, Work Phone, Home Phone, Other**. These are just optional metadata (GECOS fields) — they don't affect how the server works. Fill in what you like or just press **Enter** through all of them, then confirm with `Y` when asked "Is the information correct?".

```bash
# Give it sudo rights
usermod -aG sudo newuser

# Switch to it and confirm sudo works
su - newuser
sudo whoami   # should print "root"
```

---

## 2. SSH Hardening

### 2a. Set up key-based auth (do this from your LOCAL machine)

If you already have multiple SSH keys on your machine (common if you use several — GitHub, other VPS boxes, etc.), **generate a new, dedicated key for this server** rather than reusing an existing one. It keeps access cleanly separated and easy to revoke individually later.

```bash
# On your local machine — pick a distinct filename for this server
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_contabo_vps -C "contabo-vps"

# Copy the public key to the server (note the -i pointing at the new key)
ssh-copy-id -i ~/.ssh/id_ed25519_contabo_vps.pub newuser@your_server_ip
```

⚠️ **Gotcha — multiple local keys cause a silent fallback to password.**
If you have several key pairs in `~/.ssh`, plain `ssh newuser@your_server_ip` (no `-i` flag) may offer the wrong key, hit the server's auth-attempt limit, and silently drop to a password prompt — even though `ssh-copy-id` succeeded. Always test explicitly with `-i` pointed at the exact key file you intended:

```bash
ssh -i ~/.ssh/id_ed25519_contabo_vps newuser@your_server_ip
```

⚠️ **Gotcha — double-check the filename.** If this still asks for a password, verify `authorized_keys` on the server (`cat ~/.ssh/authorized_keys`) actually contains the key you think it does — it's easy to accidentally copy a *different* existing key (e.g. a GitHub key) if you don't pass `-i` explicitly to `ssh-copy-id`. If the wrong key ended up there, edit `~/.ssh/authorized_keys` on the server and remove that line, then re-run `ssh-copy-id` with the correct `-i` flag.

Once logged in, set up a permanent alias so you don't need `-i` every time — add this to `~/.ssh/config` **on your local machine**:
```
Host contabo-vps
    HostName your_server_ip
    User newuser
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_contabo_vps
    IdentitiesOnly yes
```
(Port 2222 goes in once you've completed step 2b below.) After this, `ssh contabo-vps` just works.

### 2b. Harden sshd config (on the server, as newuser)

```bash
sudo nano /etc/ssh/sshd_config
```

Set/change these values — **make sure there's no `#` in front of the line**, or sshd will ignore it and silently keep its default instead:
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

⚠️ **Gotcha — commented-out lines are silently ignored.** It's easy to edit a line that already has a `#` (e.g. `#Port 2222`) and not notice it's still commented. Double check with:
```bash
sudo grep -iE '^Port|^PasswordAuthentication|^PermitRootLogin' /etc/ssh/sshd_config
```
Each should print with no `#` before it.

```bash
# Check the config for syntax errors before restarting
sudo sshd -t
```
Note: this is `sshd -t` (the daemon binary), **not** `ssh -t` (the client) — easy typo, and `ssh -t` will just print usage help instead of checking anything.

```bash
# Restart SSH — the systemd unit is named "ssh", not "sshd", on Ubuntu
sudo systemctl restart ssh
```

⚠️ **Gotcha — Ubuntu 22.04+/24.04 uses socket activation for SSH.** Even after setting `Port 2222` in `sshd_config` and restarting, you may find `ss -tlnp | grep ssh` still shows port 22 only. This is because `ssh.socket` (a separate systemd unit) controls the actual listening port independently of `sshd_config`. Check:
```bash
sudo systemctl status ssh.socket
```
If it's active, the simplest fix is to disable socket activation entirely and let `ssh.service` manage its own port directly (this matches the `Port 2222` setting above):
```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh.service
```
Then confirm:
```bash
sudo ss -tlnp | grep ssh
```
It should now show `0.0.0.0:2222` (and `[::]:2222`) instead of port 22.

### ⚠️ Critical step — do not skip

**Before closing your current terminal**, open a **new** terminal window and test the new connection:

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519_contabo_vps newuser@your_server_ip
# or, once your local ~/.ssh/config alias is set up with the port:
ssh contabo-vps
```

Only close your original session once the new one connects successfully **with no password prompt**. If it fails, fix it using your still-open original session — don't disconnect first. If you get "**Connection refused**" specifically, that means nothing is listening on that port yet (see the socket-activation gotcha above) — it's a different problem from an auth failure, so check `ss -tlnp` before touching keys/passwords again.

### 2c. Install fail2ban

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban

# Optional: create a local jail override so updates don't wipe your config
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### 2d. Back up your private key

Once key login is confirmed working and password auth is disabled, **the private key file is the only way in** over SSH (the other fallback is Contabo's rescue/VNC console in their control panel, which bypasses SSH entirely — worth knowing where that is before you need it).

- **Never** back up the *private* key (no `.pub` extension) as a bare, unencrypted file in cloud storage (Drive, Dropbox, email, etc.) — that's how keys leak.
- The *public* key (`.pub`) is safe to store/share anywhere; it's already sitting in the server's `authorized_keys`.
- Good options for the private key: a password manager with file attachments (1Password, Bitwarden), an encrypted external drive, or a password-protected zip if using cloud storage (`zip -e keybackup.zip ~/.ssh/id_ed25519_contabo_vps`).
- Best of all: add a passphrase to the key itself, so even the raw file is useless without it:
  ```bash
  ssh-keygen -p -f ~/.ssh/id_ed25519_contabo_vps
  ```
- Consider adding a **second** key from a backup device now (while you have easy access), so losing one device doesn't lock you out. Any number of public keys can live in `authorized_keys`, one per line.

---

## 3. Firewall (UFW)

```bash
sudo apt install ufw -y

# Default policy: deny in, allow out
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow your NEW ssh port — do this before enabling, or you'll lock yourself out
sudo ufw allow 2222/tcp

# Allow web traffic
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable
sudo ufw enable

# Verify
sudo ufw status verbose
```

Expected output looks like this (port 22 should NOT appear — only 2222):
```
Status: active
Default: deny (incoming), allow (outgoing)

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
2222/tcp (v6)              ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)                ALLOW IN    Anywhere (v6)
```

If Contabo's control panel offers a separate cloud/network-level firewall, mirror these same rules there as a second layer.

---

## 4. Web Server — Nginx

Nginx is the recommended default here: lighter footprint and cleaner as a reverse proxy in front of Docker containers or app servers.

```bash
sudo apt install nginx -y
sudo systemctl enable --now nginx

# Allow it through ufw (if not already covered by 80/443 above)
sudo ufw allow 'Nginx Full'

# Test
curl http://localhost
```

### Multiple subdomains

If you're running more than one app (e.g. an admin panel and an API on separate subdomains), give each its own server block and its own file — don't combine them.

`/etc/nginx/sites-available/admin.yourdomain.com`:
```nginx
server {
    listen 80;
    server_name admin.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;  # match your app's actual local port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

`/etc/nginx/sites-available/api.yourdomain.com`:
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:4000;  # match your app's actual local port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

⚠️ **Gotcha — run symlink commands one at a time, not pasted as a block.** Pasting two `sudo ln -s ...` commands together (even separated by a line break, depending on your terminal) can cause the shell to misread the second command as extra arguments to the first — creating bogus symlinks (e.g. literally named `ln`, `sudo`, or a self-referential `sites-enabled` link pointing to itself). This produces an error like:
```
nginx: [emerg] open() "/etc/nginx/sites-enabled/ln" failed (40: Too many levels of symbolic links)
```
Fix: list the directory, remove any junk entries, and re-create the correct symlinks one command at a time:
```bash
ls -la /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/ln          # remove any stray/bad symlinks by name
sudo rm /etc/nginx/sites-enabled/sudo
sudo rm /etc/nginx/sites-enabled/sites-enabled
```
Then, one command per line, waiting for the prompt between each:
```bash
sudo ln -s /etc/nginx/sites-available/admin.yourdomain.com /etc/nginx/sites-enabled/admin.yourdomain.com
```
```bash
sudo ln -s /etc/nginx/sites-available/api.yourdomain.com /etc/nginx/sites-enabled/api.yourdomain.com
```
Confirm cleanly with `ls -la /etc/nginx/sites-enabled/` — you should see only your intended site files (plus `default`, which you can remove later once your real sites are confirmed working).

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 5. SSL/TLS — Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Before running Certbot:
- **DNS must have already propagated.** Confirm with `dig admin.yourdomain.com +short` and `dig api.yourdomain.com +short` — both should return your server's IP.
- **If using Cloudflare and a subdomain is "Proxied" (orange cloud)**, temporarily switch it to "DNS only" (grey cloud) before running Certbot — Cloudflare's proxy in front of your server can interfere with the validation step. Switch it back to Proxied afterward once the cert is issued. Subdomains already set to "DNS only" need no change.

One cert covering multiple subdomains:
```bash
sudo certbot --nginx -d admin.yourdomain.com -d api.yourdomain.com
```
Or separate certs per subdomain (easier to manage/renew independently):
```bash
sudo certbot --nginx -d admin.yourdomain.com
sudo certbot --nginx -d api.yourdomain.com
```

Certbot will ask for an email (for renewal notices) and whether to redirect HTTP to HTTPS — say yes.

Verify auto-renewal:
```bash
sudo certbot renew --dry-run
```

---

## 6. Docker Setup

```bash
# Remove any old versions first
sudo apt remove docker docker-engine docker.io containerd runc

# Install prerequisites
sudo apt install ca-certificates curl gnupg -y

# Add Docker's official GPG key and repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Let your user run docker without sudo
sudo usermod -aG docker newuser
```

⚠️ **Gotcha — group changes don't apply to your current session.** Running `usermod -aG docker newuser` doesn't retroactively grant the new group to your already-open terminal — running `docker run hello-world` right after will fail with "permission denied while trying to connect to the docker API". Fully log out and back in:
```bash
exit
ssh contabo-vps
groups   # confirm "docker" is listed
```
Then it'll work without `sudo`:
```bash
docker run hello-world
docker compose version
```

### ⚠️ Docker + UFW gotcha

Docker edits `iptables` directly and can **expose container ports to the internet even when UFW shows them closed**. This only becomes relevant once you're actually deploying containers with published ports — keep it in mind for later:

**Option A (recommended):** Bind container ports to localhost only, and let Nginx proxy to them:
```yaml
# docker-compose.yml
services:
  your-app:
    build: .
    ports:
      - "127.0.0.1:3000:3000"
    restart: unless-stopped
```
Make sure the port here matches the `proxy_pass` target in the corresponding Nginx config. Verify after starting the container:
```bash
sudo ss -tlnp | grep 3000
```
Should show `127.0.0.1:3000`, not `0.0.0.0:3000`.

**Option B:** Configure Docker to respect UFW rules by editing `/etc/docker/daemon.json`:
```json
{
  "iptables": false
}
```
(Then manage all container exposure through UFW/nginx manually — more advanced, only do this if you understand the tradeoffs.)

---

## 7. Swap File

A swap file is overflow space on disk the kernel uses when physical RAM fills up. It's much slower than real RAM, but prevents a low-memory situation from crashing processes outright — e.g. during a memory-heavy Docker build or several containers running at once, the OS can push inactive data to disk temporarily instead of killing something.

More important on smaller RAM plans (≤4GB); less critical if you have 8GB+ and low current usage, but still cheap insurance.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make it permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify
free -h
```
`Swap: 0B used` is the expected/good state right after setup — it only gets used under actual memory pressure.

---

## 8. Automatic Security Updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 9. Backups

- Enable Contabo snapshot/backup add-on in their control panel, **or**
- Set up your own routine (e.g. `restic` or `rsync` to off-server storage) for anything not reproducible from a snapshot

---

## 10. Basic Monitoring

```bash
# Quick disk-space check tool
sudo apt install ncdu -y

# Simple cron job example: alert-ish via log if disk >85% full
# (add to crontab -e)
0 * * * * df -h / | awk 'NR==2{print $5}' | sed 's/%//' | awk '{if ($1 > 85) print "Disk usage high: " $1 "%"}' >> /var/log/disk-alert.log
```

For anything more serious, consider a lightweight external service (e.g. UptimeRobot for uptime pings) rather than self-hosting a full monitoring stack on a small VPS.

---

## Post-Setup Checklist

- [ ] Logged in as `newuser`, not root
- [ ] Dedicated SSH key generated for this server, `ssh-copy-id` used with explicit `-i`
- [ ] SSH key auth confirmed with explicit `-i` test, password auth disabled
- [ ] SSH port changed to 2222, `ssh.socket` disabled (if applicable), confirmed listening with `ss -tlnp`
- [ ] Verified new connection in a separate session before closing the original
- [ ] Private key backed up (encrypted/password-protected), passphrase optionally added
- [ ] `fail2ban` active
- [ ] UFW enabled, only 2222/80/443 open (port 22 NOT listed)
- [ ] Nginx installed, one server block per subdomain, symlinked into `sites-enabled`
- [ ] DNS propagated and confirmed via `dig` before running Certbot
- [ ] Cloudflare proxy toggled to "DNS only" temporarily for Certbot if applicable
- [ ] SSL certificate issued and auto-renewal tested
- [ ] Docker installed, user in `docker` group (confirmed via `groups` after re-login)
- [ ] Container ports bound to `127.0.0.1` only, proxied through Nginx
- [ ] Swap configured
- [ ] Unattended upgrades enabled
- [ ] Backup method decided and enabled
- [ ] Basic monitoring in place

---

*Keep this doc updated as your setup evolves — especially your list of open ports, subdomains, and any new services you add behind Nginx.*
