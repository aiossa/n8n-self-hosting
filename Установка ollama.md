# Настройка Ollama

Ссылка на документацию
https://ollama.com/blog/ollama-is-now-available-as-an-official-docker-image

## 1. Устанавливаем через докер
Указываем, в какой сети будет доступен контейнер с ollama. Если сети нет, создайте её. 
```
docker run -d --network YOUR_NETWORK_NAME -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

## 2. Добавьте нужную модель
```
docker exec -it ollama ollama pull llama3.2
```
или, например (потребляет меньше оперативной памяти, до 700 мегабайтов)

```
docker exec -it ollama ollama pull tinyllama
```


## 3. Если нужно добавить embeddings

Выберите любую из моделей
```
# nomic-embed-text - легкая и качественная (274 MB, 768 dimensions)
docker exec -it ollama ollama pull nomic-embed-text

# mxbai-embed-large - очень хорошее качество (669 MB, 1024 dimensions)
docker exec -it ollama ollama pull mxbai-embed-large

# all-minilm - самая легкая (46 MB, 384 dimensions)
docker exec -it ollama ollama pull all-minilm

# multilingual-e5 - поддерживает русский язык (274 MB, 768 dimensions)
docker exec -it ollama ollama pull multilingual-e5
```