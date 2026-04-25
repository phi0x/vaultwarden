# vaultwarden

A reference solution for deploying [vaultwarden](https://github.com/dani-garcia/vaultwarden) (a Bitwarden-compatible server in Rust) on Ubuntu/Debian behind Caddy, with automated SQLite backups and offsite sync over SSH.

> **Note on the rename:** This project was previously called `bitwarden_rs` and the upstream Docker image was `bitwardenrs/server`. In April 2021 the project was renamed to `vaultwarden` at the request of Bitwarden Inc. to avoid trademark confusion, and the canonical Docker image is now `vaultwarden/server`. This guide has been updated accordingly. If you are migrating an existing `bitwarden_rs` install, the data directory format is identical — you only need to switch the image and container name.

Originally adapted from the [Linode self-hosting guide](https://www.linode.com/docs/guides/how-to-self-host-the-bitwarden-rs-password-manager/), then modernized for current Docker, current Caddy, and the post-rename project.

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

## 2. Deploy vaultwarden

2.1 Pull the vaultwarden image:

```
sudo docker pull vaultwarden/server:latest
```

2.2 Create the data directory with strict permissions:

```
sudo mkdir /srv/vaultwarden
sudo chmod go-rwx /srv/vaultwarden
```

2.3 Generate an admin token. As of recent vaultwarden versions, plain string tokens log a deprecation warning — generate an Argon2 PHC hash instead:

```
sudo docker run --rm -it vaultwarden/server /vaultwarden hash
```

You'll be prompted to enter a passphrase twice. Copy the resulting `$argon2id$...` string — that's your `ADMIN_TOKEN`.

2.4 Run the vaultwarden container. Compared to the old `bitwarden_rs` setup, note that **`WEBSOCKET_ENABLED` is no longer required and the separate `:3012` port has been retired** — vaultwarden serves websockets on the main HTTP port via `/notifications/hub`:

```
docker run -d --name vaultwarden \
  -v /srv/vaultwarden:/data \
  -e SIGNUPS_ALLOWED=false \
  -e ADMIN_TOKEN='$argon2id$v=19$...YOUR_HASH_HERE...' \
  -e DOMAIN=https://YOUR_DOMAIN_HERE \
  -p 127.0.0.1:8080:80 \
  --restart on-failure \
  vaultwarden/server:latest
```

The container binds to `127.0.0.1` only — Caddy (next section) is the only thing that should reach it.

---

## 3. Set up Caddy as a reverse proxy

3.1 Pull Caddy. The canonical image is `caddy:alpine` (the old `caddy/caddy` namespace is deprecated):

```
sudo docker pull caddy:alpine
```

3.2 Create the Caddy config:

```
sudo nano /etc/Caddyfile
```

```
YOUR_DOMAIN_HERE {
  encode gzip
  reverse_proxy 127.0.0.1:8080
}
```

The old config split traffic between `:8080` and `:3012` for websockets — that's no longer necessary in modern vaultwarden. A single `reverse_proxy` covers everything.

3.3 Create the Caddy state directory and run the container:

```
sudo mkdir /etc/caddy
sudo chmod go-rwx /etc/caddy

sudo docker run -d --name caddy \
  -v /etc/Caddyfile:/etc/caddy/Caddyfile \
  -v /etc/caddy:/root/.local/share/caddy \
  --net host \
  --restart on-failure \
  caddy:alpine
```

3.4 Tail the logs to watch the cert provisioning:

```
sudo docker logs -f caddy
```

If you see ACME errors, common causes are: DNS not pointing at this host yet, ports 80/443 not open in your firewall, or rate limits from prior failed attempts. Stop and start Caddy with `sudo docker stop caddy && sudo docker start caddy` after fixing.

3.5 Visit `https://YOUR_DOMAIN_HERE` — you should see the vaultwarden login screen.

---

## 4. Configure admin settings

Visit `https://YOUR_DOMAIN_HERE/admin` and log in with the admin passphrase from section 2.3 (the plain passphrase, not the Argon2 hash).

Recommended settings:

- **General Settings** → set **Domain URL** to `https://YOUR_DOMAIN_HERE`, set an invitation org name.
- **SMTP Email Settings** → enable, point at your provider. For Gmail, use `smtp.gmail.com:587` with TLS and an [App Password](https://myaccount.google.com/apppasswords) (your normal Google password will not work).
- **Email 2FA Settings** → enable.
- Click **Save**, then use **Test SMTP** to verify a delivery.

> The SMTP password is stored in plaintext in `/srv/vaultwarden/config.json`. Restrict access to the data directory accordingly.

---

## 5. Set up offsite backups over SSH

This section uses **SSH key authentication** — no plaintext passwords stored in systemd units. The original guide used `sshpass` with a hardcoded password; that approach was removed because (a) the password sat world-readable in `/etc/systemd/system/`, and (b) the unit silently failed forever the first time the remote host's keys changed (e.g. after a server rebuild) with no notification.

5.1 Install sqlite3 on the vaultwarden host:

```
sudo apt-get install sqlite3
```

5.2 Create the local backup directory:

```
sudo mkdir /srv/backup
sudo chmod go-rwx /srv/backup
```

5.3 Generate an SSH key for `root` (the user the systemd unit runs as) if one doesn't exist:

```
sudo ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ''
```

5.4 Authorize the key on your offsite/backup server. Replace `BACKUP_PORT`, `BACKUP_USER`, and `BACKUP_HOST` with your values:

```
sudo ssh-copy-id -p BACKUP_PORT -i /root/.ssh/id_ed25519.pub BACKUP_USER@BACKUP_HOST
```

5.5 Seed `known_hosts` with the offsite server's keys. **Verify the fingerprint out-of-band** (e.g. by running `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` directly on the backup server) before trusting it:

```
sudo ssh-keyscan -p BACKUP_PORT BACKUP_HOST | sudo tee -a /root/.ssh/known_hosts
sudo ssh-keygen -lf /root/.ssh/known_hosts | grep BACKUP_HOST
```

5.6 Test passwordless SSH:

```
sudo ssh -p BACKUP_PORT -o BatchMode=yes BACKUP_USER@BACKUP_HOST 'echo OK'
```

If you see `OK`, you're done. If you see `Permission denied (publickey)`, repeat 5.4. If you see `Host key verification failed`, repeat 5.5.

5.7 Create the backup service:

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

Replace `BACKUP_PORT`, `BACKUP_USER`, `BACKUP_HOST`, and `BACKUP_REMOTE_PATH` with your values. The `OnFailure=` directive is described in section 5.10 — you can omit it if you don't want notifications.

5.8 Run the unit once and verify a backup file exists:

```
sudo systemctl start vaultwarden-backup.service
sudo ls -l /srv/backup/
```

You should see something like:

```
-rw-r--r-- 1 root root 139264 Apr 25 18:16 backup-2026-04-25T18_16_50+00_00.sq3
```

5.9 Schedule daily backups via a systemd timer:

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

5.10 (Optional but strongly recommended) Add a failure-notification handler. The `OnFailure=` line in the service unit triggers a templated unit when the backup fails. Create it once:

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

## 6. Restoring from backup

vaultwarden uses SQLite's online backup format, so restore is a file copy:

```
# Stop the container so nothing is writing to the database
sudo docker stop vaultwarden

# Replace the live DB with the backup snapshot
sudo cp /srv/vaultwarden/db.sqlite3 /srv/vaultwarden/db.sqlite3.broken
sudo cp /srv/backup/backup-YYYY-MM-DDT....sq3 /srv/vaultwarden/db.sqlite3
sudo chown root:root /srv/vaultwarden/db.sqlite3

# Start vaultwarden
sudo docker start vaultwarden
```

The data directory `/srv/vaultwarden` also contains `attachments/`, `sends/`, `rsa_key.*`, and `config.json` — for a full disaster-recovery copy, archive the entire directory. The backup unit above only captures the SQLite database; consider extending it with a `tar` of the data directory if attachments matter to you.

---

## 7. Harden SSH on the vaultwarden host

7.1 Move SSH off port 22:

```
sudo nano /etc/ssh/sshd_config
```

(Note: the path is `/etc/ssh/sshd_config`, not `/etc/sshd/sshd_config` — the original guide had this wrong.)

Find `#Port 22` and change it to `Port 22022`.

7.2 Restart sshd:

```
sudo systemctl restart ssh
```

7.3 Reconnect on the new port from a second terminal **before closing your current session** — if the new port doesn't work, you'll need the old session to fix it.

```
ssh -p 22022 user@your-vaultwarden-host
```

> **Important:** the `BACKUP_PORT` from section 5 is the SSH port on your **backup target** server, which is unrelated to this host's SSH port. Don't confuse them.

---

## 8. Firewall (iptables)

8.1 Install iptables-persistent:

```
sudo apt-get install iptables iptables-persistent -y
```

8.2 Apply the rules. Note that we **do not open port 8080**: the vaultwarden container binds to `127.0.0.1:8080` only, so it's never reachable from outside regardless of firewall rules. Caddy (host network mode) handles 80/443.

```
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p udp -m multiport --dports 53 -j ACCEPT
sudo iptables -A INPUT -p tcp -m multiport --dports 22022,80,443 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables-save | sudo tee /etc/iptables/rules.v4 > /dev/null
```

8.3 Reboot to verify everything comes up cleanly:

```
sudo reboot
```

8.4 Validate rules after reboot:

```
sudo iptables -L -v -n
```

If you'd rather use `ufw`, the equivalent is:

```
sudo ufw default deny incoming
sudo ufw allow 22022/tcp
sudo ufw allow 80,443/tcp
sudo ufw enable
```

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

**Caddy can't reach vaultwarden.**
The container binds to `127.0.0.1:8080`. Caddy must run with `--net host` (as in section 3.3) to reach it. If you run Caddy in a bridge network instead, swap to a docker network shared with the vaultwarden container and reference it by container name.
