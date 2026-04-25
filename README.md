# vaultwarden

A reference solution for deploying [vaultwarden](https://github.com/dani-garcia/vaultwarden) (a Bitwarden-compatible server in Rust) on Ubuntu/Debian behind Caddy, with automated SQLite backups and offsite sync over SSH.

> **Note on the rename:** This project was previously called `bitwarden_rs` and the upstream Docker image was `bitwardenrs/server`. In April 2021 it was renamed to `vaultwarden` at the request of Bitwarden Inc. to avoid trademark confusion, and the canonical Docker image is now `vaultwarden/server`. This guide has been updated accordingly.

Originally adapted from the [Linode self-hosting guide](https://www.linode.com/docs/guides/how-to-self-host-the-bitwarden-rs-password-manager/), then modernized for current Docker, Docker Compose, current Caddy, and the post-rename project.

## Repo layout

| File | Purpose |
|---|---|
| [`compose.yaml`](compose.yaml) | The vaultwarden + Caddy stack. Modern Compose Spec (no `version:` key). |
| [`Caddyfile.example`](Caddyfile.example) | Diagnostics-tuned Caddy config for vaultwarden. Copy to `Caddyfile` and edit. |
| [`.env.example`](.env.example) | Template for the admin token and domain. Copy to `.env` and edit (gitignored). |
| [`.gitignore`](.gitignore) | Excludes `.env`, `Caddyfile`, and local data so secrets never get committed. |

---

## 1. Install Docker

1.1 Remove any pre-existing docker packages:

```
sudo apt-get remove docker docker-engine docker.io containerd runc
```

1.2 Install prerequisites:

```
sudo apt-get install ca-certificates curl gnupg
```

1.3 Add Docker's official GPG key (the modern `signed-by` keyring approach — `apt-key` is deprecated and removed in current Ubuntu/Debian):

```
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

1.4 Add the Docker upstream repository (architecture is detected automatically — works on amd64 and arm64):

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

1.5 Update apt and install Docker:

```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

1.6 Enable and start Docker, then confirm it works:

```
sudo systemctl enable --now docker
sudo docker ps
```

---

## 2. Deploy vaultwarden + Caddy with Docker Compose

This is the recommended path. It puts vaultwarden and Caddy on a private bridge network, runs Caddy as the only externally-exposed service, and reads the admin token from a gitignored `.env` file. (Prefer to do it the manual `docker run` way? See [Appendix A](#appendix-a-manual-deployment-without-docker-compose).)

2.1 **Clone this repo somewhere persistent**, e.g. `/opt/vaultwarden-stack`:

```
sudo git clone https://github.com/phi0x/vaultwarden /opt/vaultwarden-stack
cd /opt/vaultwarden-stack
```

2.2 **Create the data directory** vaultwarden will bind-mount:

```
sudo mkdir -p /srv/vaultwarden
sudo chmod go-rwx /srv/vaultwarden
```

2.3 **Generate an Argon2 admin token.** Recent vaultwarden versions reject plain-string tokens with a deprecation warning — use the built-in hasher:

```
sudo docker run --rm -it vaultwarden/server /vaultwarden hash
```

You'll be prompted for a passphrase twice. The plain passphrase is what you type into `/admin`; the resulting `$argon2id$...` string is what goes in `.env`.

2.4 **Configure `.env`:**

```
sudo cp .env.example .env
sudo nano .env
```

Paste the Argon2 hash into `VAULTWARDEN_ADMIN_TOKEN` and set `DOMAIN=https://YOUR_DOMAIN_HERE`.

2.5 **Configure the Caddyfile:**

```
sudo cp Caddyfile.example Caddyfile
sudo nano Caddyfile
```

Replace `YOUR_DOMAIN_HERE` with the same hostname as in `.env`.

> **Why the Caddyfile is more than `reverse_proxy vaultwarden:80`** — see the comments at the top of `Caddyfile.example`. The short version: it's tuned to make vaultwarden's built-in diagnostics page (`/admin/diagnostics`) report green. Specifically it injects `X-Real-IP` / `X-Forwarded-For` so vaultwarden logs and rate-limits per real client (not per Caddy), it strips frame-blocking headers only on the 2FA endpoints so FIDO2 popups can render, and its CSP allow-lists `api.pwnedpasswords.com` (breach checks) and `api.2fa.directory` (2FA service catalog) — both of which the web vault calls into. There's also a `*.map` 404 rule that keeps browser DevTools quiet without polluting your access log.

2.6 **Bring up the stack:**

```
sudo docker compose up -d
sudo docker compose logs -f caddy
```

The first time Caddy starts, it'll obtain an HTTPS cert via ACME — watch the logs for success. Press `Ctrl+C` once you see the cert is provisioned (the containers keep running).

2.7 **Verify:** Visit `https://YOUR_DOMAIN_HERE` — you should see the vaultwarden login screen. Then check `https://YOUR_DOMAIN_HERE/admin` (log in with the **plain passphrase** from step 2.3, not the Argon2 hash) and look at the **Diagnostics** tab. Everything should be green.

> **Migrating from an older `bitwarden_rs` install?** The data format is identical. Stop the old container, copy the data directory to `/srv/vaultwarden`, and `docker compose up -d`. You may see a one-time admin-token warning if your old token is a plain string — generate an Argon2 hash per 2.3 and update `.env`. The old `WEBSOCKET_ENABLED=true` env var is no longer required (the dedicated `:3012` port has been retired) and can be dropped from any custom unit/compose files.

---

## 3. Configure admin settings

Visit `https://YOUR_DOMAIN_HERE/admin` and log in with the admin passphrase from section 2.3 (the plain passphrase, not the Argon2 hash).

Recommended settings:

- **General Settings** → set **Domain URL** to `https://YOUR_DOMAIN_HERE`, set an invitation org name.
- **SMTP Email Settings** → enable, point at your provider. For Gmail, use `smtp.gmail.com:587` with TLS and an [App Password](https://myaccount.google.com/apppasswords) (your normal Google password will not work).
- **Email 2FA Settings** → enable.
- Click **Save**, then use **Test SMTP** to verify a delivery.
- **Diagnostics** → all rows green. If `Reverse Proxy IP` reports the Caddy container's IP instead of your real client IP, the `X-Real-IP` header pass-through in Caddyfile isn't working — re-check section 2.5.

> The SMTP password is stored in plaintext in `/srv/vaultwarden/config.json`. Restrict access to the data directory accordingly.

---

## 4. Set up offsite backups over SSH

This section uses **SSH key authentication** — no plaintext passwords stored in systemd units. The original guide used `sshpass` with a hardcoded password; that approach was removed because (a) the password sat world-readable in `/etc/systemd/system/`, and (b) the unit silently failed forever the first time the remote host's keys changed (e.g. after a server rebuild) with no notification.

4.1 Install sqlite3 on the vaultwarden host:

```
sudo apt-get install sqlite3
```

4.2 Create the local backup directory:

```
sudo mkdir /srv/backup
sudo chmod go-rwx /srv/backup
```

4.3 Generate an SSH key for `root` (the user the systemd unit runs as) if one doesn't exist:

```
sudo ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ''
```

4.4 Authorize the key on your offsite/backup server. Replace `BACKUP_PORT`, `BACKUP_USER`, and `BACKUP_HOST` with your values:

```
sudo ssh-copy-id -p BACKUP_PORT -i /root/.ssh/id_ed25519.pub BACKUP_USER@BACKUP_HOST
```

4.5 Seed `known_hosts` with the offsite server's keys. **Verify the fingerprint out-of-band** (e.g. by running `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` directly on the backup server) before trusting it:

```
sudo ssh-keyscan -p BACKUP_PORT BACKUP_HOST | sudo tee -a /root/.ssh/known_hosts
sudo ssh-keygen -lf /root/.ssh/known_hosts | grep BACKUP_HOST
```

4.6 Test passwordless SSH:

```
sudo ssh -p BACKUP_PORT -o BatchMode=yes BACKUP_USER@BACKUP_HOST 'echo OK'
```

If you see `OK`, you're done. If you see `Permission denied (publickey)`, repeat 4.4. If you see `Host key verification failed`, repeat 4.5.

4.7 Create the backup service:

```
sudo nano /etc/systemd/system/vaultwarden-backup.service
```

```
[Unit]
Description=Backup the vaultwarden sqlite database
OnFailure=vaultwarden-backup-failure@%n.service

[Service]
Type=oneshot
WorkingDirectory=/srv/backup

# 1. Snapshot the live database (safe to run while vaultwarden is up)
ExecStart=/usr/bin/env sh -c 'sqlite3 /srv/vaultwarden/db.sqlite3 ".backup backup-$(date -Is | tr : _).sq3"'

# 2. Prune local backups older than 30 days
ExecStart=/usr/bin/find . -type f -mtime +30 -name 'backup*' -delete

# 3. Sync to offsite host (remove this line if you don't want offsite backups)
ExecStart=/usr/bin/scp -P BACKUP_PORT -r /srv/backup/ BACKUP_USER@BACKUP_HOST:BACKUP_REMOTE_PATH
```

Replace `BACKUP_PORT`, `BACKUP_USER`, `BACKUP_HOST`, and `BACKUP_REMOTE_PATH` with your values. The `OnFailure=` directive is described in section 4.10 — you can omit it if you don't want notifications.

4.8 Run the unit once and verify a backup file exists:

```
sudo systemctl start vaultwarden-backup.service
sudo ls -l /srv/backup/
```

You should see something like:

```
-rw-r--r-- 1 root root 139264 Apr 25 18:16 backup-2026-04-25T18_16_50+00_00.sq3
```

4.9 Schedule daily backups via a systemd timer:

```
sudo nano /etc/systemd/system/vaultwarden-backup.timer
```

```
[Unit]
Description=Schedule vaultwarden backups

[Timer]
OnCalendar=04:00
Persistent=true

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable --now vaultwarden-backup.timer
sudo systemctl list-timers vaultwarden-backup.timer
```

4.10 (Optional but strongly recommended) Add a failure-notification handler. The `OnFailure=` line in the service unit triggers a templated unit when the backup fails. Create it once:

```
sudo nano /etc/systemd/system/vaultwarden-backup-failure@.service
```

```
[Unit]
Description=Notify on failure of %i

[Service]
Type=oneshot
ExecStart=/usr/bin/curl -sS -d "vaultwarden backup failed on $(hostname)" https://ntfy.sh/YOUR_PRIVATE_TOPIC
```

Swap in any notification mechanism you prefer (ntfy, Discord webhook, email via `mail`, Slack, etc.). Without this, a broken backup is invisible until you happen to check `systemctl status` — which is exactly how the original guide's setup tended to silently rot.

---

## 5. Restoring from backup

vaultwarden uses SQLite's online backup format, so restore is a file copy:

```
# Stop the container so nothing is writing to the database
sudo docker compose stop vaultwarden

# Replace the live DB with the backup snapshot
sudo cp /srv/vaultwarden/db.sqlite3 /srv/vaultwarden/db.sqlite3.broken
sudo cp /srv/backup/backup-YYYY-MM-DDT....sq3 /srv/vaultwarden/db.sqlite3
sudo chown root:root /srv/vaultwarden/db.sqlite3

# Start vaultwarden
sudo docker compose start vaultwarden
```

The data directory `/srv/vaultwarden` also contains `attachments/`, `sends/`, `rsa_key.*`, and `config.json` — for a full disaster-recovery copy, archive the entire directory. The backup unit above only captures the SQLite database; consider extending it with a `tar` of the data directory if attachments matter to you.

---

## 6. Harden SSH on the vaultwarden host

6.1 Move SSH off port 22:

```
sudo nano /etc/ssh/sshd_config
```

(Note: the path is `/etc/ssh/sshd_config`, not `/etc/sshd/sshd_config` — the original guide had this wrong.)

Find `#Port 22` and change it to `Port 22022`.

6.2 Restart sshd:

```
sudo systemctl restart ssh
```

6.3 Reconnect on the new port from a second terminal **before closing your current session** — if the new port doesn't work, you'll need the old session to fix it.

```
ssh -p 22022 user@your-vaultwarden-host
```

> **Important:** the `BACKUP_PORT` from section 4 is the SSH port on your **backup target** server, which is unrelated to this host's SSH port. Don't confuse them.

---

## 7. Firewall (iptables)

7.1 Install iptables-persistent:

```
sudo apt-get install iptables iptables-persistent -y
```

7.2 Apply the rules. Note that we **do not open port 8080**: the vaultwarden container has no externally-published ports under the Compose setup, so it's never reachable from outside regardless of firewall rules. Caddy publishes 80, 443/tcp, and 443/udp (HTTP/3).

```
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p udp -m multiport --dports 53,443 -j ACCEPT
sudo iptables -A INPUT -p tcp -m multiport --dports 22022,80,443 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables-save | sudo tee /etc/iptables/rules.v4 > /dev/null
```

7.3 Reboot to verify everything comes up cleanly:

```
sudo reboot
```

7.4 Validate rules after reboot:

```
sudo iptables -L -v -n
```

If you'd rather use `ufw`, the equivalent is:

```
sudo ufw default deny incoming
sudo ufw allow 22022/tcp
sudo ufw allow 80,443/tcp
sudo ufw allow 443/udp
sudo ufw enable
```

---

## Appendix A: Manual deployment without Docker Compose

Use this only if you specifically don't want Compose — it's the same software, just with the orchestration done by hand.

**A.1 Pull and run vaultwarden**

```
sudo docker pull vaultwarden/server:latest

sudo mkdir /srv/vaultwarden
sudo chmod go-rwx /srv/vaultwarden

# Generate Argon2 admin token (see section 2.3)
sudo docker run --rm -it vaultwarden/server /vaultwarden hash

sudo docker run -d --name vaultwarden \
  -v /srv/vaultwarden:/data \
  -e SIGNUPS_ALLOWED=false \
  -e ADMIN_TOKEN='$argon2id$v=19$...YOUR_HASH_HERE...' \
  -e DOMAIN=https://YOUR_DOMAIN_HERE \
  -e IP_HEADER=X-Real-IP \
  -p 127.0.0.1:8080:80 \
  --restart unless-stopped \
  vaultwarden/server:latest
```

(The container binds to `127.0.0.1` only — Caddy reaches it over loopback.)

**A.2 Run Caddy with host networking**

```
sudo mkdir /etc/caddy
sudo cp Caddyfile.example /etc/Caddyfile
sudo nano /etc/Caddyfile
# Replace `vaultwarden:80` with `127.0.0.1:8080` everywhere — there's no
# Docker network in this setup, so Caddy reaches vaultwarden via loopback.
# Replace YOUR_DOMAIN_HERE with your hostname.

sudo docker run -d --name caddy \
  -v /etc/Caddyfile:/etc/caddy/Caddyfile:ro \
  -v /etc/caddy:/data \
  --net host \
  --restart unless-stopped \
  caddy:alpine
```

The rest of the guide (sections 3–7) applies identically.

---

## Troubleshooting

**Backup unit fails with `Host key verification failed` after rebuilding the backup server.**
The remote host's SSH keys regenerated and no longer match `/root/.ssh/known_hosts`. Refresh:

```
sudo ssh-keygen -f /root/.ssh/known_hosts -R '[BACKUP_HOST]:BACKUP_PORT'
sudo ssh-keyscan -p BACKUP_PORT BACKUP_HOST | sudo tee -a /root/.ssh/known_hosts
# Verify the new fingerprint matches what's on the backup server before trusting it
sudo systemctl start vaultwarden-backup.service
```

**`vaultwarden` container exits immediately with admin-token warnings.**
Recent versions reject plain-string tokens. Generate an Argon2 hash per section 2.3 and pass that as `ADMIN_TOKEN`.

**Caddy can't reach vaultwarden (`dial tcp: lookup vaultwarden: no such host`).**
The `vaultwarden:80` reference in `Caddyfile` resolves only when both containers are on the same Docker network. With `compose.yaml` they share `vaultwarden_net` automatically. If you split them across separate compose projects, attach Caddy to vaultwarden's network with `external: true`.

**`/admin/diagnostics` shows `Reverse Proxy IP: 172.x.x.x` instead of my real IP.**
The `X-Real-IP` header isn't reaching vaultwarden. Confirm `Caddyfile` includes `header_up X-Real-IP {http.request.remote.host}` (it does in the example), and that `IP_HEADER: X-Real-IP` is set in the vaultwarden environment (it is in `compose.yaml`).
