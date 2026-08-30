# Самостоятельный хостинг n8n с SSL на Linux-сервере

Пошаговое развёртывание [n8n](https://n8n.io) — открытого инструмента автоматизации рабочих процессов — на своём Linux-сервере: Docker для запуска, Nginx как обратный прокси, Certbot для бесплатного SSL-сертификата от Let's Encrypt.

**Актуально для n8n 2.x** (проверено на 2.36.8, август 2026). Если вы разворачивали n8n по старым инструкциям для 1.x — загляните в раздел [Что изменилось в n8n 2.0](#что-изменилось-в-n8n-20), там есть ломающие изменения.

---

## Что понадобится

| Требование | Значение |
|---|---|
| Сервер | Ubuntu 22.04 / 24.04 или Debian 12, минимум 2 vCPU и 4 ГБ RAM |
| Домен | A-запись, указывающая на IP сервера (`n8n.example.ru` → `1.2.3.4`) |
| Доступ | root или пользователь с `sudo` |
| Порты наружу | только **80** и **443**. Порт 5678 наружу открывать не нужно и опасно |

DNS-запись должна успеть распространиться **до** шага с Certbot, иначе выпуск сертификата не пройдёт. Проверить: `dig +short n8n.example.ru`.

Дальше по тексту вместо `n8n.example.ru` подставляйте свой домен — он встречается в четырёх местах: две переменные окружения, `server_name` в Nginx и аргумент Certbot.

---

## Шаг 1. Установка Docker

Одной командой — это официальный скрипт Docker, он сам определяет дистрибутив, подключает репозиторий и ставит Docker Engine вместе с плагином `docker compose`:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Проверьте, что всё поднялось:

```bash
sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
```

Пара замечаний:

- Скрипт ставит последнюю стабильную версию и не спрашивает подтверждений. Если не хотите выполнять код из интернета вслепую — сначала посмотрите его: `curl -fsSL https://get.docker.com -o get-docker.sh && less get-docker.sh`, потом `sudo sh get-docker.sh`. Ручной способ с подключением репозитория описан [в документации Docker](https://docs.docker.com/engine/install/ubuntu/).
- `sudo apt install docker.io` тоже поставит Docker, но плагина `docker compose` в этом пакете нет. Отдельный пакет `docker-compose-v2` есть только в Ubuntu 24.04; на Ubuntu 22.04 и Debian 12 его в репозитории нет, и [вариант B](#вариант-b--docker-compose-с-postgresql-рекомендуется) из шага 3 не заработает.

---

## Шаг 2. Подготовка каталога и переменных окружения

n8n хранит настройки и ключ шифрования учётных данных в каталоге `/home/node/.n8n` внутри контейнера. Этот каталог нужно вынести на хост — иначе при пересоздании контейнера все сохранённые креды станут нечитаемыми.

1. **Каталог данных:**
   ```bash
   sudo mkdir -p /opt/n8n-data
   sudo chown -R 1000:1000 /opt/n8n-data
   sudo chmod 700 /opt/n8n-data
   ```
   `1000:1000` — это пользователь `node` внутри образа n8n. Права `700` обязательны: при более широких n8n пишет предупреждение и (с `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`) отказывается стартовать.

   Ключ шифрования учётных данных n8n сгенерирует сам при первом запуске и положит в `/opt/n8n-data/config`. Задавать его руками не нужно — достаточно бэкапить этот каталог вместе с базой.

2. **Файл с переменными окружения.** Создайте `/opt/n8n.env`:
   ```bash
   sudo nano /opt/n8n.env
   ```

   Содержимое (подставьте свой домен):
   ```ini
   N8N_HOST=n8n.example.ru
   N8N_PORT=5678
   N8N_PROTOCOL=https
   N8N_EDITOR_BASE_URL=https://n8n.example.ru/
   N8N_WEBHOOK_URL=https://n8n.example.ru/
   N8N_PROXY_HOPS=1
   N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
   ```

   Ограничьте доступ к файлу — для варианта с PostgreSQL сюда добавится пароль базы:
   ```bash
   sudo chmod 600 /opt/n8n.env
   ```

   Что здесь важно:

   - **`N8N_WEBHOOK_URL`** — публичный адрес для вебхуков. Старое имя `WEBHOOK_URL` объявлено устаревшим в n8n 2.35.0; пока работает, но пишет предупреждение в лог.
   - **`N8N_PROXY_HOPS=1`** — обязательно при работе за одним обратным прокси. По умолчанию `0`, и тогда n8n не доверяет заголовкам `X-Forwarded-*`: ломаются OAuth-редиректы, в логах вместо IP клиента виден IP Nginx, а rate limit считается на всех разом.

   Необязательно, но удобно: если планируете запуски по расписанию, добавьте туда же часовой пояс — иначе Schedule Trigger считает время по UTC и «каждый день в 9:00» сработает в 12:00 по Москве.
   ```ini
   GENERIC_TIMEZONE=Europe/Moscow
   TZ=Europe/Moscow
   ```
   Часовой пояс переключается и в интерфейсе, в настройках инстанса.

---

## Шаг 3. Запуск n8n

Два варианта. Начните с A, если это личный инстанс; выберите B, если планируете нагрузку или бэкапы без остановки сервиса.

### Вариант A — один контейнер на SQLite (просто)

```bash
sudo docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 127.0.0.1:5678:5678 \
  --env-file /opt/n8n.env \
  -v /opt/n8n-data:/home/node/.n8n \
  --user 1000:1000 \
  n8nio/n8n:2.36.8
```

Ключевые моменты:

- **`-p 127.0.0.1:5678:5678`** — контейнер слушает только петлевой интерфейс. Снаружи попасть можно исключительно через Nginx по HTTPS. Если написать просто `-p 5678:5678`, инстанс станет доступен по `http://IP:5678` в обход TLS — логин и данные вебхуков полетят открытым текстом, а Docker сам пропишет правило в iptables в обход `ufw`.
- **Тег версии** (`:2.36.8`) вместо `latest` — иначе очередной `docker pull` может незаметно подтянуть мажорную версию с ломающими изменениями. Актуальный стабильный тег смотрите на [Docker Hub](https://hub.docker.com/r/n8nio/n8n/tags).
- Флаг `-it` вместе с `-d` не нужен — это остаток из примеров для интерактивного запуска.

SQLite подходит для личного инстанса примерно до 1000 выполнений в сутки. Дальше начинаются медленная загрузка редактора и ошибки `database is locked`, а горячий бэкап без остановки контейнера невозможен.

### Вариант B — Docker Compose с PostgreSQL (рекомендуется)

1. **Каталоги и файл compose:**
   ```bash
   sudo mkdir -p /opt/n8n/postgres-data
   cd /opt/n8n
   ```

2. **Пароль базы.** Сгенерируйте:
   ```bash
   openssl rand -hex 24
   ```

   Допишите в конец `/opt/n8n.env` (`sudo nano /opt/n8n.env`):
   ```ini
   DB_TYPE=postgresdb
   DB_POSTGRESDB_HOST=postgres
   DB_POSTGRESDB_PORT=5432
   DB_POSTGRESDB_DATABASE=n8n
   DB_POSTGRESDB_USER=n8n
   DB_POSTGRESDB_PASSWORD=вставьте_сюда_сгенерированный_пароль
   ```

3. **Файл `/opt/n8n/compose.yml`** (`sudo nano /opt/n8n/compose.yml`):
   ```yaml
   services:
     postgres:
       image: postgres:16
       restart: unless-stopped
       environment:
         POSTGRES_DB: n8n
         POSTGRES_USER: n8n
         POSTGRES_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
         PGDATA: /var/lib/postgresql/data/pgdata
       volumes:
         - /opt/n8n/postgres-data:/var/lib/postgresql/data
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U n8n -d n8n"]
         interval: 10s
         timeout: 5s
         retries: 5

     n8n:
       image: n8nio/n8n:2.36.8
       restart: unless-stopped
       user: "1000:1000"
       ports:
         - "127.0.0.1:5678:5678"
       env_file:
         - /opt/n8n.env
       volumes:
         - /opt/n8n-data:/home/node/.n8n
       depends_on:
         postgres:
           condition: service_healthy
   ```

   Если используете Postgres 18, `PGDATA` обязателен: в 18-й версии сменился каталог хранения данных по умолчанию.

4. **Запуск:**
   ```bash
   cd /opt/n8n
   sudo docker compose --env-file /opt/n8n.env up -d
   sudo docker compose logs -f n8n
   ```

Каталог `/opt/n8n-data` нужен и при PostgreSQL — там лежит ключ шифрования учётных данных.

---

## Шаг 4. Установка и настройка Nginx

Nginx принимает HTTPS снаружи, терминирует TLS и проксирует запросы в n8n по HTTP внутри сервера.

1. **Установка:**
   ```bash
   sudo apt install -y nginx
   ```

2. **Конфигурация** `/etc/nginx/sites-available/n8n.conf`:
   ```bash
   sudo nano /etc/nginx/sites-available/n8n.conf
   ```

   ```nginx
   server {
       listen 80;
       server_name n8n.example.ru;

       # Загрузка файлов в воркфлоу. Дефолт Nginx — 1 МБ, любой файл крупнее упадёт с 413.
       client_max_body_size 100m;

       location / {
           proxy_pass http://127.0.0.1:5678;
           proxy_http_version 1.1;

           # WebSocket: без этого редактор не получает живые логи выполнения
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";

           proxy_set_header Host              $host;
           proxy_set_header X-Real-IP         $remote_addr;
           proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_set_header X-Forwarded-Host  $host;

           # Длинные выполнения воркфлоу не должны обрываться по таймауту
           proxy_read_timeout 3600s;
           proxy_send_timeout 3600s;
           proxy_buffering off;
           proxy_cache off;
       }
   }
   ```

   n8n 2.x ожидает все три заголовка — `X-Forwarded-For`, `X-Forwarded-Host`, `X-Forwarded-Proto`. Если пропустить `X-Forwarded-Host`, интерфейс местами начнёт генерировать ссылки на `localhost`.

3. **Активация:**
   ```bash
   sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
   sudo rm -f /etc/nginx/sites-enabled/default
   sudo nginx -t
   sudo systemctl reload nginx
   ```
   Если `/etc/nginx/sites-enabled/` не существует — создайте: `sudo mkdir -p /etc/nginx/sites-enabled/`.

---

## Шаг 5. SSL-сертификат через Certbot

1. **Установка:**
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   ```

2. **Выпуск сертификата:**
   ```bash
   sudo certbot --nginx -d n8n.example.ru
   ```
   Certbot попросит email, согласие с условиями и предложит включить редирект с HTTP на HTTPS — соглашайтесь.

3. **Проверка автопродления:**
   ```bash
   sudo systemctl status certbot.timer
   sudo certbot renew --dry-run
   ```

**Важно:** выполняйте шаги строго по порядку. Certbot сам правит `/etc/nginx/sites-available/n8n.conf` — добавляет `listen 443 ssl`, пути к сертификатам и редирект. Если сначала руками написать 443-й блок, а потом запустить Certbot, конфиг может задвоиться.

---

## Шаг 6. Брандмауэр

Наружу нужны только 80 и 443. Порт 5678 закрыт по определению — контейнер слушает `127.0.0.1`.

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

Учтите: Docker по умолчанию пишет свои правила напрямую в iptables и обходит `ufw`. Именно поэтому привязка к `127.0.0.1` в `docker run` — не украшательство, а единственная надёжная защита порта 5678.

---

## Шаг 7. Проверка

1. Откройте `https://n8n.example.ru` — должна открыться форма создания владельца инстанса. Создайте аккаунт сразу: пока владелец не задан, форма доступна любому, кто знает адрес.
2. Убедитесь, что порт закрыт снаружи — с локальной машины:
   ```bash
   curl -sS -m 5 http://n8n.example.ru:5678/ ; echo "exit=$?"
   ```
   Ожидаемо соединение не устанавливается.
3. Проверьте вебхуки: создайте воркфлоу с Webhook-нодой, в поле Production URL должен быть ваш HTTPS-адрес, а не `localhost:5678`. Если там `localhost` — не подхватились `N8N_WEBHOOK_URL` / `N8N_EDITOR_BASE_URL`.
4. Логи при проблемах:
   ```bash
   sudo docker logs -f n8n
   sudo tail -f /var/log/nginx/error.log
   ```

---

## Обновление n8n

```bash
# Вариант A (docker run)
sudo docker pull n8nio/n8n:2.37.0        # подставьте нужный тег
sudo docker stop n8n && sudo docker rm n8n
# заново выполните команду docker run из шага 3, изменив тег

# Вариант B (compose)
cd /opt/n8n
sudo nano compose.yml                     # поменяйте тег образа
sudo docker compose --env-file /opt/n8n.env pull
sudo docker compose --env-file /opt/n8n.env up -d
```

Перед сменой мажорной версии сделайте бэкап и прочитайте [список ломающих изменений](https://docs.n8n.io/changelog/v20-breaking-changes). Каталог `/opt/n8n-data` при пересоздании контейнера не трогается — все воркфлоу и креды на месте.

---

## Резервное копирование

**Что бэкапить:**

- `/opt/n8n-data` — ключ шифрования и настройки (при SQLite здесь же вся база: `database.sqlite`).
- `/opt/n8n.env` — переменные окружения.
- Дамп PostgreSQL, если используете вариант B.

**SQLite** (контейнер лучше остановить — горячая копия файла может оказаться битой):
```bash
sudo docker stop n8n
sudo tar czf ~/n8n-backup-$(date +%F).tar.gz /opt/n8n-data /opt/n8n.env
sudo docker start n8n
```

**PostgreSQL** (можно на ходу):
```bash
cd /opt/n8n
sudo docker compose exec -T postgres pg_dump -U n8n n8n | gzip > ~/n8n-db-$(date +%F).sql.gz
sudo tar czf ~/n8n-files-$(date +%F).tar.gz /opt/n8n-data /opt/n8n.env
```

Дамп базы без файла `/opt/n8n-data/config` бесполезен: в нём лежит ключ, которым зашифрованы все учётные данные, и без него они не расшифруются. Поэтому бэкапьте каталог и базу вместе.

---

## Что изменилось в n8n 2.0

Если вы ставили n8n по инструкции для версии 1.x, при переходе на 2.x вас ждёт следующее.

| Изменение | Что делать |
|---|---|
| Ноды **Execute Command** и **Local File Trigger** выключены по умолчанию (риск выполнения произвольных команд) | Включать осознанно через `NODES_EXCLUDE` / соответствующие переменные, если они действительно нужны |
| Доступ к системным переменным окружения из Code-ноды заблокирован: `N8N_BLOCK_ENV_ACCESS_IN_NODE=true` | Передавать секреты через креды n8n, а не через `process.env` |
| `N8N_SKIP_AUTH_ON_OAUTH_CALLBACK` теперь `false` | OAuth-callback требует аутентификации; перепроверьте настроенные интеграции |
| Появилось значение по умолчанию у `N8N_RESTRICT_FILE_ACCESS_TO` | Проверьте воркфлоу, читающие файлы с диска |
| `N8N_GIT_NODE_DISABLE_BARE_REPOS=true` по умолчанию | Актуально только при использовании Git-ноды |
| **MySQL и MariaDB больше не поддерживаются** | Мигрировать на PostgreSQL или SQLite **до** обновления |
| Удалён legacy-драйвер SQLite, по умолчанию используется пулинговый | Ничего не требуется, работает быстрее |
| `N8N_RUNNERS_ENABLED` объявлен устаревшим — task runners включены всегда | Убрать переменную из конфигурации; в 1.x она была обязательной |

### Task runners: внутренний и внешний режим

В 2.0 весь код из Code-нод выполняется в task runner. По умолчанию работает **внутренний режим** — рантайм запускается дочерним процессом рядом с n8n. Документация прямо называет его небезопасным by design: код, вырвавшийся из песочницы, получает доступ к учётным данным и окружению n8n. Для личного инстанса это приемлемо, для продакшена — нет.

**Внешний режим** выносит выполнение в отдельный контейнер `n8nio/runners`. Добавьте в `compose.yml`:

```yaml
  runners:
    image: n8nio/runners:2.36.8      # версия должна совпадать с образом n8n
    restart: unless-stopped
    environment:
      N8N_RUNNERS_AUTH_TOKEN: ${N8N_RUNNERS_AUTH_TOKEN}
      N8N_RUNNERS_TASK_BROKER_URI: http://n8n:5679
    depends_on:
      - n8n
```

И в `/opt/n8n.env`:
```
N8N_RUNNERS_MODE=external
N8N_RUNNERS_AUTH_TOKEN=<openssl rand -hex 32>
N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0
```

`N8N_RUNNERS_BROKER_LISTEN_ADDRESS` по умолчанию `127.0.0.1` — для соседнего контейнера этого недостаточно, брокер должен слушать сетевой интерфейс Docker. Токен `N8N_RUNNERS_AUTH_TOKEN` должен совпадать у обоих сервисов. Порт брокера (5679) наружу не публикуется.

---

## Устранение неполадок

| Симптом | Причина и решение |
|---|---|
| В Webhook-ноде URL вида `localhost:5678` | Не заданы `N8N_WEBHOOK_URL` и `N8N_EDITOR_BASE_URL`, либо контейнер не перезапущен после правки `.env` |
| Ошибка **413 Request Entity Too Large** при загрузке файла | Нет `client_max_body_size` в конфиге Nginx |
| Редактор не показывает ход выполнения, в консоли ошибки WebSocket | В `location /` отсутствуют заголовки `Upgrade` / `Connection` |
| OAuth-подключения (Google, Telegram и др.) не завершаются | Не задан `N8N_PROXY_HOPS=1`, либо адрес в `N8N_EDITOR_BASE_URL` не совпадает с redirect URI в настройках приложения |
| В логе `Permissions 0644 for n8n settings file are too wide` | `sudo chmod 600 /opt/n8n-data/config` |
| Контейнер перезапускается по кругу | `sudo docker logs n8n` — чаще всего проблема с правами на `/opt/n8n-data` или недоступная база |
| `nginx -t` ругается на дубликат `server_name` | Не удалён `/etc/nginx/sites-enabled/default` либо конфиг задвоился после Certbot |
| Certbot: «Timeout during connect» | A-запись не указывает на сервер или порт 80 закрыт брандмауэром |

---

## Зачем здесь Nginx и Certbot

**Nginx** принимает соединения из интернета, терминирует TLS и передаёт запросы в n8n по внутреннему HTTP. Сам n8n не занимается сертификатами, а держать его открытым в интернет напрямую небезопасно.

**Certbot** — утилита от Electronic Frontier Foundation, автоматизирующая выпуск и продление бесплатных сертификатов Let's Encrypt. После установки продление выполняется системным таймером без вашего участия.

---

## Полезные ссылки

- [Установка n8n через Docker](https://docs.n8n.io/deploy/host-n8n/install-options/install-with-docker/) — официальная документация
- [Установка через Docker Compose](https://docs.n8n.io/deploy/host-n8n/install-options/install-using-docker-compose/)
- [Настройка webhook URL за обратным прокси](https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/configuration-examples/configure-webhook-urls-with-reverse-proxy/)
- [Ломающие изменения версии 2.0](https://docs.n8n.io/changelog/v20-breaking-changes)
- [Настройка task runners](https://docs.n8n.io/deploy/host-n8n/configure-n8n/set-up-task-runners/)
- [Теги образа на Docker Hub](https://hub.docker.com/r/n8nio/n8n/tags)
