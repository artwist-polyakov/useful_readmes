# 🐘 Deploying PostgreSQL with SSL in Docker (for n8n and DBeaver)

This guide explains how to run PostgreSQL in Docker, secure it with SSL, and make it available:

- ✅ Locally (for apps like n8n) **without SSL**
- ✅ Remotely (e.g. via DBeaver) **with SSL and certificates**

---

## 📁 1. Prepare Directories and Certificates

Create a directory for PostgreSQL data and SSL certificates:

```bash
mkdir -p ~/n8n_postgres
mkdir -p ~/n8n_postgres_ssl
```

Place your certificate and private key in `~/n8n_postgres_ssl/`:

```
~/n8n_postgres_ssl/server.crt
~/n8n_postgres_ssl/server.key
```

Set correct permissions:

```bash
chmod 600 ~/n8n_postgres_ssl/server.key
chmod 644 ~/n8n_postgres_ssl/server.crt
```

---

## 🐳 2. Run PostgreSQL in Docker

```bash
docker run -d --restart unless-stopped \
  --name postgres \
  -e POSTGRES_USER=aleksandrpoliakov \
  -e POSTGRES_PASSWORD=YourStrongPassword \
  -e POSTGRES_DB=n8n \
  -v ~/n8n_postgres:/var/lib/postgresql/data \
  -v ~/n8n_postgres_ssl:/var/lib/postgresql \
  --network n8n-net \
  -p 5432:5432 \
  postgres:16
```

---

## 🔧 3. Enable SSL in postgresql.conf

Open config inside the container:

```bash
docker exec -it postgres vi /var/lib/postgresql/data/postgresql.conf
```

Edit or add:

```conf
ssl = on
ssl_cert_file = '/var/lib/postgresql/server.crt'
ssl_key_file = '/var/lib/postgresql/server.key'
```

> Make sure paths are **absolute** and point to the mounted files.

---

## 🔐 4. Update `pg_hba.conf` for SSL access

In the same folder (`/var/lib/postgresql/data/pg_hba.conf`), add this line at the top:

```conf
hostssl all all 0.0.0.0/0 md5
```

(You can replace `0.0.0.0/0` with specific IP ranges later.)

---

## 🔁 5. Restart PostgreSQL

```bash
docker restart postgres
```

Check logs for errors:

```bash
docker logs postgres
```

---

## 🧩 6. How to connect

### 🔹 From **n8n** (inside Docker):

Use standard PostgreSQL node, connect via:

- Host: `postgres` (Docker hostname)
- Port: `5432`
- SSL: **disabled**

> n8n and postgres are in the same Docker network (`n8n-net`), so they talk directly without SSL.

---

### 🔹 From **DBeaver** (external connection with SSL):

1. Open the connection editor → tab **SSL**
2. Enable **"Use SSL"**
3. Upload:
   - **Root cert**: `server.crt`
   - **Client cert**: *(not required)*
   - **Client key**: *(not required)*
4. Choose:
   - **SSL mode**: `require` or `verify-ca`
5. Test connection

