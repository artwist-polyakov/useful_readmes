# AFFiNE: Self-hosted - Инструкция по установке

В этом руководстве описаны основные шаги для развертывания AFFiNE на собственном сервере.

[Официальная инструкция от разработчиков](https://docs.affine.pro/docs/self-host-affine) содержит минимум информации, поэтому я подготовил более подробную инструкцию с рекомендациями.

## Содержание
- [Аренда сервера](#аренда-сервера)
- [Рекомендуемое ПО для работы](#рекомендуемое-по-для-работы)
- [Установка и запуск AFFiNE](#установка-и-запуск-affine)
- [Тонкая настройка AFFiNE](#тонкая-настройка-affine)
- [Настройка домена с SSL](#настройка-домена-с-ssl)
- [Подключение ИИ-функций](#подключение-ии-функций)
- [Полезные команды](#полезные-команды)

## Аренда сервера

У AFFiNE есть мощный набор инструментов для работы с AI. Рекомендуется выбирать зарубежный сервер, так как с российских IP-адресов не получится делать запросы к OpenAI.

### Минимальные требования:
- 2 ГБ оперативной памяти
- 2 CPU
- OS: Ubuntu

> **Примечание**: На больших страницах AFFiNE может тормозить, возможно потребуется более мощная система.



## Рекомендуемое ПО для работы

- **Termius** - для работы с удаленным сервером через SSH
  - Позволяет быстро настроить авторизацию
  - Синхронизирует виртуальные машины между устройствами
  - [Скачать Termius](https://termius.com/)
  
- **Sublime Text** - для работы с текстовыми документами
  - [Скачать Sublime Text](https://www.sublimetext.com/)

## Установка и запуск AFFiNE

### 1. Установка Docker

Для запуска AFFiNE необходим Docker:

```bash
# Установка Docker
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y docker-ce docker-compose
```

Проверка установки:
```bash
docker --version
```

### 2. Скачивание репозитория

```bash
git clone https://github.com/toeverything/AFFiNE.git --branch stable
```

### 3. Настройка переменных окружения

Откройте файл конфигурации:
```bash
cd AFFiNE/
nano ./.github/deployment/self-host/compose.yaml
```

Заполните переменные:
- `AFFINE_ADMIN_EMAIL`
- `AFFINE_ADMIN_PASSWORD`

> **Примечание**: Хотя эти логин и пароль нужно указать, они не используются для входа в систему. Вам всё равно нужно будет создать новых пользователей.

> **Примечание 2**: Для сохранения в nano нажимаем `ctrl`+`X`, соглашаемся `Y`, `Enter`. 


### 4. Установка версии PostgreSQL

> **Примечание**: Другие пользователи подсказывают, что версию постгрес можно не менять. Видимо проблема только в миграциях, если база была создана в прошлой версии, то могут быть проблемы с обновлениями.

В том же файле `compose.yaml` установите версию PostgreSQL:
```yaml
postgres:
  image: postgres:16.4
```

### 5. Запуск приложения

```bash
docker compose -f ./.github/deployment/self-host/compose.yaml up -d
```

После запуска контейнеров, приложение будет доступно по адресу:
```
http://<YOURS-SERVER-IP>:3010
```

> **Важно**: У вас будет 2 пользователя в AFFiNE:
> - Администратор
> - Обычный пользователь для работы с функциями (AI, шеринг и т.д.)
>
> После создания администратора, создайте второго пользователя из админской панели. Обязательно выберите опцию "reset password" и сбросьте ему пароль в отдельном окне браузера.

## Тонкая настройка AFFiNE

### Задачи для настройки
- Предоставить боевому пользователю расширенные права
- Отключить лимиты на использование AI

### Подключение к базе данных

```bash
docker exec -it affine_postgres psql -U affine -d affine
```

### Получение информации о пользователях и ролях

Просмотр списка таблиц:
```sql
\dt
```

Просмотр пользователей:
```sql
SELECT * FROM users;
```

> **Важно**: Запишите идентификаторы пользователей (`user_id`), чтобы знать какой соответствует обычному пользователю, а какой администратору.

Просмотр ролей:
```sql
SELECT * FROM user_features;
SELECT * FROM features;
```

### Настройка ролей пользователей

Для администратора (удаление ограничений):
```sql
DELETE FROM user_features
WHERE user_id = '<admin-user-id>' AND feature_id = 13; 
```

Для обычного пользователя (удаление роли администратора):
```sql
DELETE FROM user_features
WHERE user_id = '<regular-user-id>' AND feature_id = 7; 
```

Установка расширенного плана для обычного пользователя:
```sql
UPDATE user_features
SET feature_id = 16
WHERE user_id = '<regular-user-id>'; 
```

Увеличение лимитов для AI:
```sql
UPDATE features
SET configs = jsonb_set(
                jsonb_set(configs::jsonb, '{memberLimit}', '100', false),
                '{copilotActionLimit}', '1000', false
            )
WHERE id = 16;
```

Для выхода из базы данных:
```sql
\q
```

## Настройка домена с SSL

### Настройка DNS
Пропишите A-запись для поддомена, указывающую на IP-адрес вашего сервера.

Отслеживать распространение DNS-записей можно через сервис [whatsmydns.net](https://www.whatsmydns.net/).

### Установка Nginx

```bash
sudo apt update
sudo apt install nginx
```

### Установка SSL с помощью Certbot

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d <YOURS-DOMAIN>
```

Проверка и перезапуск Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

Настройка автоматического обновления сертификата:
```bash
sudo certbot renew --dry-run
```

### Создание конфигурации Nginx для AFFiNE

```bash
sudo nano /etc/nginx/sites-available/affine
```

Содержимое файла (замените <YOURS-DOMAIN> на ваш домен):
```nginx
server {
  server_name <YOURS-DOMAIN>;
  location / {
    proxy_pass http://localhost:3010; # Прокси на локальный адрес
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    # Настройки для WebSocket
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
  error_page 404 /404.html;
  location = /404.html {
    internal;
  }
  listen 443 ssl; # managed by Certbot
  ssl_certificate /etc/letsencrypt/live/<YOURS-DOMAIN>/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/<YOURS-DOMAIN>/privkey.pem; # managed by Certbot
  include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
server {
  if ($host = <YOURS-DOMAIN>) {
    return 301 https://$host$request_uri;
  } # managed by Certbot
  listen 80;
  server_name <YOURS-DOMAIN>;
  return 404; # managed by Certbot
}
```

Создание символической ссылки:
```bash
sudo ln -s /etc/nginx/sites-available/affine /etc/nginx/sites-enabled/
```

### Настройка основного конфигурационного файла Nginx

```bash
sudo nano /etc/nginx/nginx.conf
```

Используйте следующую конфигурацию:
```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;
events {
  worker_connections 768;
  # multi_accept on;
}
http {
  ##
  # Basic Settings
  ##
  sendfile on;
  tcp_nopush on;
  types_hash_max_size 2048;
  # server_tokens off;
  # server_names_hash_bucket_size 64;
  # server_name_in_redirect off;
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
    client_max_body_size 100m; # Для загрузки контента на сервак
  ##
  # SSL Settings
  ##
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
  ssl_prefer_server_ciphers on;
  ##
  # Logging Settings
  ##
  access_log /var/log/nginx/access.log;
  ##
  # Gzip Settings
  ##
  gzip on;
  # gzip_vary on;
  # gzip_proxied any;
  # gzip_comp_level 6;
  # gzip_buffers 16 8k;
  # gzip_http_version 1.1;
  # gzip_types text/plain text/css application/json application/javascript text/xml
application/xml application/xml+rss text/javascript;
  ##
  # Virtual Host Configs
  ##
  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
```

Перезапуск Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Подключение ИИ-функций

Регистрация и получение API-ключей:
- [OpenAI Platform](https://platform.openai.com/docs/overview)
- [FAL.AI](https://fal.ai/)

Настройка конфигурационного файла:
```bash
sudo nano ~root/.affine/self-host/config/affine.js
```

Важные параметры:
```javascript
// Настройка HTTPS
AFFiNE.server.https = true;
AFFiNE.server.host = '<YOURS-DOMAIN>';
AFFiNE.server.port = 3010;
AFFiNE.server.externalUrl = 'https://<YOURS-DOMAIN>:443'

// Настройка API-ключей для ИИ-сервисов (в конце файла)
AFFiNE.use('copilot', {
   openai: {
     apiUrl: 'https://api.openapi.com/',
     apiKey: 'XXXX-XXXX',
   },
  fal: {
    apiKey: 'YYYYYYYYY',
  },
//   unsplashKey: 'your-key',
//   storage: {
//     provider: 'cloudflare-r2',
//     bucket: 'copilot',
//   }
 })
```

Перезапуск сервиса после настройки:
```bash
docker compose -f ./.github/deployment/self-host/compose.yaml restart
```

## Полезные команды

### Обновление пакетов
```bash
docker compose -f ./.github/deployment/self-host/compose.yaml pull
```

### Запуск (используйте при обновлении версии)
```bash
docker compose -f ./.github/deployment/self-host/compose.yaml up -d
```

### Перезапуск (сохранит текущую версию)
```bash
docker compose -f ./.github/deployment/self-host/compose.yaml restart
```

### Проверка и перезапуск Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Известные проблемы

В [AFFiNE есть критичный баг с шарингом](https://github.com/toeverything/AFFiNE/issues/8015) связных заметок.

---

*Инструкция составлена Александром Поляковым, Март 2025*
