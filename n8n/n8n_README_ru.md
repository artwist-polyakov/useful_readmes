# 🧠 n8n на собственном сервере с доменом, HTTPS и бэкапами

Полноценная инструкция по развертыванию `n8n` на VPS с Ubuntu, собственным доменом и автоматическим бэкапом данных. Всё, что нужно для продакшна.

---

## 🛠️ Что будет в итоге:

- Собственный сервер n8n, доступный по `https://n8n.ваш-домен.ru` (можете выбрать иной поддомен, на свой вкус, но в примерах этой инструкции будет такой)
- HTTPS-сертификат от Let's Encrypt (автообновление)
- Проксирование через nginx
- Автоматические ежедневные бэкапы
- Поддержка буферизации в Nginx — улучшит работу воркфлоу с работающих с файлами.

---

## ⚙️ Требования:

- VPS с Ubuntu 20+ (например, 2 ГБ RAM). Лично я беру серверы тут [VDSina.com — партнерская сылка 10% бонус к пополнению](https://www.vdsina.com/?partner=1r8tcykewa) или [Firstbyte](https://firstbyte.ru/?from=196382)
- Установленный Docker
- Домен + доступ к DNS (я держу домены в reg.ru)

---

## 🔧 1. Настройка DNS

Добавьте A-запись в панели управления доменом:

```
n8n     A     <Внешний IP вашего сервера>
```

---

## 🐳 2. Запуск n8n через Docker

Для подключения по SSH я предпочитаю утилиту [Termius](https://termius.com).
Предварительно следуюет настроить на сервере Docker: [инструкция на оф сайте](https://docs.docker.com/engine/install/ubuntu/)


Создайте локальную папку для данных:

```bash
mkdir -p ~/n8n_data
```

Затем запустите контейнер:

```bash
docker run -d --restart unless-stopped \
  --name n8n \
  -p 5678:5678 \
  -v ~/n8n_data:/home/node/.n8n \
  -e N8N_HOST="n8n.ваш-домен.ru" \
  -e WEBHOOK_TUNNEL_URL="https://n8n.ваш-домен.ru/" \
  -e WEBHOOK_URL="https://n8n.ваш-домен.ru/" \
  n8nio/n8n
```

**Важно**: если переносите данные вручную, убедитесь, что у папки правильные права:

```bash
sudo chown -R 1000:1000 ~/n8n_data
sudo chmod -R u+rwX ~/n8n_data
```

---

## 🌐 3. Установка nginx и сертификата

Установите nginx и certbot:

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

Создайте файл:

```bash
sudo nano /etc/nginx/sites-available/n8n.conf
```

Вставьте:

```nginx
server {
    listen 443 ssl;
    server_name n8n.ваш-домен.ru;

    ssl_certificate     /etc/letsencrypt/live/n8n.ваш-домен.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.ваш-домен.ru/privkey.pem;
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
    server_name n8n.ваш-домен.ru;

    return 301 https://$host$request_uri;
}
```

Активируйте сайт:

```bash
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔒 4. Установка HTTPS

```bash
sudo certbot --nginx -d n8n.ваш-домен.ru
```

Let's Encrypt автоматически настроит сертификат и включит автоматическое продление.

---

## 💾 5. Автоматические бэкапы

Создайте папку:

```bash
mkdir -p ~/n8n_backups
```

Создайте скрипт:

```bash
sudo nano /usr/local/bin/n8n-backup.sh
```

Вставьте:

```bash
#!/bin/bash

SRC_DIR="$HOME/n8n_data"
BACKUP_DIR="$HOME/n8n_backups"
BACKUP_FILE="$BACKUP_DIR/n8n-backup-$(date +%F-%H%M%S).tar.gz"

tar -czf "$BACKUP_FILE" -C "$SRC_DIR" .
ls -1t "$BACKUP_DIR"/n8n-backup-*.tar.gz | tail -n +8 | xargs -r rm --
echo "[$(date)] Backup created: $BACKUP_FILE"
```

Сделайте исполняемым:

```bash
chmod +x /usr/local/bin/n8n-backup.sh
```

Добавьте в `cron` (выполняется каждый день в 3:00):

```bash
crontab -e
```

И добавьте:

```
0 3 * * * /usr/local/bin/n8n-backup.sh >> $HOME/n8n_backups/backup.log 2>&1
```

---

## 📦 Готово!

Теперь:

- `n8n` работает на `https://n8n.ваш-домен.ru`
- Есть HTTPS и автоматическое продление
- Все данные сохраняются в `~/n8n_data`
- Ежедневно создаются бэкапы в `~/n8n_backups`

---

## 🧠 Полезные команды

- Ручной бэкап: `n8n-backup.sh`
- Проверка nginx: `sudo nginx -t`
- Перезапуск nginx: `sudo systemctl reload nginx`
- Логи n8n: `docker logs -f n8n`
