# Инструкция по развертыванию

## Подготовка сервера

### 1. Установка системных зависимостей

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install python3 python3-pip python3-venv nginx postgresql postgresql-contrib redis-server

# CentOS/RHEL
sudo yum install python3 python3-pip nginx postgresql postgresql-server redis
```

### 2. Настройка PostgreSQL

```bash
sudo -u postgres psql
CREATE DATABASE sellmind;
CREATE USER sellmind_user WITH PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE sellmind TO sellmind_user;
\q
```

### 3. Настройка Redis

```bash
sudo systemctl enable redis
sudo systemctl start redis
```

## Развертывание приложения

### 1. Клонирование и настройка

```bash
cd /var/www/
git clone <repository-url> sellmind
cd sellmind/sellmind
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2. Настройка окружения

```bash
cp .env.example .env
nano .env
```

Заполните переменные:
```env
SECRET_KEY=generate-secure-key-here
DEBUG=False
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
POSTGRES_DB=sellmind
POSTGRES_USER=sellmind_user
POSTGRES_PASSWORD=secure_password
POSTGRES_HOST=localhost
OPENAI_API_KEY=your_openai_key
```

### 3. Применение миграций

```bash
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

### 4. Настройка Gunicorn

```bash
pip install gunicorn

# Тест запуска
gunicorn conf.wsgi:application --bind 0.0.0.0:8000
```

Создайте systemd service `/etc/systemd/system/sellmind.service`:

```ini
[Unit]
Description=Sellmind Django app
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/sellmind/sellmind
Environment="PATH=/var/www/sellmind/sellmind/venv/bin"
ExecStart=/var/www/sellmind/sellmind/venv/bin/gunicorn --workers 3 --bind unix:/var/www/sellmind/sellmind.sock conf.wsgi:application

[Install]
WantedBy=multi-user.target
```

### 5. Настройка Nginx

Создайте `/etc/nginx/sites-available/sellmind`:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    
    location /static/ {
        root /var/www/sellmind/sellmind;
    }

    location /media/ {
        root /var/www/sellmind/sellmind;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/sellmind/sellmind.sock;
    }
}
```

Активируйте конфигурацию:

```bash
sudo ln -s /etc/nginx/sites-available/sellmind /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
```

### 6. Запуск Celery

Создайте systemd service `/etc/systemd/system/sellmind-celery.service`:

```ini
[Unit]
Description=Sellmind Celery Worker
After=network.target

[Service]
Type=forking
User=www-data
Group=www-data
EnvironmentFile=/var/www/sellmind/sellmind/.env
WorkingDirectory=/var/www/sellmind/sellmind
ExecStart=/var/www/sellmind/sellmind/venv/bin/celery multi start worker1 \
    -A conf --pidfile=/var/run/celery/%n.pid \
    --logfile=/var/log/celery/%n%I.log --loglevel=INFO
ExecStop=/var/www/sellmind/sellmind/venv/bin/celery multi stopwait worker1 \
    --pidfile=/var/run/celery/%n.pid
ExecReload=/var/www/sellmind/sellmind/venv/bin/celery multi restart worker1 \
    -A conf --pidfile=/var/run/celery/%n.pid \
    --logfile=/var/log/celery/%n%I.log --loglevel=INFO

[Install]
WantedBy=multi-user.target
```

Создайте директории для логов:

```bash
sudo mkdir -p /var/run/celery /var/log/celery
sudo chown www-data:www-data /var/run/celery /var/log/celery
```

### 7. Запуск сервисов

```bash
sudo systemctl daemon-reload
sudo systemctl enable sellmind
sudo systemctl enable sellmind-celery
sudo systemctl start sellmind
sudo systemctl start sellmind-celery
```

## SSL сертификат (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

## Мониторинг

### Проверка статуса

```bash
sudo systemctl status sellmind
sudo systemctl status sellmind-celery
sudo systemctl status nginx
sudo systemctl status postgresql
sudo systemctl status redis
```

### Логи

```bash
# Django логи
tail -f /var/log/celery/worker1.log

# Nginx логи
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Systemd логи
journalctl -u sellmind -f
journalctl -u sellmind-celery -f
```

## Backup

### База данных

```bash
# Создание бэкапа
pg_dump -U sellmind_user -h localhost sellmind > backup_$(date +%Y%m%d_%H%M%S).sql

# Восстановление
psql -U sellmind_user -h localhost sellmind < backup_file.sql
```

### Файлы

```bash
# Создание архива
tar -czf sellmind_backup_$(date +%Y%m%d_%H%M%S).tar.gz /var/www/sellmind/

# Исключая виртуальное окружение
tar --exclude='venv' -czf sellmind_backup_$(date +%Y%m%d_%H%M%S).tar.gz /var/www/sellmind/
```

## Обновление

```bash
cd /var/www/sellmind/sellmind
git pull origin main
source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
sudo systemctl restart sellmind
sudo systemctl restart sellmind-celery
``` 