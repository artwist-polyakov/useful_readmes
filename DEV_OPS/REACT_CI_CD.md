# 🚀 Инструкция по развертыванию (CI/CD) проекта на React

Этот гайд описывает полную настройку сервера Ubuntu для автоматического развертывания (CI/CD) React-проекта.

Мы настроим Caddy (веб-сервер с авто-HTTPS) и GitHub Action, которая будет автоматически собирать и публиковать ваш сайт при каждом `git push`.

### 🧭 Переменные

В этой инструкции мы будем использовать следующие переменные. Замените их *значения* на свои.

| Переменная | Описание | Пример значения |
| :--- | :--- | :--- |
| **`$DOMAIN`** | Ваш публичный домен. | `gooooooooooogle.ru` |
| **`$SERVER_IP`** | Публичный IP-адрес вашего сервера. | `195.2.79.226` |
| **`$PROJECT_NAME`** | Короткое имя проекта (для папок). | `ai-moms` |
| **`$DEPLOY_USER`** | Имя сервисного пользователя на сервере. | `deployer` |

-----

## Часть 1: Настройка сервера (Один раз)

Эти шаги выполняются **только один раз** на вашем сервере Ubuntu.

### 1\. Установка Caddy

Caddy — это наш веб-сервер. Он автоматически получит SSL-сертификат для вашего домена.

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### 2\. Настройка Firewall (UFW)

Открываем порты для HTTP (получение сертификата) и HTTPS (обслуживание сайта).

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### 3\. Создание директории для сайта

Мы будем хранить публичные файлы сайта в `/var/www/html/`.

```bash
# Используйте здесь ваш $PROJECT_NAME
sudo mkdir -p /var/www/html/$PROJECT_NAME
sudo chown -R caddy:caddy /var/www/html/$PROJECT_NAME
```

> **Пример:**
> `sudo mkdir -p /var/www/html/ai-moms`
> `sudo chown -R caddy:caddy /var/www/html/ai-moms`

### 4\. Настройка Caddy

Откройте `Caddyfile` для редактирования:

```bash
sudo nano /etc/caddy/Caddyfile
```

**Полностью удалите** всё содержимое и вставьте эту конфигурацию, используя ваши `$DOMAIN` и `$PROJECT_NAME`.

```caddy
$DOMAIN {
    # Путь к папке, которую мы создали
    root * /var/www/html/$PROJECT_NAME

    # Обработка 404 ошибок для React Router
    try_files {path} /index.html

    # Включение режима файлового сервера
    file_server
}
```

> **Пример:**
>
> ```caddy
> gooooooooooogle.ru {
>     root * /var/www/html/ai-moms
>     try_files {path} /index.html
>     file_server
> }
> ```

Сохраните файл (`Ctrl+O`, `Enter`) и выйдите (`Ctrl+X`).

### 5\. Создание пользователя `deployer`

Мы создадим специального пользователя, который будет заниматься *только* развертыванием.

```bash
# Используйте здесь ваш $DEPLOY_USER
sudo adduser $DEPLOY_USER --disabled-password --gecos ""
```

> **Пример:**
> `sudo adduser deployer --disabled-password --gecos ""`

### 6\. Настройка SSH для `deployer`

GitHub Action будет подключаться к серверу от имени этого пользователя.

Шаги надо делать последовательно по номерам. Не копируйте всё содержимое контейнера в консоль.

```bash
# 1. Переключаемся на пользователя $DEPLOY_USER
sudo su - $DEPLOY_USER

# 2. Создаем папку .ssh и настраиваем права
mkdir ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# 3. Генерируем SSH-ключ (назовем id_rsa_action)
# Нажимайте Enter на все вопросы (пароль должен быть пустым)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_action

# 4. Добавляем этот ключ в список "разрешенных"
cat ~/.ssh/id_rsa_action.pub >> ~/.ssh/authorized_keys

# 5. Выводим ПРИВАТНЫЙ ключ. Он понадобится для GitHub.
echo "--- Скопируйте этот ключ в GitHub Secrets ---"
cat ~/.ssh/id_rsa_action
echo "-------------------------------------------"

# 6. Возвращаемся в root
exit
```

### 7\. Исправление прав SSH (Критически важно)

SSH не будет работать, если права на папки и файлы неверные.

```bash
# Убедимся, что $DEPLOY_USER владеет своей папкой .ssh
sudo chown -R $DEPLOY_USER:$DEPLOY_USER /home/$DEPLOY_USER/.ssh

# Папка .ssh должна быть доступна ТОЛЬКО владельцу
sudo chmod 700 /home/$DEPLOY_USER/.ssh

# Файл authorized_keys должен быть доступен ТОЛЬКО владельцу
sudo chmod 600 /home/$DEPLOY_USER/.ssh/authorized_keys
```

