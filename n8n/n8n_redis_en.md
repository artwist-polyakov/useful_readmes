# 🚀 Using Redis with n8n: Caching and Data Storage

This is an add-on to the main `n8n` deployment guide. It describes how to add and configure **Redis** for two key tasks:

1.  **Global Caching:** To speed up n8n's internal operations.
2.  **Data Storage:** To connect to Redis directly from your workflows.

---

## 🐳 1. Run the Redis Container

This step is common for both scenarios. Redis will run in the same `n8n-net` Docker network as n8n, ensuring a secure and fast connection.

Create a local directory for Redis data, where it will store its database snapshots:

```bash
mkdir -p ~/redis_data
```

Run the Redis container:

```bash
docker run -d \
  --name redis \
  --restart unless-stopped \
  --network n8n-net \
  -v ~/redis_data:/data \
  redis:latest redis-server --save 60 1 --loglevel warning
```

**What's happening here:**
* `--name redis`: Sets the container name to `redis`. Other containers on the `n8n-net` network can use this name to connect.
* `--network n8n-net`: Connects Redis to our isolated network.
* `-v ~/redis_data:/data`: Mounts a host directory to store Redis data (snapshots).
* `redis-server --save 60 1`: Additional parameters for Redis. This tells it to save its state to disk (to the `/data` folder inside the container) if at least 1 key has changed in the last 60 seconds. This is useful for backups.

---

## ⚙️ 2. Choose Your Use Case

Now that Redis is running, you can configure n8n to work with it.

### Scenario A: Global Caching to Speed Up n8n

This scenario improves n8n's overall performance, especially with a high volume of workflow executions. This is configured via environment variables.

**1. Stop and remove the current n8n container:**

```bash
docker stop n8n
docker rm n8n
```

**2. Run n8n with the new Redis environment variables:**

Add the following `-e` flags to your `docker run` command for n8n:

```bash
docker run -d --restart unless-stopped \
  --name n8n \
  --network n8n-net \
  -p 5678:5678 \
  # ... your current variables ...
  -e N8N_HOST="n8n.your-domain.com" \
  -e WEBHOOK_TUNNEL_URL="[https://n8n.your-domain.com/](https://n8n.your-domain.com/)" \
  \
  # ----- New variables for GLOBAL caching -----
  -e CACHE_ENABLED=true \
  -e CACHE_TYPE=redis \
  -e CACHE_REDIS_HOST=redis \
  # ---------------------------------------------
  \
  n8nio/n8n
```

### Scenario B: Connecting to Redis from Workflows (as Credentials)

This scenario allows you to use the `Redis` node in your workflows to store and retrieve data (e.g., for counters, temporary flags, or caching API request results).

**No additional environment variables are needed for n8n.** Just make sure the Redis container from step 1 is running.

**How to set up the connection in n8n:**

1.  In the n8n UI, go to `Credentials` and click `Add credential`.
2.  Search for `Redis` and select it.
3.  Fill in the fields:
    * **Credential Name:** Give it a descriptive name, e.g., `Local Redis`.
    * **Host:** `redis` (this is our container's name in the Docker network).
    * **Port:** `6379` (the default port).
    * Leave the other fields (User, Password) empty, as we haven't configured authentication.
4.  Click **Save**.

Now you can use these credentials in any `Redis` node within your workflows.

---

## 💾 3. Adding Redis to the Backup Script

To ensure your backups include not only the n8n database but also the data from Redis, update your backup script.

Open the script:

```bash
sudo nano /usr/local/bin/n8n-backup.sh
```

Modify it to look like this:

```bash
#!/bin/bash

# Directories to back up
N8N_DATA_DIR="$HOME/n8n_data"
REDIS_DATA_DIR="$HOME/redis_data"
# If you use Postgres, you can add its dump as well
# POSTGRES_BACKUP_FILE="$HOME/n8n_backups/postgres_dump_$(date +%F).sql"

BACKUP_DIR="$HOME/n8n_backups"
BACKUP_FILE="$BACKUP_DIR/n8n-full-backup-$(date +%F-%H%M%S).tar.gz"

# Create a Postgres dump (optional)
# docker exec postgres pg_dumpall -U your_user > "$POSTGRES_BACKUP_FILE"

echo "[$(date)] Creating full backup..."

# Archive all data into a single file
# Note the -C flag: we change the directory to preserve the folder structure
tar -czf "$BACKUP_FILE" \
    -C "$HOME" "n8n_data" \
    -C "$HOME" "redis_data"
    # -C "$HOME/n8n_backups" "$(basename $POSTGRES_BACKUP_FILE)" # Add this if you're dumping PG

# Remove old backups (keep the last 7)
ls -1t "$BACKUP_DIR"/n8n-full-backup-*.tar.gz | tail -n +8 | xargs -r rm --

# Remove the old Postgres dump (optional)
# rm "$POSTGRES_BACKUP_FILE"

echo "[$(date)] Backup created: $BACKUP_FILE"
```

This script will now package both the n8n data and the Redis data into a single archive.
