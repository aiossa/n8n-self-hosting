# n8n self-hosting

Материалы по самостоятельному развёртыванию [n8n](https://n8n.io) и смежных сервисов на своём сервере. Все инструкции на русском языке.

## Основное руководство

**[Самостоятельный хостинг n8n с SSL на Linux-сервере](n8n_hosting_guide_ru.md)** — полное развёртывание с нуля: Docker, запуск n8n (SQLite или PostgreSQL), Nginx как обратный прокси, бесплатный SSL от Let's Encrypt, брандмауэр, обновление и резервное копирование.

Актуально для **n8n 2.x** (проверено на 2.36.8, август 2026).

### Быстрый старт

Полный порядок действий — в [руководстве](n8n_hosting_guide_ru.md). Здесь только основная команда запуска, чтобы было видно, к чему всё сводится:

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

Три вещи, на которых чаще всего спотыкаются:

- **`-p 127.0.0.1:5678:5678`, а не `-p 5678:5678`.** Иначе n8n доступен по `http://IP:5678` в обход HTTPS, и Docker пропишет правило в iptables мимо `ufw`.
- **`N8N_PROXY_HOPS=1`** в переменных окружения. Без него за Nginx ломаются OAuth-редиректы, а в логах вместо IP клиента виден IP прокси.
- **Тег версии вместо `latest`.** У n8n 2.0 есть ломающие изменения, ловить их случайным `docker pull` не стоит.

## Остальные инструкции

| Инструкция | О чём |
|---|---|
| [Отправка сообщений в Telegram через n8n](telegram_n8n_guide.md) | Автоматическая отправка от личного аккаунта |
| [Установка Supabase](supabase/Supabase%20install.md) | Развёртывание Supabase рядом с n8n, конфиг Nginx и схема БД |
| [Локальный MCP-сервер](Разворачиваем%20локальный%20MCP%20сервер.md) | Запуск MCP-сервера в одной Docker-сети с n8n |
| [Установка Ollama](Установка%20ollama.md) | Локальные модели в Docker для использования из воркфлоу |

## Официальная документация

- [Установка через Docker](https://docs.n8n.io/deploy/host-n8n/install-options/install-with-docker/)
- [Ломающие изменения n8n 2.0](https://docs.n8n.io/changelog/v20-breaking-changes)
- [Переменные окружения](https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/deployment/)
