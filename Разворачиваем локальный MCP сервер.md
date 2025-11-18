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
  --network supabase_default \
  -e BRAVE_API_KEY=API_KEY \
  -e BRAVE_MCP_TRANSPORT="http" \
  --name brave-search-mcp \
  mcp/brave-search
```

## 4. Указываем настройки в ноде
  <img width="602" height="521" alt="image" src="https://github.com/user-attachments/assets/6ab6cd56-9ad7-4e3c-8e0e-22389f6f69f7" />
