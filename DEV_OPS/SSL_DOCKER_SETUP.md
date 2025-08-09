# 🔐 Универсальная инструкция по развертыванию проекта с SSL сертификатом в Docker

## Архитектура системы

Система состоит из трех Docker-контейнеров:
1. **App** - Ваше основное приложение (любой язык/фреймворк)
2. **Nginx** - Веб-сервер с SSL терминацией
3. **Certbot** - Автоматическое получение и обновление Let's Encrypt сертификатов

## 📁 Структура проекта

```
your-project/
├── app/
│   ├── Dockerfile
│   └── [файлы вашего приложения]
├── nginx/
│   ├── Dockerfile
│   └── conf.d/
│       └── app.conf.template
├── certbot/
│   ├── Dockerfile
│   ├── conf/          (создается автоматически)
│   └── www/           (создается автоматически)
├── static_files/      (опционально)
│   └── [статические файлы]
├── scripts/
│   ├── nginx-entrypoint.sh
│   └── certbot-entrypoint.sh
├── docker-compose.yml
└── .env
```

## 🔧 Шаг 1: Настройка переменных окружения

Создайте файл `.env` в корне проекта:

```bash
# Домены и Email для Let's Encrypt
LE_DOMAINS=yourdomain.com,www.yourdomain.com
LE_EMAIL=your-email@domain.com

# Тестовый режим Let's Encrypt (убирает лимиты API)
# LE_STAGING=true

# Переменные для вашего приложения
# APP_ENV=production
# DATABASE_URL=...
# API_KEY=...
```

### Обязательные переменные:
- `LE_DOMAINS` - домены для SSL сертификата (через запятую)
- `LE_EMAIL` - email для уведомлений Let's Encrypt

## 🐳 Шаг 2: Docker файлы

### app/Dockerfile (пример для Python)

```dockerfile
FROM python:3.11-slim
# ИЛИ
# FROM node:18-alpine
# FROM openjdk:11-jre-slim
# FROM nginx:alpine

WORKDIR /app

# Копируем зависимости и устанавливаем
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Копируем код приложения
COPY .. .

# Команда запуска вашего приложения
CMD ["python", "main.py"]
# ИЛИ
# CMD ["npm", "start"]
# CMD ["java", "-jar", "app.jar"]
```

### nginx/Dockerfile
```dockerfile
FROM nginx:alpine

RUN apk add --no-cache openssl gettext

COPY ./scripts/nginx-entrypoint.sh /docker-entrypoint.d/10-init.sh
RUN chmod +x /docker-entrypoint.d/10-init.sh
```

### certbot/Dockerfile
```dockerfile
FROM certbot/certbot

RUN apk add --no-cache docker-compose curl

COPY ./scripts/certbot-entrypoint.sh /scripts/certbot-entrypoint.sh
RUN chmod +x /scripts/certbot-entrypoint.sh
```

## ⚙️ Шаг 3: Конфигурация Nginx

### nginx/conf.d/app.conf.template
```nginx
server {
    listen 80;
    server_name ${LE_DOMAINS};

    # Блок для проверки Let's Encrypt
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Перенаправляем все HTTP запросы на HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name ${LE_DOMAINS};

    # Пути к SSL сертификатам
    ssl_certificate /etc/letsencrypt/live/${LE_DOMAINS}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${LE_DOMAINS}/privkey.pem;

    # ВАРИАНТ 1: Статические файлы
    root /var/www/html;
    index index.html index.php;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # ВАРИАНТ 2: Проксирование к вашему приложению
    # location / {
    #     proxy_pass http://app:8000;
    #     proxy_set_header Host $host;
    #     proxy_set_header X-Real-IP $remote_addr;
    #     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #     proxy_set_header X-Forwarded-Proto $scheme;
    # }

    # ВАРИАНТ 3: PHP-FPM
    # location ~ \.php$ {
    #     try_files $uri =404;
    #     fastcgi_split_path_info ^(.+\.php)(/.+)$;
    #     fastcgi_pass app:9000;
    #     fastcgi_index index.php;
    #     include fastcgi_params;
    #     fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    # }

    # Служебные файлы
    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }
}
```

## 📜 Шаг 4: Скрипты инициализации

