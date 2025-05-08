# Создание VLESS сервера

## Аренда VPS
Для того, чтобы создать сервер, необходимо завести виртулальную машину. Довольно дешево арендовать тут https://akile.io/

## Установка 3x-ui

Сначала устанавливаем curl

```bash
apt update && apt upgrade -y
apt install curl -y
```
установим панель 3X-UI

```bash

bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)

```

В ходе установки попросит ввести порт. Это может быть любое число до 65535.
Считается, что лучше, чтобы число было ближе к концу диапазона, но подтверждения этому нет. 
В конце устанвки будет адрес входа панели и логин + пароль панели управления сервером.

Чтобы посмотреть текущие логин с паролем используем команду 

```bash

x-ui settings

```

чтобы заменить существующие логин с парольем используем команду

```bash

x-ui

```

далее выбираем Reset Usename & Password.


## Установка fail2bin

```bash

apt install fail2ban -y && systemctl start fail2ban && systemctl enable fail2ban

```

добавим пользователя

```bash

adduser username

```

делаем пользователя sudo-пользователем

```bash

usermod -aG sudo username

```

и переключаемся на него

```bash

su - username

```

отредактируем права

```bash

sudo nano /etc/ssh/sshd_config

```

Делаем PermitRootLogin — **no**

## Конфигурация клиента

Создаём новое соединение.
uTLS — chrome.
Dest — google.com:443
Server names — google.com, www.google.com

Ключи получаем через кнопку генерации ключей.


Sniffing — enabled + все галки
