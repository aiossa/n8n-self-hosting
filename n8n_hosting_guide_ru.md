# Самостоятельный хостинг N8N с поддержкой SSL на Linux сервере

Данное руководство содержит пошаговые инструкции по самостоятельному развертыванию [n8n](https://n8n.io) — бесплатного инструмента автоматизации рабочих процессов с открытым исходным кодом — на Linux сервере с использованием Docker, Nginx и Certbot для SSL с пользовательским доменным именем.

## Шаг 1: Установка Docker

1. **Обновление индекса пакетов:**
   ```bash
   sudo apt update
   ```

2. **Установка Docker:**
   ```bash
   sudo apt install docker.io
   ```

3. **Запуск Docker:**
   ```bash
   sudo systemctl start docker
   ```

4. **Включение автозапуска Docker при загрузке системы:**
   ```bash
   sudo systemctl enable docker
   ```

5. **Подготовка папок:**
   ```bash
   # Создание директории .n8n в домашней папке
   mkdir -p ~/.n8n

   # Установка владельца для пользователя ID 1000 (соответствует пользователю контейнера)
   sudo chown 1000:1000 ~/.n8n

   # Установка правильных разрешений (чтение/запись/выполнение для владельца, чтение/выполнение для группы)
   chmod 755 ~/.n8n
   ```

## Шаг 2: Запуск n8n в Docker

Выполните следующую команду для запуска n8n в Docker. Замените your-domain.com на ваше фактическое доменное имя:

```bash
sudo docker run -d \
 --name n8n \
 --restart unless-stopped \
 -p 5678:5678 \
 -e N8N_HOST=domain.ru \
 -e N8N_PORT=5678 \
 -e N8N_PROTOCOL=https \
 -e N8N_EDITOR_BASE_URL="https://domain.ru/" \
 -e WEBHOOK_URL=https://domain.ru/ \
 -v ~/.n8n:/home/node/.n8n \
 --user "1000:1000" \
 -it \
 n8nio/n8n
```

Данная команда выполняет следующие действия:

- Загружает и запускает Docker образ n8n
- Открывает n8n на порту 5678
- Устанавливает переменные окружения для хоста n8n и URL туннеля webhook'ов
- Монтирует директорию данных n8n для постоянного хранения
- После выполнения команды n8n будет доступен по адресу your-domain.com:5678

Или если вы используете поддомен, команда должна выглядеть так:

```bash
sudo docker run -d --restart unless-stopped -it \
--name n8n \
-p 5678:5678 \
-e N8N_HOST="subdomain.your-domain.com" \
-e WEBHOOK_TUNNEL_URL="https://subdomain.your-domain.com/" \
-e N8N_EDITOR_BASE_URL="https://subdomain.your-domain.com/" \
-e WEBHOOK_URL="https://subdomain.your-domain.com/" \
-v ~/.n8n:/root/.n8n \
n8nio/n8n
```

## Шаг 3: Установка Nginx

Nginx используется как обратный прокси для перенаправления запросов к n8n и обработки SSL терминации.

1. **Установка Nginx:**
   ```bash
   sudo apt install nginx
   ```

## Шаг 4: Настройка Nginx

Настройте Nginx для работы в качестве обратного прокси для веб-интерфейса n8n:

1. **Создание нового файла конфигурации Nginx:**
   ```bash
   sudo nano /etc/nginx/sites-available/n8n.conf
   ```

2. **Вставьте следующую конфигурацию:**
   ```nginx
   server {
      listen 80;
      server_name n8n.n8n-courses.ru;
  
      location / {
          proxy_pass http://localhost:5678;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_read_timeout  3600s;
          proxy_send_timeout 3600s;
          proxy_buffering off;
      }
  }
   ```
   Замените your-domain.com на ваш фактический домен.

3. **Активация конфигурации:**
   ```bash
   sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
   ```

   Если вы видите ошибку о том, что /etc/nginx/sites-enabled/ не существует, создайте её командой: `sudo mkdir /etc/nginx/sites-enabled/`

4. **Тестирование конфигурации Nginx и перезапуск:**
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

## Шаг 5: Настройка SSL с помощью Certbot

Certbot получит и установит SSL сертификат от Let's Encrypt.

1. **Установка Certbot и плагина Nginx:**
   ```bash
   sudo apt install certbot python3-certbot-nginx
   ```

2. **Получение SSL сертификата:**
   ```bash
   sudo certbot --nginx -d your-domain.com
   // Если у вас поддомен, то это будет subdomain.your-domain.com
   ```

Следуйте инструкциям на экране для завершения настройки SSL.
После завершения n8n будет безопасно доступен по HTTPS по адресу your-domain.com.

**ВАЖНО:** Убедитесь, что вы выполняете вышеуказанные шаги по порядку. Шаг 5 изменит ваш файл /etc/nginx/sites-available/n8n.conf.

## Важные замечания

- Убедитесь, что DNS A-запись вашего домена указывает на IP-адрес вашего сервера
- Разрешите порты 80 (HTTP), 443 (HTTPS) и 5678 (n8n) в брандмауэре вашего сервера
- Nginx обрабатывает SSL терминацию, поэтому он перенаправляет запросы к экземпляру n8n по HTTP внутри системы
