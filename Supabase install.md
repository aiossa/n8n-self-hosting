# Устанавливаем docker compose plugin

## Обновляем package index
sudo apt update

## Добавляем Docker's GPG key
```
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

## Добавляем Docker repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Обновляем package index
```
sudo apt update
```

## Устанавливаем Docker Compose plugin
```
sudo apt install docker-compose-plugin
```

# Устанавливаем supabase

## Клонируем репозиторий с гитхаба
```
git clone --depth 1 https://github.com/supabase/supabase
```
## Создаём собственную supabase папку для проекта
```
mkdir supabase-project
```
## Копируем конфигурационные файлы дл docker compose файлы для вашего проекта
```
cp -rf supabase/docker/* supabase-project
```

## Либо используйте конфигурационный файл по умолчанию и редактируйте его, либо используйте конфигурационный файл из этого туториала:

### Копируем предлагаемые переменные среды env
```
cp supabase/docker/.env.example supabase-project/.env
```

ИЛИ
## Обновляе переменные ручками (рекоммендую)
[ссылка]([url](https://github.com/aiossa/n8n-self-hosting/blob/main/supabase%20env%20file)) на конфиг
```
nano supabase-project/.env
```

# Настраиваем nginx
## Получаем сертификаты для supabase
```
sudo certbot --nginx -d your-domain.com
```

## Редактируем настройки nginx'а
[ссылка]([url](https://github.com/aiossa/n8n-self-hosting/blob/main/supabase_n8n_nginx_config))
```
rm /etc/nginx/sites-available/n8n.conf 
sudo nano /etc/nginx/sites-available/n8n.conf 
```
## Удаляем стандартный конфиг nginx
```
sudo rm /etc/nginx/sites-enabled/default
```

## Тестируем конфирурации Nginx и перезапускаем
```
sudo nginx -t
sudo systemctl restart nginx
```

# Переключаемся в наш репозиторий и запускаем supabase
```
cd supabase-project
```
# Загружаем последние образы контейнеров
```
docker compose pull
```

# Соединяем сети
```
docker network ls
docker network connect supabase_default n8n
```
# Start the services (in detached mode)
```
docker compose up -d
```
