# Руководство по деплою в Digital Ocean

## Подготовка к деплою

### 1. Создайте droplet в Digital Ocean
- Выберите Ubuntu 22.04 LTS
- Минимум: 2GB RAM, 1 vCPU, 50GB SSD
- Рекомендуется: 4GB RAM, 2 vCPU, 80GB SSD

### 2. Настройте сервер

```bash
# Обновите систему
sudo apt update && sudo apt upgrade -y

# Установите Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Установите Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Перезайдите в систему для применения прав docker
```

### 3. Скопируйте проект на сервер

```bash
# Клонируйте репозиторий или загрузите файлы
git clone <your-repo-url>
cd trcapp
```

### 4. Настройте переменные окружения

Создайте файл `.env`:

```bash
# Database
DATABASE_URL=postgresql://postgres:your_strong_password@db:5432/trcapp
POSTGRES_PASSWORD=your_strong_password
POSTGRES_USER=postgres
POSTGRES_DB=trcapp

# Redis
REDIS_URL=redis://redis:6379/0

# Message Queue
CELERY_BROKER_URL=amqp://rabbitmq:5672//
RABBITMQ_USER=rabbitmq
RABBITMQ_PASS=your_rabbitmq_password

# Security (КРИТИЧЕСКИ ВАЖНО!)
SECRET_KEY=your_very_long_secret_key_here_at_least_32_characters_very_secure
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=14

# Admin
ADMIN_EMAILS=admin@yourdomain.com
ADMIN_DEFAULT_PASSWORD=your_strong_admin_password

# External APIs
OPENAI_API_KEY=your_openai_api_key

# Google OAuth
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_REDIRECT_URI=https://yourdomain.com/api/auth/google/callback

# Frontend
VITE_API_BASE_URL=https://yourdomain.com
```

### 5. Настройте домен и SSL

#### A. Настройте DNS
Добавьте A-запись в вашем DNS провайдере:
```
A    @    your_server_ip
A    www  your_server_ip
```

#### B. Получите SSL сертификат (рекомендуется Certbot)

```bash
# Установите Certbot
sudo apt install certbot python3-certbot-nginx

# Временно остановите nginx если запущен
sudo docker-compose down nginx

# Получите сертификат
sudo certbot certonly --standalone -d yourdomain.com -d www.yourdomain.com

# Создайте директорию для SSL
mkdir -p nginx/ssl
sudo cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem nginx/ssl/
sudo cp /etc/letsencrypt/live/yourdomain.com/privkey.pem nginx/ssl/
```

#### C. Обновите nginx.conf для HTTPS

### 6. Запустите приложение

```bash
# Для продакшена используйте:
docker-compose -f docker-compose.prod.yml up -d

# Или для разработки:
docker-compose up -d
```

### 7. Проверьте статус

```bash
# Проверьте статус контейнеров
docker-compose ps

# Проверьте логи
docker-compose logs backend
docker-compose logs frontend
docker-compose logs nginx

# Проверьте health endpoints
curl http://your_domain/api/health
```

## Мониторинг и обслуживание

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
docker-compose up -d --build
```

### Логи
```bash
# Все логи
docker-compose logs -f

# Конкретный сервис
docker-compose logs -f backend
```

## Безопасность

1. **Смените все пароли по умолчанию**
2. **Используйте сильные пароли (минимум 16 символов)**
3. **Настройте firewall**:
   ```bash
   sudo ufw enable
   sudo ufw allow ssh
   sudo ufw allow 80
   sudo ufw allow 443
   ```
4. **Регулярно обновляйте систему и контейнеры**
5. **Настройте регулярные backup'ы**

## Масштабирование

Для увеличения нагрузки:
- Увеличьте количество workers в backend
- Добавьте больше Celery workers
- Используйте внешние сервисы (AWS RDS, Redis Cloud)
- Настройте load balancer 