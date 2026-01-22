# Plane v1.2.1 - Docker Swarm / Dokploy Stack

Self-hosted [Plane](https://plane.so) — управление проектами, оптимизированный для Dokploy с поддержкой Yandex Cloud S3.

## Требования

- **Dokploy** установлен и работает
- **Yandex Cloud S3** bucket создан (или другое S3-совместимое хранилище)
- **Домен** настроен и направлен на сервер

## Сервисы

| Сервис | Описание |
|--------|----------|
| web | Веб-интерфейс |
| space | Публичные страницы |
| admin | Админ-панель (god-mode) |
| live | Real-time обновления |
| api | Backend API |
| worker | Фоновые задачи |
| beat-worker | Планировщик задач |
| migrator | Миграции БД |
| plane-db | PostgreSQL 15 |
| plane-redis | Valkey 7.2 |
| plane-mq | RabbitMQ 3.13 |
| proxy | Caddy reverse proxy |

## Настройка Yandex Cloud S3

1. Перейти в [Yandex Cloud Console](https://console.yandex.cloud/)
2. Создать bucket в Object Storage
3. Создать сервисный аккаунт с ролью `storage.editor`
4. Сгенерировать статический ключ доступа
5. Записать: Access Key ID, Secret Access Key, Bucket Name

Документация: https://yandex.cloud/ru/docs/storage/tools/boto

## Деплой в Dokploy

### 1. Создать проект

1. **Projects** → **Create Project**
2. **Add Docker Compose** (не Application!)
3. **Provider**: Git → `https://github.com/mikey-semy/plane-stack.git`
4. **Compose Type**: выбрать **Stack** (включает Docker Swarm mode)

### 2. Настроить Environment

После первого деплоя запомните **имя стека** (например, `plane-stack-abc123`).

Перейти в **Environment** и добавить:

```env
# Application
APP_RELEASE=v1.2.1
APP_DOMAIN=your-domain.com
WEB_URL=https://your-domain.com
DEBUG=0

# Replicas
WEB_REPLICAS=1
SPACE_REPLICAS=1
ADMIN_REPLICAS=1
API_REPLICAS=1
WORKER_REPLICAS=1
BEAT_WORKER_REPLICAS=1
LIVE_REPLICAS=1

# Security (сгенерируйте свои!)
# openssl rand -base64 48
SECRET_KEY=ваш-сгенерированный-ключ
CORS_ALLOWED_ORIGINS=https://your-domain.com

# openssl rand -base64 24
LIVE_SERVER_SECRET_KEY=ваш-сгенерированный-live-ключ

# Database - ЗАМЕНИТЕ <stack-name> на реальное имя стека!
PGHOST=<stack-name>_plane-db
PGDATABASE=plane
POSTGRES_USER=plane
POSTGRES_PASSWORD=надежный-пароль-бд
POSTGRES_DB=plane
POSTGRES_PORT=5432
PGDATA=/var/lib/postgresql/data
DATABASE_URL=postgresql://plane:надежный-пароль-бд@<stack-name>_plane-db/plane

# Redis - ЗАМЕНИТЕ <stack-name>!
REDIS_HOST=<stack-name>_plane-redis
REDIS_PORT=6379
REDIS_URL=redis://<stack-name>_plane-redis:6379/

# RabbitMQ - ЗАМЕНИТЕ <stack-name>!
RABBITMQ_HOST=<stack-name>_plane-mq
RABBITMQ_PORT=5672
RABBITMQ_USER=plane
RABBITMQ_PASSWORD=надежный-пароль-mq
RABBITMQ_VHOST=plane
AMQP_URL=amqp://plane:надежный-пароль-mq@<stack-name>_plane-mq:5672/plane

# Yandex Cloud S3
USE_MINIO=0
AWS_REGION=ru-central1
AWS_ACCESS_KEY_ID=ваш-yandex-access-key
AWS_SECRET_ACCESS_KEY=ваш-yandex-secret-key
AWS_S3_ENDPOINT_URL=https://storage.yandexcloud.net
AWS_S3_BUCKET_NAME=имя-вашего-bucket
FILE_SIZE_LIMIT=5242880
MINIO_ENDPOINT_SSL=1

# API - ЗАМЕНИТЕ <stack-name>!
API_BASE_URL=http://<stack-name>_api:8000
GUNICORN_WORKERS=1
API_KEY_RATE_LIMIT=60/minute

# Proxy
SITE_ADDRESS=:80
```

### 3. Создать Caddyfile на сервере

SSH на сервер и создать файл:

```bash
mkdir -p /root/plane-caddyfile-fix

cat > /root/plane-caddyfile-fix/Caddyfile << 'EOF'
{
    auto_https off
    admin localhost:2019
}

:80 {
    reverse_proxy / web:3000
    reverse_proxy /api/* api:8000
    reverse_proxy /auth/* api:8000
    reverse_proxy /spaces/* space:3000
    reverse_proxy /god-mode/* admin:3000
    reverse_proxy /live/* live:3000
}
EOF
```

### 4. Настроить домен

1. Перейти в настройки Compose → **Domains**
2. Добавить домен на сервис **proxy**, порт **80**
3. Traefik автоматически настроит SSL через Let's Encrypt

### 5. Redeploy

Нажать **Redeploy** в Dokploy.

## Отладка

### Проверить статус сервисов
```bash
docker service ls | grep plane-stack
```

### Просмотр логов
```bash
docker service logs <stack-name>_api --tail 100
docker service logs <stack-name>_proxy --tail 100
```

### Частые проблемы

1. **"Name does not resolve"** - проверьте что все hostnames в .env используют правильный префикс стека
2. **"Bad Gateway"** - proxy не может достучаться до backend сервисов, проверьте пути в Caddyfile
3. **"Port already allocated"** - уберите привязку портов, пусть Traefik управляет маршрутизацией

## MCP Сервер (опционально)

Для интеграции с AI раскомментируйте сервис `plane-mcp` в docker-compose.yml.

После запуска Plane:
1. Получить API ключ: **Profile → Settings → API Tokens**
2. Указать в Environment:
   ```
   PLANE_API_KEY=ваш-api-ключ
   PLANE_WORKSPACE_SLUG=ваш-workspace
   ```
3. MCP endpoint: `http://ваш-домен:8001/sse`

## Масштабирование

Настройка реплик в Environment:
```
API_REPLICAS=2
WORKER_REPLICAS=3
WEB_REPLICAS=2
```

## Volumes

- `pgdata` — данные PostgreSQL
- `redisdata` — данные Redis
- `rabbitmq_data` — данные RabbitMQ
- `proxy_config`, `proxy_data` — конфиг proxy
- `logs_*` — логи приложений

## Лицензия

MIT
