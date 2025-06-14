# 🧠 Deploying n8n on a Self-Hosted Server with Domain, HTTPS, and Backups

A complete guide to deploying `n8n` on a VPS with Ubuntu, custom domain, automatic SSL, and daily backups. Everything you need for a production setup.

---

## 🛠️ What You'll Get:

- A self-hosted `n8n` instance accessible at `https://n8n.your-domain.com` (you can choose another subdomain, but this one is used in the examples)
- HTTPS via Let's Encrypt (auto-renewal enabled)
- nginx reverse proxy
- Daily automatic backups
- Enabled nginx buffering for improved file processing in workflows

---

## ⚙️ Requirements:

- A VPS with Ubuntu 20+ (2 GB RAM recommended)
- I usually rent servers from [VDSina.com — referral link with a 10% top-up bonus](https://www.vdsina.com/?partner=1r8tcykewa) or [Firstbyte](https://firstbyte.ru/?from=196382)
- Docker installed
- A registered domain with DNS management (e.g., via reg.ru)

---

## 🔧 1. DNS Setup

Add an A record in your domain's DNS panel:

```
n8n     A     <Your Server's Public IP>
```

---

## 🐳 2. Running n8n via Docker

For SSH access I prefer the [Termius](https://termius.com) app.
Make sure Docker is installed on the server: [official instructions](https://docs.docker.com/engine/install/ubuntu/)

Create a directory to persist data:

```bash
mkdir -p ~/n8n_data
mkdir -p ~/n8n_files
```

Create a network for postgres connection

```bash
docker network create n8n-net
```

Then run the container:

```bash
docker run -d --restart unless-stopped \
  --name n8n \
  --network n8n-net \
  -p 5678:5678 \
  -v ~/n8n_data:/home/node/.n8n \
  -v ~/n8n_files:/files \
  -e N8N_HOST="n8n.your-domain.com" \
  -e WEBHOOK_TUNNEL_URL="https://n8n.your-domain.com/" \
  -e WEBHOOK_URL="https://n8n.your-domain.com/" \
  -e N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true \
  n8nio/n8n
```

**Important**: Let's make right permissions in created folders:

```bash
sudo chown -R 1000:1000 ~/n8n_data
sudo chmod -R u+rwX ~/n8n_data
sudo chown -R 1000:1000 ~/n8n_files
sudo chmod -R u+rwX ~/n8n_files
```

---

## 🌐 3. Setting up nginx and SSL

Install nginx and certbot:

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

Create a configuration file:

```bash
sudo nano /etc/nginx/sites-available/n8n.conf
```

Paste the following initial configuration **without SSL**:

```nginx
server {
    listen 80;
    server_name n8n.your-domain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;

        proxy_buffering on;
        proxy_buffers 8 256k;
        proxy_buffer_size 128k;
        proxy_busy_buffers_size 512k;
        proxy_max_temp_file_size 0;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔒 4. Enabling HTTPS

```bash
sudo certbot --nginx -d n8n.your-domain.com
```

Let's Encrypt will automatically configure your SSL and enable auto-renewal.

After the certificate is issued, edit `n8n.conf` to include SSL and redirect HTTP
to HTTPS. Use `Ctrl + K` in `nano` to remove old lines and replace them with the
following configuration:

```nginx
server {
    listen 443 ssl;
    server_name n8n.your-domain.com;

    ssl_certificate     /etc/letsencrypt/live/n8n.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.your-domain.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;

        proxy_buffering on;
        proxy_buffers 8 256k;
        proxy_buffer_size 128k;
        proxy_busy_buffers_size 512k;
        proxy_max_temp_file_size 0;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 80;
    server_name n8n.your-domain.com;

    return 301 https://$host$request_uri;
}
```

Reload nginx:

```bash
sudo ln -sf /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 💾 5. Automatic Backups

Create a backup directory:

```bash
mkdir -p ~/n8n_backups
```

Create a backup script:

```bash
sudo nano /usr/local/bin/n8n-backup.sh
```

Insert the following:

```bash
#!/bin/bash

SRC_DIR="$HOME/n8n_data"
BACKUP_DIR="$HOME/n8n_backups"
BACKUP_FILE="$BACKUP_DIR/n8n-backup-$(date +%F-%H%M%S).tar.gz"

tar -czf "$BACKUP_FILE" -C "$SRC_DIR" .
ls -1t "$BACKUP_DIR"/n8n-backup-*.tar.gz | tail -n +8 | xargs -r rm --
echo "[$(date)] Backup created: $BACKUP_FILE"
```

Make it executable:

```bash
chmod +x /usr/local/bin/n8n-backup.sh
```

Add it to `cron` (runs daily at 03:00 AM):

```bash
crontab -e
```

Add this line:

```
0 3 * * * /usr/local/bin/n8n-backup.sh >> $HOME/n8n_backups/backup.log 2>&1
```

---

## 📦 Done!

Now you have:

- A working instance at `https://n8n.your-domain.com`
- HTTPS with auto-renewal
- Data stored in `~/n8n_data`
- Daily backups stored in `~/n8n_backups`

---

## 🧠 Handy Commands

- Manual backup: `n8n-backup.sh`
- Test nginx config: `sudo nginx -t`
- Reload nginx: `sudo systemctl reload nginx`
- View n8n logs: `docker logs -f n8n`
- Stop and remove the container
  ```bash
  docker stop n8n
  docker rm n8n
  ```

## Update

```bash
docker pull n8nio/n8n
```

Then restart the container after stopping and removing the previous version.
