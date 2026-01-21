# Plane v1.2.1 - Docker Swarm / Dokploy

Self-hosted [Plane](https://plane.so) — управление проектами с MCP сервером для интеграции с AI.

## Сервисы

| Сервис | Описание |
|--------|----------|
| web | Веб-интерфейс |
| space | Публичные страницы |
| admin | Админ-панель |
| live | Real-time обновления |
| api | Backend API |
| worker | Фоновые задачи |
| beat-worker | Планировщик задач |
| migrator | Миграции БД |
| plane-db | PostgreSQL 15 |
| plane-redis | Valkey 7.2 |
| plane-mq | RabbitMQ 3.13 |
| plane-minio | MinIO (S3 хранилище) |
| plane-mcp | MCP сервер для AI |
| proxy | Reverse proxy |

## Быстрый старт

```bash
# 1. Скопировать шаблон окружения
cp .env.example .env

# 2. Заполнить .env своими значениями:
# - APP_DOMAIN
# - Пароли (POSTGRES_PASSWORD, RABBITMQ_PASSWORD, и т.д.)
# - SECRET_KEY, LIVE_SERVER_SECRET_KEY

# 3. Деплой в Swarm
docker stack deploy -c docker-compose.yml plane

# 4. Проверить статус
docker stack services plane
```

## Деплой в Dokploy

1. Создать новый **Compose** проект в Dokploy
2. Вставить содержимое `docker-compose.yml`
3. Добавить переменные окружения из `.env`
4. Задеплоить
5. Привязать домен через UI Dokploy (SSL настроится автоматически)

## MCP Сервер

После запуска Plane настроить MCP:

1. Получить API ключ: **Profile → Settings → API Tokens**
2. Указать в `.env`:
   ```
   PLANE_API_KEY=ваш-api-ключ
   PLANE_WORKSPACE_SLUG=ваш-workspace
   ```
3. MCP endpoint: `http://ваш-домен:8001/sse`

## Масштабирование

Настройка реплик в `.env`:
```
API_REPLICAS=2
WORKER_REPLICAS=3
WEB_REPLICAS=2
```

## Порты

| Порт | Сервис |
|------|--------|
| 80 | HTTP (proxy) |
| 443 | HTTPS (proxy) |
| 8001 | MCP Server |

## Volumes

- `pgdata` — данные PostgreSQL
- `redisdata` — данные Redis
- `uploads` — файлы MinIO
- `rabbitmq_data` — данные RabbitMQ
- `proxy_config`, `proxy_data` — конфиг proxy
- `logs_*` — логи приложений
