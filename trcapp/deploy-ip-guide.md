# Руководство по деплою в Digital Ocean (без домена, по IP)

## Подготовка к деплою

### 1. Создайте droplet в Digital Ocean
- Выберите Ubuntu 22.04 LTS
- Минимум: 2GB RAM, 1 vCPU, 50GB SSD
- Рекомендуется: 4GB RAM, 2 vCPU, 80GB SSD
- **Запомните IP адрес вашего сервера!**

### 2. Настройте сервер

```bash
# Подключитесь к серверу
ssh root@YOUR_SERVER_IP

# Обновите систему
apt update && apt upgrade -y

# Установите Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Установите Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Установите Git
apt install git -y
```

### 3. Скопируйте проект на сервер

```bash
# Клонируйте репозиторий
git clone <your-repo-url>
cd trcapp

# Или загрузите файлы через scp
scp -r /path/to/your/project root@YOUR_SERVER_IP:/root/trcapp
```

### 4. Настройте переменные окружения

Создайте файл `.env` (замените YOUR_SERVER_IP на реальный IP):

```bash
nano .env
```

Содержимое файла:

```env
# Database
DATABASE_URL=postgresql://postgres:your_strong_password@db:5432/trcapp
POSTGRES_PASSWORD=your_strong_password_here
POSTGRES_USER=postgres
POSTGRES_DB=trcapp

# Redis
REDIS_URL=redis://redis:6379/0

# Message Queue
CELERY_BROKER_URL=amqp://rabbitmq:5672//
RABBITMQ_USER=rabbitmq
RABBITMQ_PASS=your_rabbitmq_password

# Security (КРИТИЧЕСКИ ВАЖНО!)
SECRET_KEY=your_very_long_secret_key_here_at_least_32_characters_very_secure_random
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=14

# Admin
ADMIN_EMAILS=admin@example.com
ADMIN_DEFAULT_PASSWORD=your_strong_admin_password

# External APIs
OPENAI_API_KEY=your_openai_api_key_if_needed

# Google OAuth (если используете)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_REDIRECT_URI=http://YOUR_SERVER_IP/api/auth/google/callback

# Frontend
VITE_API_BASE_URL=http://YOUR_SERVER_IP
```

**⚠️ ВАЖНО:** Замените `YOUR_SERVER_IP` на реальный IP адрес вашего сервера!

### 5. Запустите приложение

```bash
# Запустите продакшен версию
docker-compose -f docker-compose.prod.yml up -d

# Или обычную версию для тестирования
docker-compose up -d
```

### 6. Проверьте работу

```bash
# Проверьте статус контейнеров
docker-compose ps

# Проверьте логи
docker-compose logs -f

# Проверьте health endpoint
curl http://YOUR_SERVER_IP/api/health

# Проверьте frontend
curl http://YOUR_SERVER_IP
```

### 7. Настройте firewall (рекомендуется)

```bash
# Включите UFW
ufw enable

# Разрешите SSH
ufw allow ssh

# Разрешите HTTP
ufw allow 80

# Проверьте статус
ufw status
```

## Доступ к приложению

После успешного запуска:
- **Frontend:** http://YOUR_SERVER_IP
- **Backend API:** http://YOUR_SERVER_IP/api/
- **Admin панель:** http://YOUR_SERVER_IP (войдите с email из ADMIN_EMAILS)

## Полезные команды

### Перезапуск приложения
```bash
docker-compose down
docker-compose -f docker-compose.prod.yml up -d
```

### Просмотр логов
```bash
# Все сервисы
docker-compose logs -f

# Конкретный сервис
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f nginx
```

### Backup базы данных
```bash
# Создание backup
docker-compose exec db pg_dump -U postgres trcapp > backup_$(date +%Y%m%d_%H%M%S).sql

# Восстановление
docker-compose exec -T db psql -U postgres trcapp < backup_file.sql
```

### Обновление приложения
```bash
# Остановите приложение
docker-compose down

# Получите обновления
git pull

# Пересоберите и запустите
docker-compose -f docker-compose.prod.yml up -d --build
```

## Мониторинг

### Проверка ресурсов
```bash
# Использование ресурсов контейнерами
docker stats

# Дисковое пространство
df -h

# Память
free -h
```

### Очистка системы
```bash
# Удаление неиспользуемых образов
docker image prune -a

# Удаление неиспользуемых томов
docker volume prune

# Полная очистка
docker system prune -a
```

## Безопасность

1. **Смените пароли по умолчанию**
2. **Используйте сильные пароли (минимум 16 символов)**
3. **Регулярно обновляйте систему:**
   ```bash
   apt update && apt upgrade -y
   ```
4. **Настройте SSH ключи вместо паролей**
5. **Регулярно делайте backup базы данных**

## Возможные проблемы

### Контейнер не запускается
```bash
# Проверьте логи
docker-compose logs [service_name]

# Пересоберите образ
docker-compose build [service_name]
```

### Приложение недоступно
```bash
# Проверьте порты
netstat -tlnp | grep :80

# Проверьте firewall
ufw status

# Проверьте nginx
docker-compose logs nginx
```

### База данных не подключается
```bash
# Проверьте статус PostgreSQL
docker-compose logs db

# Проверьте переменные окружения
docker-compose config
```

## Масштабирование

Для увеличения производительности:
```bash
# Увеличьте workers в docker-compose.prod.yml
# backend service command:
command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4

# Добавьте больше Celery workers
docker-compose up -d --scale celery_worker=3
``` 