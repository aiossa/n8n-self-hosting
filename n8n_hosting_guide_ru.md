# Самостоятельный хостинг n8n с SSL на Linux-сервере

Пошаговое развёртывание [n8n](https://n8n.io) — открытого инструмента автоматизации рабочих процессов — на своём Linux-сервере: Docker для запуска, Nginx как обратный прокси, Certbot для бесплатного SSL-сертификата от Let's Encrypt.

Актуально для n8n 2.x (проверено на версии 2.36.8, август 2026).

---

## Что понадобится

| Требование | Значение |
|---|---|
| Сервер | Ubuntu 22.04 / 24.04 или Debian 12, минимум 1 vCPU и 2 ГБ RAM |
| Домен | A-запись, указывающая на IP сервера (`n8n.example.ru` → `1.2.3.4`) |
| Доступ | root или пользователь с `sudo` |
| Порты | открыты 80 и 443 |

DNS-запись должна успеть распространиться до шага с Certbot, иначе сертификат не выпустится. Проверить: `dig +short n8n.example.ru`.

Дальше по тексту вместо `n8n.example.ru` подставляйте свой домен — он встречается в четырёх местах: две переменные окружения, `server_name` в Nginx и аргумент Certbot.

---

## Шаг 1. Установка Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Проверьте, что Docker поднялся:

```bash
sudo docker version
```

---

## Шаг 2. Установка редактора nano

Дальше понадобится создавать и править файлы конфигурации прямо на сервере. Проще всего это делать в редакторе `nano`:

```bash
sudo apt install -y nano
```

Как им пользоваться: открываете файл командой `sudo nano путь/к/файлу`, набираете текст, затем `Ctrl+O` и `Enter` — сохранить, `Ctrl+X` — выйти.

---

## Шаг 3. Каталог и переменные окружения

1. **Каталог данных.** Здесь n8n будет хранить воркфлоу и настройки:
   ```bash
   sudo mkdir -p /opt/n8n-data
   sudo chown -R 1000:1000 /opt/n8n-data
   sudo chmod 700 /opt/n8n-data
   ```
   `1000:1000` — пользователь `node` внутри контейнера n8n.

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

   Ограничьте доступ к файлу:
   ```bash
   sudo chmod 600 /opt/n8n.env
   ```

   Две переменные легко упустить:

   - **`N8N_WEBHOOK_URL`** — публичный адрес для вебхуков. В инструкциях для n8n 1.x она называлась `WEBHOOK_URL`, это имя устарело.
   - **`N8N_PROXY_HOPS=1`** — нужна, когда n8n работает за Nginx. Без неё не заработают OAuth-подключения вроде Google.

---

## Шаг 4. Запуск n8n

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

Проверьте, что контейнер запустился:

```bash
sudo docker ps
sudo docker logs -f n8n
```

---

## Шаг 5. Установка и настройка Nginx

Nginx принимает запросы из интернета по HTTPS и передаёт их в n8n.

1. **Установка:**
   ```bash
   sudo apt install -y nginx
   ```

2. **Конфигурация.** Создайте файл:
   ```bash
   sudo nano /etc/nginx/sites-available/n8n.conf
   ```

   ```nginx
   server {
       listen 80;
       server_name n8n.example.ru;

       # Загрузка файлов в воркфлоу: без этой строки файлы больше 1 МБ не пройдут
       client_max_body_size 100m;

       location / {
           proxy_pass http://127.0.0.1:5678;
           proxy_http_version 1.1;

           # Без этих двух строк редактор не показывает ход выполнения
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";

           proxy_set_header Host              $host;
           proxy_set_header X-Real-IP         $remote_addr;
           proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_set_header X-Forwarded-Host  $host;

           # Длинные воркфлоу не должны обрываться по таймауту
           proxy_read_timeout 3600s;
           proxy_send_timeout 3600s;
           proxy_buffering off;
       }
   }
   ```

3. **Активация:**
   ```bash
   sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
   sudo rm -f /etc/nginx/sites-enabled/default
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

## Шаг 6. SSL-сертификат через Certbot

1. **Установка:**
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   ```

2. **Выпуск сертификата:**
   ```bash
   sudo certbot --nginx -d n8n.example.ru
   ```
   Certbot попросит email, согласие с условиями и предложит включить редирект с HTTP на HTTPS — соглашайтесь. Дальше сертификат продлевается автоматически.

Выполняйте шаги по порядку: Certbot сам дописывает в конфиг Nginx блок для HTTPS.

---

## Шаг 7. Проверка

Откройте `https://n8n.example.ru` — откроется форма создания аккаунта владельца.

---

## Обновление n8n

```bash
sudo docker pull n8nio/n8n:2.37.0        # подставьте нужный тег
sudo docker stop n8n
sudo docker rm n8n
```

Затем заново выполните команду `docker run` из шага 4, поменяв в ней тег версии. Воркфлоу и настройки останутся на месте — они лежат в `/opt/n8n-data`.

---

## Резервное копирование

Всё, что нужно сохранить, лежит в `/opt/n8n-data` и `/opt/n8n.env`. Контейнер на время копирования лучше остановить:

```bash
sudo docker stop n8n
sudo tar czf ~/n8n-backup-$(date +%F).tar.gz /opt/n8n-data /opt/n8n.env
sudo docker start n8n
```
