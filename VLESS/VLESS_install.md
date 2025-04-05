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
