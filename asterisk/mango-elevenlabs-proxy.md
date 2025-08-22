# Инструкция по настройке Asterisk как SIP-прокси между Mango Office и ElevenLabs

## Обзор решения

Эта инструкция покажет, как развернуть Asterisk на сервере в роли B2BUA (Back-to-Back User Agent) для проксирования голосовых звонков между российским провайдером Mango Office и AI-сервисом ElevenLabs.

**Схема работы:**
```
[Клиент] ←→ [Mango Office] ←→ [Ваш сервер с Asterisk] ←→ [ElevenLabs AI]
```

**Стоимость:** SIP Trunk в Mango Office стоит 450 рублей в месяц.

## Требования

### Технические требования
- **Сервер:** VPS/VDS с публичным IP, расположенный ВНЕ России (рекомендуется Нидерланды, Германия)
- **Ресурсы:** минимум 1 CPU / 2GB RAM, рекомендуется 2 CPU / 2GB RAM
- **ОС:** Ubuntu 20.04+ или Debian 11+
- **Сеть:** статический публичный IP-адрес
- **Домен:** желательно (например, `sip.yourdomain.com`)

### Сервисы
- Активная учетная запись в **Mango Office** с возможностью создания SIP Trunk
- Настроенный **ElevenLabs Conversational AI** с SIP-доступом
- Телефонный номер, импортированный в ElevenLabs

## Шаг 1: Подготовка сервера

### 1.1 Подключение и обновление
```bash
# Подключитесь к серверу по SSH
ssh root@YOUR_SERVER_IP

# Обновите систему
sudo apt update && sudo apt upgrade -y
```

### 1.2 Настройка файрвола (КРИТИЧЕСКИ ВАЖНО!)
```bash
# Разрешаем SSH (чтобы не потерять доступ)
sudo ufw allow ssh

# Открываем порты для SIP
sudo ufw allow 5060/tcp    # SIP сигнализация
sudo ufw allow 5061/tcp    # SIP через TLS
sudo ufw allow 5060/udp    # SIP UDP (для совместимости)

# Открываем порты для RTP (аудио)
sudo ufw allow 10000:20000/udp

# Включаем файрвол
sudo ufw enable
```

### 1.3 Защита от брутфорс-атак
⚠️ **ВАЖНО:** Одна из главных проблем - постоянные SSH-атаки, которые могут нарушить работу Asterisk.

```bash
# Устанавливаем fail2ban для защиты от атак
sudo apt install fail2ban -y

# Настраиваем fail2ban
sudo nano /etc/fail2ban/jail.local
```

Добавьте в файл:
```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
```

```bash
# Перезапускаем fail2ban
sudo systemctl restart fail2ban
```

## Шаг 2: Установка Asterisk

### 2.1 Добавление официального репозитория Asterisk
```bash
# Устанавливаем утилиты
sudo apt install -y curl gnupg ca-certificates

# Добавляем ключ репозитория
curl -fsSL https://packages.asterisk.org/keys/DEB-GPG-KEY-asterisk | sudo gpg --dearmor -o /usr/share/keyrings/asterisk-archive-keyring.gpg

# Добавляем репозиторий
echo "deb [signed-by=/usr/share/keyrings/asterisk-archive-keyring.gpg] http://packages.asterisk.org/deb bullseye main" | sudo tee /etc/apt/sources.list.d/asterisk.list

# Обновляем список пакетов
sudo apt update
```

### 2.2 Установка Asterisk
```bash
# Устанавливаем Asterisk
sudo apt install -y asterisk

# Проверяем статус
sudo systemctl status asterisk

# Включаем автозапуск
sudo systemctl enable asterisk
```

## Шаг 3: Получение учетных данных от Mango Office

### 3.1 Создание SIP Trunk в Mango Office

1. Войдите в личный кабинет Mango Office
2. Перейдите в раздел **"Настройка SIP"** → **"SIP транки"**
3. Нажмите **"Добавить SIP транк"**
4. Заполните параметры:
   - **Название:** например, "AI Proxy"
   - **IP-адрес:** публичный IP вашего сервера
   - **Порт:** 5060
   - **Режим работы:** TCP
   - **Команды DTMF:** RFC2833
   - **Голосовые кодеки:** только G.711 A-law
   - **Количество линий:** 5

### 3.2 Получение учетных данных

После создания SIP Trunk перейдите в раздел **"Учетные записи и домены SIP"**.

Там вы увидите примерно такую информацию:
```
Пользователь: eleven
Пароль: ••••••••••• (нажмите на глаз, чтобы увидеть)
Домен: vpbx400354196.mango-office.ru
```

