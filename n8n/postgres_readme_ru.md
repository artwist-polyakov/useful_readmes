# 🐘 PostgreSQL в Docker с поддержкой SSL и приватной сетью для n8n

Этот гайд описывает, как развернуть PostgreSQL в Docker-контейнере с включенным SSL, приватной сетью и доступом:

- из n8n по внутренней сети **без SSL**
- из DBeaver — **с SSL**, используя самоподписанные сертификаты

---

## 📦 1. Подготовка директорий

Создаём каталоги для:

- данных PostgreSQL
- SSL-сертификатов

```bash
mkdir -p ~/n8n_postgres
mkdir -p ~/n8n_postgres_ssl
```

---

## 🔐 2. Генерация самоподписанного сертификата

```bash
openssl req -new -x509 -days 365 -nodes \
  -out ~/n8n_postgres_ssl/server.crt \
  -keyout ~/n8n_postgres_ssl/server.key \
  -subj "/CN=n8n_postgres"

chmod 600 ~/n8n_postgres_ssl/server.key
chmod 644 ~/n8n_postgres_ssl/server.crt
```

---

## 🐳 3. Запуск контейнера PostgreSQL

```bash
docker network create n8n-net

docker run -d \
  --name postgres \
  --restart unless-stopped \
  --network n8n-net \
  -p 5432:5432 \
  -e POSTGRES_USER=aleksandrpoliakov \
  -e POSTGRES_PASSWORD=пароль \
  -e POSTGRES_DB=n8n \
  -v ~/n8n_postgres:/var/lib/postgresql/data \
  -v ~/n8n_postgres_ssl/server.crt:/var/lib/postgresql/server.crt \
  -v ~/n8n_postgres_ssl/server.key:/var/lib/postgresql/server.key \
  postgres:16
```

---

## ⚙️ 4. Включение SSL в конфиге PostgreSQL

Открываем файл:

```bash
nano ~/n8n_postgres/postgresql.conf
```

Вставляем/раскомментируем:

```conf
ssl = on
ssl_cert_file = '/var/lib/postgresql/server.crt'
ssl_key_file = '/var/lib/postgresql/server.key'
```

Затем перезапускаем PostgreSQL:

```bash
docker restart postgres
```

---

## 📥 Подключение из n8n

В конфигурации PostgreSQL-ноды в n8n:

- Host: `postgres`
- Port: `5432`
- SSL: `отключен`
- Подключение работает через сеть `n8n-net`

---

## 📤 Подключение из DBeaver (с SSL)

1. Включаем SSL в настройках подключения
2. Указываем:
   - **SSL режим:** `require`
   - **Клиентский сертификат:** не требуется
   - **Клиентский ключ:** не требуется
   - **Корневой сертификат:** `server.crt`

---

## 🔐 Результат

- Внутреннее подключение — без SSL
- Внешнее подключение (например, из DBeaver) — **через SSL** с самоподписанным сертификатом.

