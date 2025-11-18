# Установка локального MCP-сервера

## 1. Создаём сеть
```
docker network create n8n_network
```

## 2. Добавляем в сеть контейнер n8n
```
docker network connect n8n n8n_network
```

## 3. Запускаем MCP сервер, указывая ваш API ключ
```
docker run -d \
  --network n8n_network \
  -e BRAVE_API_KEY=API_KEY \
  -e BRAVE_MCP_TRANSPORT="http" \
  --name brave-search-mcp \
  mcp/brave-search
```

## 4. Указываем настройки в ноде
  <img width="602" height="521" alt="image" src="https://github.com/user-attachments/assets/6ab6cd56-9ad7-4e3c-8e0e-22389f6f69f7" />


# Устанавливаем сервер с puppeter
https://github.com/bytedance/UI-TARS-desktop/tree/main/packages/agent-infra/mcp-servers/browser

## 1. Создаём сеть
Если до этого не была создана сеть, создаём
```
docker network create n8n_network
```

## 2. Добавляем в сеть контейнер n8n
Если до этого контейнер не был добавлен в сеть, добавляем
```
docker network connect n8n n8n_network
```

## 3. Скачиваем сервер с github
```
git clone https://github.com/bytedance/UI-TARS-desktop.git
```

## 4. Переходим в нужную папку
Обратите внимание, если не получается попасть в нужную папку одной командой, перейдите туда постепенно, используя команду cd
```
cd UI-TARS-desktop/packages/agent-infra/mcp-servers/browser
```

## 4. Собираем docker образ
Обратите внимание, может потребоваться несколько минут. И это ресурсоёмкая задача. Ваша виртуальная машина может не справиться
```
docker build -t mcp-server-browser .
```

## 4. Запускаем контейнер, указывая !вашу! network
```
docker run -d --rm --network n8n_network -p 8089:8089 mcp-server-browser --port 8089
```
## 5. Указываем настройки в ноде
  <img width="1586" height="1296" alt="image" src="https://github.com/user-attachments/assets/abb039aa-d6aa-4296-aa39-e075719100a8" />