**Запишите эти данные - они понадобятся для конфигурации!**

### 3.3 Настройка маршрутизации звонков

1. Перейдите в **"Обработка входящих"** → **"Схемы вызовов"**
2. Выберите нужный телефонный номер
3. В настройках маршрутизации выберите ваш созданный SIP транк
4. Сохраните изменения

## Шаг 4: Получение данных от ElevenLabs

### 4.1 Настройка SIP в ElevenLabs

1. Войдите в ElevenLabs
2. Перейдите в **Conversational AI** → **Phone Numbers**
3. Нажмите **"Import number"** → **"From SIP trunk"**
4. Введите ваш номер в формате E.164 (например: `+74951234567`)

### 4.2 Настройка Outbound Configuration

В настройках импорта номера:
- **Transport:** выберите `sip:sip.rtc.elevenlabs.io:5061;transport=tls`
- **Address:** укажите публичный IP или домен вашего сервера
- **Authentication:** Digest Authentication
- **Username:** придумайте логин (например: `asterisk_user`)
- **Password:** придумайте надежный пароль

**Запишите эти учетные данные!**

## Шаг 5: Конфигурация Asterisk

### 5.1 Создание файла pjsip.conf

```bash
sudo nano /etc/asterisk/pjsip.conf
```

Вставьте следующую конфигурацию, заменив значения в ЗАГЛАВНЫХ БУКВАХ на ваши реальные данные:

```ini
; ===== ГЛОБАЛЬНЫЕ НАСТРОЙКИ =====
[global]
type=global
user_agent=MyAsteriskProxy
endpoint_identifier_order=ip,username,anonymous

; ===== ТРАНСПОРТЫ =====
[transport-tcp]
type=transport
protocol=tcp
bind=0.0.0.0:5060
external_signaling_address=146.103.122.8
external_media_address=146.103.122.8
local_net=192.168.0.0/16
local_net=10.0.0.0/8
local_net=172.16.0.0/12

[transport-tls]
type=transport
protocol=tls
bind=0.0.0.0:5061
method=tlsv1_2
external_signaling_address=146.103.122.8
external_media_address=146.103.122.8
local_net=192.168.0.0/16
local_net=10.0.0.0/8
local_net=172.16.0.0/12
verify_server=no
verify_client=no

[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060
external_signaling_address=146.103.122.8
external_media_address=146.103.122.8
local_net=192.168.0.0/16
local_net=10.0.0.0/8
local_net=172.16.0.0/12

; ===== ПОДКЛЮЧЕНИЕ К MANGO OFFICE =====
[mango-registration]
type=registration
transport=transport-tcp
outbound_auth=mango-auth
; Замените на домен из личного кабинета  — учетные записи и домены SIP
server_uri=sip:vpbx4XXXXX6.mangosip.ru ; ‹- ДОМЕН
client_uri=sip:SIP_NAME@vpbx4XXXXX6.mangosip.ru ; ‹- SIP АДРЕС
retry_interval=60

[mango-auth]
type=auth
auth_type=userpass
; Замените на ваши данные из Mango Office
username=eleven
password=YOUR_MANGO_PASSWORD

[mango-trunk-aor]
type=aor
remove_existing=yes
maximum_expiration=3600
qualify_frequency=60

[mango-trunk]
type=endpoint
transport=transport-tcp
context=from-mango
disallow=all
allow=alaw,ulaw
outbound_auth=mango-auth
aors=mango-trunk-aor
direct_media=no
force_rport=yes
rtp_symmetric=yes
rewrite_contact=yes
rtp_timeout=8            ; если 8 сек нет RTP — рвём
rtp_timeout_hold=12
rtp_keepalive=1          ; посылаем пустышки, чтобы NAT не засыпал
timers=yes
timers_min_se=90
timers_sess_expires=1800

; Идентификация входящих звонков от Mango
[mango-identify]
type=identify
endpoint=mango-trunk
; сигнализация Mango (SIP/TCP/UDP 5060 и 60000)
match=81.88.86.0/24
; медиасерверы Mango (RTP/UDP 1024-65535)
match=81.88.88.0/24

; ===== ПОДКЛЮЧЕНИЕ К ELEVENLABS =====
[elevenlabs-auth]
type=auth
auth_type=userpass
; Замените на ваши данные для ElevenLabs
username=YOUR_ELEVENLABS_USER
password=YOUR_ELEVENLABS_PASSWORD

[elevenlabs-aor]
type=aor
contact=sip:sip.rtc.elevenlabs.io:5061;transport=tls ; ‹- Самая • важная строка

[elevenlabs-trunk]
type=endpoint
transport=transport-tls
aors=elevenlabs-aor
context=from-elevenlabs
disallow=all
allow=alaw,ulaw
outbound_auth=elevenlabs-auth
direct_media=no
force_rport=yes
rtp_symmetric=yes
; Явно указываем ElevenLabs как прокси-сервер для всех исходящих вызовов
outbound_proxy=sip:sip.rtc.elevenlabs.io:5061\;transport=tls
rtp_timeout=8
rtp_timeout_hold=12
rtp_keepalive=1
timers=yes
timers_min_se=90
timers_sess_expires=1800

```