### scripts/nginx-entrypoint.sh
```bash
#!/bin/sh
set -e

if [ -z "$LE_DOMAINS" ]; then
  echo "Переменная LE_DOMAINS не установлена." >&2
  exit 1
fi

DOMAIN=$(echo "$LE_DOMAINS" | cut -d ',' -f 1)
CERT_PATH="/etc/letsencrypt/live/$DOMAIN"

# Подставляем переменные окружения в шаблон конфигурации
envsubst '${LE_DOMAINS}' < /etc/nginx/conf.d/app.conf.template > /etc/nginx/conf.d/default.conf

# Создаем временный самоподписанный сертификат если его нет
if [ ! -f "$CERT_PATH/fullchain.pem" ]; then
  echo "### Создание временного самоподписанного сертификата для $DOMAIN... ###"
  mkdir -p "$CERT_PATH"
  openssl req -x509 -nodes -newkey rsa:2048 -days 1 \
    -keyout "$CERT_PATH/privkey.pem" \
    -out "$CERT_PATH/fullchain.pem" \
    -subj "/CN=localhost"
fi
```

### scripts/certbot-entrypoint.sh
```bash
#!/bin/sh
set -e

# Проверяем обязательные переменные
if [ -z "$LE_DOMAINS" ] || [ -z "$LE_EMAIL" ]; then
  echo "Переменные LE_DOMAINS или LE_EMAIL не установлены. Пропуск..." >&2
  exit 0
fi

# Тестовый режим Let's Encrypt
STAGING_FLAG=""
if [ "$LE_STAGING" = "true" ] || [ "$LE_STAGING" = "1" ]; then
  echo ">>> Включаем тестовый режим Let's Encrypt <<<"
  STAGING_FLAG="--staging"
fi

# ЗАМЕНИТЕ НА НАЗВАНИЕ ВАШЕГО ПРОЕКТА
PROJECT_NAME="your-project-name"
DOMAIN=$(echo "$LE_DOMAINS" | cut -d ',' -f 1)
CERT_PATH="/etc/letsencrypt/live/$DOMAIN"

# Ждем запуска Nginx
echo "### Ожидание запуска Nginx... ###"
until curl -s http://nginx; do
  >&2 echo "Nginx недоступен, ждем 5 секунд..."
  sleep 5
done
echo ">>> Nginx готов!"

# Проверяем нужность нового сертификата
if ! openssl x509 -in "$CERT_PATH/fullchain.pem" -checkend 86400 >/dev/null 2>&1 || \
   openssl x509 -in "$CERT_PATH/fullchain.pem" -subject -noout | grep -q "CN = localhost"; then

  echo "### Запрашиваем новый сертификат... ###"
  
  # Удаляем временный сертификат
  rm -rf "$CERT_PATH"

  # Получаем реальный сертификат
  certbot certonly --webroot --webroot-path=/var/www/certbot \
    --non-interactive --agree-tos --email "$LE_EMAIL" \
    -d "$LE_DOMAINS" --rsa-key-size 4096 --force-renewal \
    --cert-name $DOMAIN \
    $STAGING_FLAG

  echo "### Перезагрузка Nginx... ###"
  docker-compose -p "$PROJECT_NAME" restart nginx
fi

# Запуск цикла обновления сертификатов
echo "### Запуск цикла обновления сертификатов... ###"
trap exit TERM;
while :; do
  (certbot renew --post-hook "docker-compose -p '$PROJECT_NAME' restart nginx" $STAGING_FLAG) || true
  sleep 12h
done
```

## 🐳 Шаг 5: Docker Compose

### docker-compose.yml
```yaml
services:
  app:
    build: ./app
    env_file:
      - .env
    restart: unless-stopped
    # Для веб-приложений откройте нужный порт:
    # ports:
    #   - "8000:8000"
    
    # Для приложений с базой данных:
    # depends_on:
    #   - db

  nginx:
    build:
      context: .
      dockerfile: nginx/Dockerfile
    ports:
      - "80:80"
      - "443:443"
    volumes:
      # Подключите ваши статические файлы
      - ./static_files:/var/www/html
      # Обязательные для SSL
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
      - ./nginx/conf.d/app.conf.template:/etc/nginx/conf.d/app.conf.template
    env_file:
      - .env
    depends_on:
      - app
    restart: unless-stopped

  certbot:
    build:
      context: .
      dockerfile: certbot/Dockerfile
    env_file:
      - .env
    volumes:
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock
      - ./:/app:ro
    working_dir: /app
    entrypoint: /scripts/certbot-entrypoint.sh
    depends_on:
      - nginx
    restart: unless-stopped

  # Пример базы данных (опционально)
  # db:
  #   image: postgres:15
  #   environment:
  #     POSTGRES_DB: mydb
  #     POSTGRES_USER: user
  #     POSTGRES_PASSWORD: password
  #   volumes:
  #     - db_data:/var/lib/postgresql/data
  #   restart: unless-stopped

# volumes:
#   db_data:
```

