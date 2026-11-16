# Headscale + Headplane + Tailscale-AWG (Exit Node)

Готовый стек для развёртывания собственной защищённой VPN-сети на базе **Headscale** (открытый координатор Tailscale), веб-панели **Headplane** и выходного узла **Tailscale-AWG** с поддержкой протокола обфускации **AmneziaWG (AWG v2/v3)**.

---

## Архитектура решения

* **Headscale (`hs-awg-headscale`)**: Сервер координации сети Tailnet (порт `8080/tcp` внутри сети Docker, STUN/DERP `3478/udp` на хосте).
* **Headplane (`hs-awg-headplane`)**: Веб-интерфейс администратора для управления пользователями, устройствами, маршрутами и ACL (порт `3000/tcp` внутри сети Docker).
* **Tailscale-AWG (`hs-awg-tailscaled`)**: Контейнер на базе форка `ltlei/tailscale-awg`, работающий как **Exit Node** (шлюз для всего трафика сети) с поддержкой обфускации AmneziaWG (порт `41641/udp`).
* **Nginx / Nginx-UI (Reverse Proxy)**: Внешний прокси, осуществляющий TLS-терминацию (HTTPS на порту `443`), проксирование запросов к Headscale и веб-панели Headplane, а также проброс WebSocket-соединений.

Рекомендуемая схема доменов:
- `hs.yourdomain.com` — публичный URL Headscale и DERP через Nginx.
- `hsp.yourdomain.com` — веб-панель управления Headplane через Nginx.
- `mesh.yourdomain.com` — внутренний суффикс MagicDNS (не требует публичной A-записи в DNS).