### 5.2 Создание файла extensions.conf

```bash
sudo nano /etc/asterisk/extensions.conf
```

```ini
[globals]
; Замените на ваш номер в формате E.164
MY_NUMBER=+74951234567

[general]
autofallthrough=yes

; ===== ОБРАБОТКА ЗВОНКОВ ОТ MANGO =====
[from-mango]
; Любой входящий номер (_X. означает "любой номер")
exten => _X.,1,NoOp(== Входящий звонок от Mango на номер ${EXTEN} ==)
 same => n,Dial(PJSIP/${MY_NUMBER}@elevenlabs-trunk, 90)
 same => n,Hangup()

; ===== ОБРАБОТКА ЗВОНКОВ ОТ ELEVENLABS =====
[from-elevenlabs]
; Если AI будет инициировать исходящие звонки
exten => _X.,1,NoOp(== Исходящий звонок от ElevenLabs на номер ${EXTEN} ==)
 same => n,Dial(PJSIP/${EXTEN}@mango-trunk, 90)
 same => n,Hangup()
```

### 5.3 Создание файла rtp.conf

```bash
sudo nano /etc/asterisk/rtp.conf
```

```ini
[general]
; Диапазон портов для передачи голоса
rtpstart=10000
rtpend=20000
```

## Шаг 6: Запуск и тестирование

### 6.1 Перезапуск Asterisk
```bash
# Перезапускаем Asterisk с новой конфигурацией
sudo systemctl restart asterisk

# Проверяем статус
sudo systemctl status asterisk
```

### 6.2 Проверка регистрации
```bash
# Входим в консоль Asterisk
sudo asterisk -rvvv

# Проверяем регистрацию в Mango Office
pjsip show registrations
```

**Результат должен быть:**
```
<Registration/ServerURI>                              <Status>
mango-registration/sip:vpbx400354196.mango-office.ru  Registered
```

### 6.3 Тестовый звонок
1. Позвоните на ваш номер в Mango Office с любого телефона
2. В консоли Asterisk вы должны увидеть логи обработки звонка
3. Звонок должен переадресоваться на ElevenLabs AI

## Шаг 7: Типичные проблемы и решения

### Проблема: Статус регистрации "Rejected"
**Причина:** Неверные учетные данные или домен
**Решение:** 
- Проверьте логин, пароль и домен в файле `pjsip.conf`
- Убедитесь, что нет лишних пробелов или опечаток

### Проблема: "No matching endpoint found"
**Причина:** Asterisk не знает, как обрабатывать входящие звонки от Mango
**Решение:** 
- Добавьте секцию `[mango-identify]` в `pjsip.conf`
- Проверьте IP-адрес Mango в логах и обновите параметр `match`

### Проблема: Звонок доходит, но нет звука
**Причина:** Проблемы с RTP (файрвол или NAT)
**Решение:**
- Убедитесь, что порты 10000-20000/udp открыты
- Проверьте настройки `external_signaling_address` и `external_media_address`

### Проблема: Зависание сервисов
**Причина:** SSH-атаки или высокая нагрузка
**Решение:**
- Установите fail2ban
- Заблокируйте подозрительные IP через `sudo ufw deny from IP_ADDRESS`

## Шаг 8: Мониторинг и обслуживание

### Полезные команды для отладки
```bash
# Просмотр логов Asterisk
sudo journalctl -u asterisk -f

# Просмотр статуса SIP-транков
sudo asterisk -rvvv
pjsip show endpoints

# Просмотр активных звонков
core show channels

# Перезагрузка конфигурации без перезапуска
module reload res_pjsip.so
```

### Логирование
Для детального логирования SIP-трафика добавьте в консоль Asterisk:
```
pjsip set logger on
```

## Заключение

После выполнения всех шагов у вас будет работающий SIP-прокси, который:
- Принимает звонки от Mango Office
- Переадресует их на ElevenLabs AI
- Обеспечивает двустороннюю передачу голоса

**Стоимость решения:** 450 рублей/месяц за SIP Trunk в Mango Office + стоимость VPS.

Система готова к продакшену и может обрабатывать несколько одновременных звонков.