## 🚀 Шаг 6: Запуск системы

### Предварительные требования:
1. Зарегистрированный домен
2. A/AAAA записи DNS указывают на ваш сервер
3. Открыты порты 80 и 443
4. Установлен Docker и Docker Compose

### Пошаговый запуск:

```bash
# 1. Создаем структуру проекта
mkdir your-project && cd your-project

# 2. Создаем все необходимые директории
mkdir -p app nginx/conf.d certbot scripts static_files

# 3. Копируем файлы согласно инструкции выше

# 4. Создаем .env файл
cat > .env << EOF
LE_DOMAINS=yourdomain.com
LE_EMAIL=your-email@domain.com
EOF

# 5. Делаем скрипты исполняемыми
chmod +x scripts/*.sh

# 6. ВАЖНО: Обновите PROJECT_NAME в scripts/certbot-entrypoint.sh

# 7. Запускаем систему
docker-compose up -d --build

# 8. Проверяем логи
docker-compose logs -f
```

## 🔧 Настройка для разных типов приложений

### Python Web App (Flask/Django/FastAPI)
```yaml
# docker-compose.yml - секция app
app:
  build: ./app
  ports:
    - "8000:8000"  # внутренний порт приложения
  
# nginx/conf.d/app.conf.template - добавить в server блок
location / {
    proxy_pass http://app:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### Node.js App
```yaml
app:
  build: ./app
  ports:
    - "3000:3000"
    
# В nginx конфиге: proxy_pass http://app:3000;
```

### Статический сайт
```yaml
# Уберите секцию app из docker-compose.yml
# Используйте только nginx с volume для статических файлов
volumes:
  - ./static_files:/var/www/html
```

### PHP приложение
```dockerfile
# app/Dockerfile
FROM php:8.1-fpm
# ... установка зависимостей

# В nginx конфиге раскомментируйте PHP-FPM секцию
```

## 📊 Проверка и мониторинг

### Проверка SSL сертификата:
```bash
# Проверка сертификата
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com

# Проверка статуса
curl -I https://yourdomain.com

# SSL тест
curl -I https://www.ssllabs.com/ssltest/analyze.html?d=yourdomain.com
```

### Полезные команды:
```bash
# Логи сервисов
docker-compose logs nginx
docker-compose logs certbot
docker-compose logs app

# Перезапуск
docker-compose restart nginx
docker-compose restart app

# Принудительное обновление сертификата
docker-compose exec certbot certbot renew --force-renewal

# Проверка конфигурации nginx
docker-compose exec nginx nginx -t
```

## 🔄 Автоматизация

### Cron для дополнительной надежности (опционально):
```bash
# Добавить в crontab сервера
0 3 * * * cd /path/to/project && docker-compose exec certbot certbot renew --quiet
```

### Мониторинг сертификатов:
```bash
# Скрипт проверки срока действия
#!/bin/bash
DOMAIN="yourdomain.com"
DAYS_LEFT=$(openssl s_client -connect $DOMAIN:443 -servername $DOMAIN 2>/dev/null | openssl x509 -noout -dates | grep notAfter | cut -d= -f2 | xargs -I {} date -d "{}" +%s)
CURRENT=$(date +%s)
DIFF=$(( ($DAYS_LEFT - $CURRENT) / 86400 ))
echo "Дней до истечения сертификата: $DIFF"
```

## 🛡️ Безопасность и оптимизация

### Дополнительные настройки SSL в nginx:
```nginx
# Добавить в server блок для HTTPS
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;

# HSTS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Дополнительные заголовки безопасности
add_header X-Frame-Options DENY always;
add_header X-Content-Type-Options nosniff always;
add_header X-XSS-Protection "1; mode=block" always;
```

---

*Эта универсальная инструкция подходит для любого проекта и может быть адаптирована под различные технологии и фреймворки.*