*(Также поддерживается вариант на одном домене `vpn.yourdomain.com` с панелью по пути `/admin/`, см. [nginx/locations.conf](file:///opt/headscale-awg/nginx/locations.conf)).*

---

## 1. Требования к серверу

1. **Операционная система**: Linux (Debian 11/12/13, Ubuntu 20.04/22.04/24.04).
2. **Docker & Docker Compose**: установлены Docker Engine и плагин `docker compose` (v2+).
3. **Виртуализация и сеть**:
   - Ядро сервера должно поддерживать `/dev/net/tun` (для OpenVZ/LXC убедитесь, что TUN включен у провайдера).
   - Включен IP-форвардинг в ядре хоста.
4. **Firewall / Сетевые порты**:
   - `80/tcp`, `443/tcp` — Nginx (HTTP/HTTPS)
   - `3478/udp` — Headscale STUN / встроенный DERP
   - `41641/udp` — Tailscale-AWG WireGuard-порт
   *(Внутренние порты 8080, 3000, 9090 публично открывать **не нужно**).*
5. **DNS-записи**:
   - A-запись для `hs.yourdomain.com` -> Публичный IPv4 сервера.
   - A-запись для `hsp.yourdomain.com` -> Публичный IPv4 сервера.
   - *Важно:* Если используется Cloudflare, переключите записи в режим **DNS only** (серое облако, без проксирования Cloudflare CDN).

---

## 2. Подготовка хоста

### 2.1. Включение IP Forwarding

Для работы выходной ноды (Exit Node) хост должен маршрутизировать транзитные пакеты:

```bash
sudo tee /etc/sysctl.d/99-headscale-tailscale.conf << 'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

sudo sysctl -p /etc/sysctl.d/99-headscale-tailscale.conf
```

Проверьте доступность устройства TUN:
```bash
test -c /dev/net/tun && echo 'TUN доступен' || echo 'Ошибка: /dev/net/tun недоступен'
```

### 2.2. Создание внешней Docker-сети для Nginx

Если Nginx работает в Docker-контейнере (например, `nginx-ui`), убедитесь, что создана внешняя bridge-сеть `nginx` и прокси подключен к ней:

```bash
docker network create nginx 2>/dev/null || true
```

---

## 3. Установка на новом сервере

### Шаг 1: Клонирование репозитория

```bash
git clone <URL_ВАШЕГО_РЕПОЗИТОРИЯ> /opt/headscale-awg
cd /opt/headscale-awg
```

### Шаг 2: Создание каталогов данных и локальных секретов

Создайте директории для постоянного хранения данных и сгенерируйте секрет сессий Headplane:

```bash
mkdir -p data/headscale data/headplane data/tailscale-state secrets
umask 077
openssl rand -hex 16 | tr -d '\n' > secrets/headplane-cookie
chmod 600 secrets/headplane-cookie
```

> [!NOTE]
> Файлы в каталогах `data/` и `secrets/` содержат приватные ключи шифрования и базы данных. Они добавлены в `.gitignore` и никогда не должны попадать в публичный репозиторий.

### Шаг 3: Настройка конфигурации Headscale

Скопируйте шаблон конфигурации (если `headscale/config.yaml` отсутствует):

```bash
cp headscale/config.example.yaml headscale/config.yaml
```

Откройте `headscale/config.yaml` и настройте ключевые параметры:
1. `server_url`: установите публичный URL вашего Headscale (например, `https://hs.yourdomain.com`).
2. `base_domain`: MagicDNS суффикс (например, `mesh.yourdomain.com`).
3. В секции `derp.server`:
   - Если используете встроенный DERP-сервер, укажите ваш публичный IPv4:
     ```yaml
     derp:
       server:
         enabled: true # или false, если используете публичные DERP
         ipv4: YOUR_PUBLIC_IPV4
     ```

### Шаг 4: Настройка конфигурации Headplane

Скопируйте шаблон конфигурации:

```bash
cp headplane/config.example.yaml headplane/config.yaml
```

В `headplane/config.yaml` укажите:
1. `server.base_url`: `https://hsp.yourdomain.com`
2. `headscale.public_url`: `https://hs.yourdomain.com`
3. `headscale.url`: `http://headscale:8080` (внутреннее имя сервиса в Docker)
4. Параметр `headscale.api_key` временно оставьте пустым — ключ будет сгенерирован после первого старта Headscale.

### Шаг 5: Настройка переменных окружения (.env)

Скопируйте шаблон `.env.example`:

```bash
cp .env.example .env
```

В файле `.env` задаются параметры для автоматического подключения Exit Node. На этапе первичного развертывания можно оставить `TS_AUTHKEY` пустым:

```env
TS_AUTHKEY=
TS_EXTRA_ARGS=--login-server=https://hs.yourdomain.com --advertise-exit-node --reset
```

---

## 4. Настройка Nginx (Reverse Proxy)

Nginx осуществляет TLS-терминацию и проксирует трафик в контейнеры `hs-awg-headscale` и `hs-awg-headplane`.

### Вариант А (рекомендуемый): Два поддомена (`hs.` и `hsp.`)

Используйте готовый шаблон [nginx/nginx-subdomains.conf.example](file:///opt/headscale-awg/nginx/nginx-subdomains.conf.example).

Основные блоки для Nginx:

```nginx
# --- 1. Headscale (hs.yourdomain.com) ---
server {
    listen 443 ssl http2;
    server_name hs.yourdomain.com;
    
    # SSL-сертификаты
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    location / {
        proxy_pass http://hs-awg-headscale:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header Host $host;
        proxy_set_header True-Client-IP $remote_addr;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}

# --- 2. Headplane (hsp.yourdomain.com) ---
server {
    listen 443 ssl http2;
    server_name hsp.yourdomain.com;

    # SSL-сертификаты
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    location = / {
        return 302 /admin/;
    }

    location / {
        proxy_pass http://hs-awg-headplane:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

### Вариант Б: Единый домен (`vpn.yourdomain.com`)

Если у вас один домен, добавьте директивы из [nginx/locations.conf](file:///opt/headscale-awg/nginx/locations.conf) внутрь блока `server` вашего домена:
- Путь `/admin/` перенаправляется на `http://hs-awg-headplane:3000`.
- Путь `/` перенаправляется на `http://hs-awg-headscale:8080`.

После настройки примените конфигурацию Nginx:
```bash
nginx -t && nginx -s reload
# Или перезагрузите конфигурацию через интерфейс nginx-ui
```

---

## 5. Запуск Headscale и Headplane

1. Проверьте валидность файла docker-compose:
   ```bash
   docker compose config -q
   ```

2. Запустите службы Headscale и Headplane:
   ```bash
   docker compose pull headscale headplane
   docker compose up -d headscale headplane
   docker compose ps
   ```

3. Проверьте доступность Headscale по HTTPS:
   ```bash
   curl -fsS https://hs.yourdomain.com/health
   ```

4. Создайте первого пользователя в Headscale:
   ```bash
   docker compose exec headscale headscale users create admin
   ```

5. Сгенерируйте API-ключ для Headplane:
   ```bash
   docker compose exec headscale headscale apikeys create --expiration 90d
   ```
   *Скопируйте выданный ключ (`hskey-api-...`).*

6. Вставьте ключ в `headplane/config.yaml` в секцию `headscale.api_key`:
   ```yaml
   headscale:
     url: "http://headscale:8080"
     public_url: "https://hs.yourdomain.com"
     api_key: "hskey-api-ВАШ_КЛЮЧ"
   ```

7. Перезапустите Headplane для применения ключа:
   ```bash
   docker compose restart headplane
   ```

8. Откройте `https://hsp.yourdomain.com` (или `https://vpn.yourdomain.com/admin/`). Вы должны увидеть панель управления Headplane.

---

## 6. Запуск и регистрация Exit Node (Tailscale-AWG)

1. Сгенерируйте постоянный pre-auth ключ для Exit Node:
   ```bash
   docker compose exec headscale headscale preauthkeys create --user admin --reusable --expiration 24h
   ```
   *Скопируйте ключ вида `hskey-auth-...`.*

2. Добавьте его в файл `.env`:
   ```bash
   echo "TS_AUTHKEY=hskey-auth-ВАШ_КЛЮЧ" > .env
   echo "TS_EXTRA_ARGS=--login-server=https://hs.yourdomain.com --advertise-exit-node --reset" >> .env
   ```

3. Запустите контейнер Tailscale:
   ```bash
   docker compose up -d tailscaled
   docker compose logs --tail=50 tailscaled
   ```

4. Проверьте статус подключения:
   ```bash
   docker compose exec tailscaled tailscale status
   ```

5. **Одобрите маршруты Exit Node в Headscale**:
   Получите ID созданной ноды:
   ```bash
   docker compose exec headscale headscale nodes list
   ```
   Одобрите маршрутизацию трафика `0.0.0.0/0` и `::/0`:
   ```bash
   docker compose exec headscale headscale nodes approve-routes --identifier <ID_НОДЫ> --routes 0.0.0.0/0,::/0
   ```

6. Убедитесь, что маршруты одобрены:
   ```bash
   docker compose exec headscale headscale nodes list-routes
   ```

---

## 7. Включение обфускации AmneziaWG (AWG v3)

Контейнер `ltlei/tailscale-awg` поддерживает протокол AmneziaWG (AWG v3) для маскировки WireGuard-трафика от DPI-блокировок.

1. Запустите генерацию параметров AWG:
   ```bash
   docker compose exec tailscaled tailscale awg set --no-restart
   ```
   В интерактивном диалоге выберите версию **AWG v3**. Будут сгенерированы параметры обфускации (`Jc`, `Jmin`, `Jmax`, `S1`, `S2`, `H1-H4`).

2. Получите созданный профиль:
   ```bash
   docker compose exec tailscaled tailscale awg get
   ```
   > [!IMPORTANT]
   > Сохраните полученные параметры обфускации. Клиентские устройства, использующие этот форк, должны иметь идентичные параметры AWG для успешного соединения.

3. Перезапустите узел и проверьте статус обфускации:
   ```bash
   docker compose restart tailscaled
   docker compose exec tailscaled tailscale awg status
   docker compose exec tailscaled tailscale awg doctor
   ```

---

## 8. Подключение клиентов

Клиенты могут подключаться как через официальное приложение Tailscale (в стандартном режиме WireGuard), так и через совместимые форки Tailscale-AWG (с поддержкой обфускации).

### Подключение через официальный клиент Tailscale:
1. **Авторизация**:
   - В терминале:
     ```bash
     tailscale up --login-server https://hs.yourdomain.com
     ```
   - На Android / iOS / Windows / macOS:
     В настройках аккаунта выберите смену сервера («Change Server») и введите URL `https://hs.yourdomain.com`.
2. Пройдите по выданной ссылке регистрации или сгенерируйте pre-auth ключ:
   ```bash
   docker compose exec headscale headscale preauthkeys create --user admin --expiration 1h
   ```
3. После подключения выберите `awg-exit` в качестве Exit Node («Use exit node») в интерфейсе клиента.
4. Проверьте свой внешний IP (он должен совпадать с публичным IP сервера).

---

## 9. Полезные команды и обслуживание

### Просмотр логов:
```bash
docker compose logs -f headscale
docker compose logs -f headplane
docker compose logs -f tailscaled
```

### Диагностика сети Tailscale:
```bash
docker compose exec tailscaled tailscale status
docker compose exec tailscaled tailscale netcheck
docker compose exec tailscaled tailscale debug derp-map
```

### Управление Headscale:
```bash
# Список пользователей
docker compose exec headscale headscale users list

# Список узлов
docker compose exec headscale headscale nodes list

# Удаление узла
docker compose exec headscale headscale nodes delete --identifier <ID>

# Генерация pre-auth ключа
docker compose exec headscale headscale preauthkeys create --user admin --expiration 24h
```

### Резервное копирование (Backup)
Для полного переноса или сохранения состояния сервера достаточно архивировать следующие каталоги и файлы:
```bash
# Остановите стек перед резервированием SQLite для гарантии целостности:
docker compose stop

tar -czvf headscale-backup-$(date +%Y%m%d).tar.gz \
    headscale/config.yaml \
    headplane/config.yaml \
    docker-compose.yaml \
    .env \
    secrets/ \
    data/headscale/ \
    data/headplane/ \
    data/tailscale-state/

docker compose start
```

> [!CAUTION]
> Никогда не используйте одну директорию `data/tailscale-state` для двух одновременно запущенных узлов. При переносе на новый сервер удалите старый контейнер перед запуском нового.

---

## Структура файлов проекта

```text
/opt/headscale-awg/
├── .env.example                 # Пример переменных окружения для compose
├── .gitignore                   # Исключение секретов, баз данных и ключей
├── README.md                    # Инструкция по установке и эксплуатации
├── docker-compose.yaml          # Описание сервисов Headscale, Headplane, Tailscale-AWG
├── headplane/
│   ├── config.example.yaml      # Пример конфигурации Headplane
│   └── config.yaml              # Рабочий конфиг Headplane (в .gitignore, содержит секреты)
├── headscale/
│   ├── config.example.yaml      # Пример конфигурации Headscale с заполнителями
│   └── config.yaml              # Рабочая конфигурация Headscale (в .gitignore)
├── nginx/
│   ├── locations.conf           # Шаблон location-блоков для единого домена
│   └── nginx-subdomains.conf.example # Пример конфигурации Nginx для двух поддоменов
└── upstream/                    # Исходные файлы upstream для справки
    ├── headplane-v0.8.2.yaml
    ├── headscale-v0.29.0.yaml
    └── tailscale-awg-compose.yaml
```