> **Пример:**
> `sudo chmod 700 /home/deployer/.ssh`
> `sudo chmod 600 /home/deployer/.ssh/authorized_keys`

### 8\. Создание Deploy-скрипта

Этот скрипт будет выполнять всю "грязную работу" по публикации файлов.

```bash
# 1. Создаем файл скрипта, используя $PROJECT_NAME
sudo nano /usr/local/bin/deploy-$PROJECT_NAME
```

> **Пример:**
> `sudo nano /usr/local/bin/deploy-ai-moms`

**Вставьте в этот файл** следующий код, используя ваши `$DEPLOY_USER` и `$PROJECT_NAME`:

```bash
#!/bin/bash
set -e

# 1. Очищаем старую папку
rm -rf /var/www/html/$PROJECT_NAME/*

# 2. Копируем новую сборку (из папки, куда ее прислал Action)
cp -r /home/$DEPLOY_USER/dist/* /var/www/html/$PROJECT_NAME/

# 3. Устанавливаем владельца для Caddy
chown -R caddy:caddy /var/www/html/$PROJECT_NAME

# 4. Удаляем временную папку 'dist'
rm -rf /home/$DEPLOY_USER/dist

echo "✅ Deployment successful!"
```

Сохраните (`Ctrl+O`, `Enter`) и выйдите (`Ctrl+X`).

### 9\. Настройка `sudo` для `deployer`

Мы дадим `deployer` право запускать **только** этот скрипт от имени `root`.

```bash
# 1. Делаем скрипт исполняемым
sudo chmod +x /usr/local/bin/deploy-$PROJECT_NAME

# 2. Открываем файл sudoers для $DEPLOY_USER
sudo nano /etc/sudoers.d/$DEPLOY_USER

# 3. Вставляем ОДНУ строку, дающую права на скрипт:
$DEPLOY_USER ALL=(ALL) NOPASSWD: /usr/local/bin/deploy-$PROJECT_NAME

# 4. Сохраняем (Ctrl+O, Enter) и выходим (Ctrl+X)
```

> **Пример файла `/etc/sudoers.d/deployer`:**
> `deployer ALL=(ALL) NOPASSWD: /usr/local/bin/deploy-ai-moms`

-----

## \#\# Часть 2: Настройка GitHub (Один раз)

Перейдите в настройки вашего репозитория на GitHub.

1.  Перейдите в `Settings` \> `Secrets and variables` \> `Actions`.

2.  Нажмите `New repository secret` и создайте **три** секрета:

      * **`SERVER_HOST`**

          * **Значение:** Ваш `$SERVER_IP` (напр. `195.2.79.226`)

      * **`SERVER_USER`**

          * **Значение:** Ваш `$DEPLOY_USER` (напр. `deployer`)

      * **`SERVER_SSH_KEY`**

          * **Значение:** Вставьте сюда **ПРИВАТНЫЙ** ключ, который вы скопировали на **Шаге 1.6** (`cat ~/.ssh/id_rsa_action`).

-----

## \#\# Часть 3: Создание GitHub Action (Один раз)

В **локальной** папке вашего проекта (на вашем компьютере или сразу в githib.com) создайте файл:

**`.github/workflows/deploy.yml`**

‼️ Обратите внимание, что в конце надо заменить `ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} 'sudo /usr/local/bin/deploy-ai-moms'` на `ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} 'sudo /usr/local/bin/deploy-$PROJECT_NAME'`

```yaml
name: Build and Deploy

# Запускать при каждом пуше в 'main'
on:
  push:
    branches:
      - main  # Убедитесь, что это ваша главная ветка

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: 1. Checkout repository
        uses: actions/checkout@v4

      - name: 2. Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: 3. Install dependencies
        run: npm install

      - name: 4. Build project
        # Сборка проекта
        run: npm run build

      - name: 5. Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SERVER_SSH_KEY }}

      - name: 6. Add server to known hosts
        run: ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts

      - name: 7. Deploy to server
        # Используем переменные, которые мы задали в GitHub Secrets
        run: |
          # 7a. Копируем 'dist' на сервер в /home/$DEPLOY_USER/
          scp -r ./dist ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }}:/home/${{ secrets.SERVER_USER }}/

          # 7b. Подключаемся и запускаем наш скрипт деплоя
          # (Замените 'deploy-ai-moms' на 'deploy-ВАШ_PROJECT_NAME')
          ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} 'sudo /usr/local/bin/deploy-ai-moms'
```

-----

## Ваш новый рабочий процесс

Теперь вам больше не нужно заходить на сервер.

1.  Вы пишете код на своем компьютере или через ИИ-конструктор.
2.  Делаете `git commit` и `git push` в ветку `main`. Или дожидаетесь, что ИИ-контруктор синхронизируется через git.
3.  GitHub Action запускается автоматически, собирает проект и публикует его на сервере.
4.  Через 1-2 минуты ваш сайт обновлен.
