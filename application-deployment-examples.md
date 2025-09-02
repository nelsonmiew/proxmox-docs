# Production Application Deployment Examples on Proxmox

## Overview

This guide provides real-world examples of deploying production applications on Proxmox VMs, covering both traditional LAMP/LEMP stacks and modern containerized deployments. Each example includes complete configurations, performance tuning, and best practices.

## VM ID Numbering Convention

All examples follow the standardized VM ID numbering scheme for consistent infrastructure management:

| VM ID Range | Purpose | Example VMs in This Guide |
|-------------|---------|---------------------------|
| **120-139** | Network Infrastructure | VM 121: HAProxy Load Balancer |
| **200-219** | Web Servers | VM 201: CakePHP E-commerce Web Server |
| **220-239** | Application Servers | VM 221: Django SaaS Application Server |
| **260-279** | Cache Servers | VM 261: Redis Cache (E-commerce)<br>VM 262: Redis Cache (Django) |
| **300-319** | Primary Databases | VM 301: MySQL Primary (E-commerce) |
| **320-339** | Database Replicas | VM 321: MySQL Replica (E-commerce) |
| **340-359** | Analytics Databases | VM 341: PostgreSQL (Django SaaS) |
| **500-519** | Docker Hosts | VM 501: Docker Host #1<br>VM 502: Docker Host #2 |

### VM Naming Convention

Each VM follows a descriptive naming pattern that matches its ID range:

```bash
# Format: [tier]-[technology]-[environment][number]
VM 201: web-cakephp-prod01          # Web tier, CakePHP, Production #1
VM 301: mysql-ecommerce-prod01      # Database tier, MySQL, E-commerce, Production #1
VM 221: app-django-saas-prod01      # Application tier, Django, SaaS, Production #1
VM 501: docker-host-prod01          # Container tier, Docker Host, Production #1
```

This systematic approach ensures:
- **Clear identification** of VM purpose from ID alone
- **Logical grouping** of related services
- **Easy scaling** with room for additional VMs in each tier
- **Consistent backup scheduling** based on ID ranges
- **Simplified monitoring** and automation

## Architecture Patterns

```
┌────────────────────────────────────────────────────────────┐
│                  Application Architecture                   │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Load Balancer (HAProxy/Nginx)                            │
│         │                                                   │
│         ├──── Web Tier (Multiple VMs)                     │
│         │      ├── Ubuntu + Apache/Nginx                  │
│         │      ├── PHP/Python Applications                │
│         │      └── Static Assets                          │
│         │                                                   │
│         ├──── Application Tier                            │
│         │      ├── CakePHP/Laravel                        │
│         │      ├── Django/Flask                           │
│         │      └── Node.js/Express                        │
│         │                                                   │
│         ├──── Database Tier                               │
│         │      ├── MySQL/MariaDB Cluster                  │
│         │      ├── PostgreSQL                             │
│         │      └── Redis/Memcached                        │
│         │                                                   │
│         └──── Container Platform                          │
│                ├── Docker Swarm                           │
│                └── Kubernetes                             │
└────────────────────────────────────────────────────────────┘
```

## Example 1: CakePHP E-Commerce Platform

### VM Configuration

```bash
# VM ID Guidelines for Applications:
# 200-219: Web servers (Apache/Nginx)
# 220-239: Application servers (PHP/Python/Node.js)
# 240-259: API gateways and microservices
# 300-319: Primary databases
# 320-339: Database replicas

# Create VM for CakePHP application (Web Server tier)
qm create 201 \
  --name "web-cakephp-prod01" \
  --description "CakePHP E-commerce Production Web Server #1" \
  --memory 8192 \
  --cores 4 \
  --sockets 1 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=200 \
  --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:60,cache=writeback,discard=on \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1
```

### Ubuntu Server Setup

```bash
#!/bin/bash
# /root/setup-cakephp.sh

# Update system
apt update && apt upgrade -y

# Install Apache, PHP 8.1 and required extensions
apt install -y apache2 \
  php8.1 php8.1-cli php8.1-common php8.1-curl \
  php8.1-mbstring php8.1-mysql php8.1-xml php8.1-zip \
  php8.1-gd php8.1-intl php8.1-bcmath php8.1-soap \
  php8.1-redis php8.1-opcache php8.1-apcu \
  libapache2-mod-php8.1 composer git unzip

# Enable Apache modules
a2enmod rewrite ssl headers expires deflate
systemctl restart apache2

# Install MySQL client
apt install -y mysql-client

# Configure PHP for production
cat > /etc/php/8.1/apache2/conf.d/99-custom.ini <<EOF
; Performance
max_execution_time = 300
memory_limit = 256M
post_max_size = 100M
upload_max_filesize = 100M
max_file_uploads = 20

; OPCache
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=2
opcache.fast_shutdown=1

; Session
session.cookie_httponly = 1
session.use_only_cookies = 1
session.cookie_secure = 1

; Security
expose_php = Off
display_errors = Off
log_errors = On
error_log = /var/log/php/error.log
EOF

# Create application directory
mkdir -p /var/www/ecommerce
cd /var/www/ecommerce

# Install CakePHP via Composer
composer create-project --prefer-dist cakephp/app:~4.4 .

# Set permissions
chown -R www-data:www-data /var/www/ecommerce
chmod -R 755 /var/www/ecommerce
chmod -R 777 /var/www/ecommerce/tmp
chmod -R 777 /var/www/ecommerce/logs

# Configure Apache Virtual Host
cat > /etc/apache2/sites-available/ecommerce.conf <<'EOF'
<VirtualHost *:80>
    ServerName ecommerce.example.com
    ServerAdmin admin@example.com
    DocumentRoot /var/www/ecommerce/webroot

    <Directory /var/www/ecommerce/webroot>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Security Headers
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    # Compression
    <IfModule mod_deflate.c>
        AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css text/javascript application/javascript
    </IfModule>

    # Caching
    <FilesMatch "\.(jpg|jpeg|png|gif|ico|css|js)$">
        Header set Cache-Control "max-age=2592000, public"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/ecommerce-error.log
    CustomLog ${APACHE_LOG_DIR}/ecommerce-access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName ecommerce.example.com
    DocumentRoot /var/www/ecommerce/webroot
    
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/ecommerce.crt
    SSLCertificateKeyFile /etc/ssl/private/ecommerce.key
    
    # Same Directory and Header configurations as above
</VirtualHost>
EOF

# Enable site
a2dissite 000-default.conf
a2ensite ecommerce.conf
systemctl reload apache2
```

### Database Configuration (Separate VM)

```bash
# Create MySQL VM (Primary Database tier - ID range 300-319)
qm create 301 \
  --name "mysql-ecommerce-prod01" \
  --description "MySQL Primary Database for E-commerce Platform" \
  --memory 16384 \
  --cores 8 \
  --sockets 1 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=210 \
  --scsi0 local-lvm:32,cache=none \
  --scsi1 local-lvm:200,cache=writeback \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1

# Optional: Create MySQL replica (Database replica tier - ID range 320-339)
qm create 321 \
  --name "mysql-ecommerce-replica01" \
  --description "MySQL Read Replica for E-commerce Platform" \
  --memory 12288 \
  --cores 6 \
  --sockets 1 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=210 \
  --scsi0 local-lvm:32,cache=none \
  --scsi1 local-lvm:200,cache=writeback \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1

# MySQL Installation and Optimization
apt install -y mysql-server mysql-client

# Production MySQL configuration
cat > /etc/mysql/mysql.conf.d/production.cnf <<EOF
[mysqld]
# Basic Settings
bind-address = 0.0.0.0
max_connections = 500
max_allowed_packet = 64M

# InnoDB Settings
innodb_buffer_pool_size = 12G  # 75% of RAM
innodb_log_file_size = 2G
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 2
innodb_thread_concurrency = 0
innodb_read_io_threads = 64
innodb_write_io_threads = 64

# Query Cache (MySQL 5.7)
query_cache_type = 1
query_cache_size = 256M
query_cache_limit = 2M

# Logging
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# Binary Logging for Replication
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW
expire_logs_days = 7
EOF

# Create database and user
mysql -e "CREATE DATABASE ecommerce_prod CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -e "CREATE USER 'ecommerce'@'10.0.200.%' IDENTIFIED BY 'SecurePassword123!';"
mysql -e "GRANT ALL PRIVILEGES ON ecommerce_prod.* TO 'ecommerce'@'10.0.200.%';"
mysql -e "FLUSH PRIVILEGES;"

# Additional cache/session VM (Cache tier - ID range 260-279)  
# Create Redis VM for sessions and caching
qm create 261 \
  --name "redis-cache-prod01" \
  --description "Redis Cache Server for Sessions and Application Cache" \
  --memory 4096 \
  --cores 2 \
  --sockets 1 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=210 \
  --scsi0 local-lvm:20,cache=writeback \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1
```

### CakePHP Application Configuration

```php
// config/app_local.php
<?php
return [
    'Datasources' => [
        'default' => [
            'host' => '10.0.210.51',  // MySQL Primary (VM 301)
            'username' => 'ecommerce',
            'password' => 'SecurePassword123!',
            'database' => 'ecommerce_prod',
            'driver' => 'Cake\Database\Driver\Mysql',
            'persistent' => false,
            'timezone' => 'UTC',
            'flags' => [],
            'cacheMetadata' => true,
            'log' => false,
            'quoteIdentifiers' => false,
        ],
        'read_replica' => [
            'host' => '10.0.210.52',  // MySQL Replica (VM 321)
            'username' => 'ecommerce',
            'password' => 'SecurePassword123!',
            'database' => 'ecommerce_prod',
            'driver' => 'Cake\Database\Driver\Mysql',
        ],
    ],
    
    'Cache' => [
        'default' => [
            'className' => 'Redis',
            'servers' => ['10.0.210.61'],  // Redis Cache (VM 261)
            'port' => 6379,
            'password' => false,
            'database' => 0,
            'duration' => '+2 hours',
        ],
    ],
    
    'Session' => [
        'defaults' => 'cache',
        'handler' => [
            'config' => 'default'
        ],
    ],
];
```

## Example 2: Django Multi-Tenant SaaS Application

### VM Setup for Django

```bash
# VM ID Guidelines for Django SaaS Platform:
# 220-239: Application servers (Django/Python)
# 340-359: Analytics databases (PostgreSQL)
# 260-279: Cache servers (Redis)

# Create Django application VM (Application Server tier)
qm create 221 \
  --name "app-django-saas-prod01" \
  --description "Django Multi-Tenant SaaS Application Server #1" \
  --memory 8192 \
  --cores 4 \
  --net0 virtio,bridge=vmbr0,tag=200 \
  --scsi0 local-lvm:40 \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1

# Create PostgreSQL database VM (Analytics Database tier - ID range 340-359)
qm create 341 \
  --name "postgres-saas-prod01" \
  --description "PostgreSQL Database for Django SaaS Platform" \
  --memory 16384 \
  --cores 8 \
  --sockets 1 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=210 \
  --scsi0 local-lvm:40,cache=none \
  --scsi1 local-lvm:300,cache=writeback \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1

# Create Redis VM for Django caching/sessions (Cache tier - reusing range 260-279)
qm create 262 \
  --name "redis-django-prod01" \
  --description "Redis Cache for Django SaaS Sessions and Celery" \
  --memory 4096 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0,tag=210 \
  --scsi0 local-lvm:20 \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1
```

### Django Production Setup

```bash
#!/bin/bash
# /root/setup-django.sh

# Install Python and dependencies
apt update
apt install -y python3.11 python3.11-venv python3.11-dev \
  python3-pip nginx supervisor postgresql-client \
  redis-server build-essential libpq-dev

# Create application user
useradd -m -d /home/django -s /bin/bash django

# Setup application directory
mkdir -p /var/www/saas-app
cd /var/www/saas-app

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Install Django and dependencies
pip install --upgrade pip
pip install django==4.2 \
  gunicorn==21.2 \
  psycopg2-binary==2.9 \
  redis==5.0 \
  celery==5.3 \
  django-tenant-schemas==1.12 \
  djangorestframework==3.14 \
  django-cors-headers==4.3 \
  whitenoise==6.6 \
  sentry-sdk==1.39

# Create Django project
django-admin startproject saas_platform .

# Production settings
cat > saas_platform/settings_production.py <<'EOF'
from .settings import *
import os
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

DEBUG = False
ALLOWED_HOSTS = ['saas.example.com', '10.0.200.101']

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django_tenants.postgresql_backend',
        'NAME': 'saas_prod',
        'USER': 'saas_user',
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': '10.0.210.91',  # PostgreSQL VM 341
        'PORT': '5432',
        'CONN_MAX_AGE': 600,
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}

# Cache
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://10.0.210.62:6379/1',  # Redis Django VM 262
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {'max_connections': 50}
        }
    }
}

# Static files
STATIC_ROOT = '/var/www/saas-app/static/'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Security
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'ERROR',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/error.log',
            'maxBytes': 1024 * 1024 * 15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['file'],
        'level': 'INFO',
    },
}

# Sentry
sentry_sdk.init(
    dsn="https://your-sentry-dsn@sentry.io/project-id",
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,
    send_default_pii=False
)
EOF

# Gunicorn configuration
cat > gunicorn_config.py <<EOF
import multiprocessing

bind = "127.0.0.1:8000"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "sync"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 50
preload_app = True
accesslog = "/var/log/gunicorn/access.log"
errorlog = "/var/log/gunicorn/error.log"
loglevel = "info"
EOF

# Nginx configuration
cat > /etc/nginx/sites-available/saas-app <<'EOF'
upstream django {
    server 127.0.0.1:8000;
    keepalive 32;
}

server {
    listen 80;
    server_name saas.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name saas.example.com;

    ssl_certificate /etc/ssl/certs/saas.crt;
    ssl_certificate_key /etc/ssl/private/saas.key;
    
    client_max_body_size 100M;
    
    location / {
        proxy_pass http://django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_buffering off;
        
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
    }
    
    location /static/ {
        alias /var/www/saas-app/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    location /media/ {
        alias /var/www/saas-app/media/;
        expires 7d;
    }
}
EOF

# Supervisor configuration
cat > /etc/supervisor/conf.d/django.conf <<EOF
[program:django]
command=/var/www/saas-app/venv/bin/gunicorn saas_platform.wsgi:application -c /var/www/saas-app/gunicorn_config.py
directory=/var/www/saas-app
user=django
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/supervisor/django.log
environment=PATH="/var/www/saas-app/venv/bin",DJANGO_SETTINGS_MODULE="saas_platform.settings_production"

[program:celery]
command=/var/www/saas-app/venv/bin/celery -A saas_platform worker -l info
directory=/var/www/saas-app
user=django
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/supervisor/celery.log

[program:celery-beat]
command=/var/www/saas-app/venv/bin/celery -A saas_platform beat -l info
directory=/var/www/saas-app
user=django
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/supervisor/celery-beat.log
EOF

# Set permissions
chown -R django:django /var/www/saas-app
chmod 755 /var/www/saas-app

# Start services
ln -s /etc/nginx/sites-available/saas-app /etc/nginx/sites-enabled/
nginx -t && systemctl restart nginx
supervisorctl reread
supervisorctl update
```

## Example 3: Docker Containerized Microservices

> **⚠️ IMPORTANT: Docker Deployment Strategy**
>
> **Always run Docker inside VMs for production deployments.** While the Proxmox community sometimes runs Docker in LXC containers for homelab use, this approach is officially unsupported and creates significant security and stability risks:
>
> - **Security**: LXC containers share the host kernel, making container escapes immediately compromise the entire Proxmox host
> - **Stability**: Proxmox updates frequently break Docker installations in LXC containers
> - **Support**: Docker-in-LXC is explicitly unsupported by Proxmox
> - **Compliance**: VM isolation is required for enterprise security standards (NIST 800-190, PCI-DSS)
>
> **The 5-10% resource overhead of VMs** is insignificant compared to the security, stability, and operational benefits provided by proper kernel isolation.

### Docker Host VM Setup

```bash
# VM ID Guidelines for Container Infrastructure:
# 500-519: Docker hosts
# 520-539: Kubernetes nodes
# 540-559: Container registries

# Create Docker host VM with more resources (Container Host tier - ID range 500-519)
# scsi1 provides additional storage for Docker volumes
qm create 501 \
  --name "docker-host-prod01" \
  --description "Docker Host for Production Microservices #1" \
  --memory 32768 \
  --cores 16 \
  --net0 virtio,bridge=vmbr0,tag=200 \
  --scsi0 local-lvm:100 \
  --scsi1 local-lvm:500 \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1

# Create additional Docker host for HA (optional)
qm create 502 \
  --name "docker-host-prod02" \
  --description "Docker Host for Production Microservices #2" \
  --memory 32768 \
  --cores 16 \
  --net0 virtio,bridge=vmbr0,tag=200 \
  --scsi0 local-lvm:100 \
  --scsi1 local-lvm:500 \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1
```

### Docker and Docker Compose Installation

```bash
#!/bin/bash
# /root/setup-docker.sh

# Install Docker
apt update
apt install -y ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Configure Docker daemon for production
cat > /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "10"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true,
  "features": {
    "buildkit": true
  },
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
EOF

systemctl restart docker

# Configure separate disk for Docker volumes
mkfs.ext4 /dev/sdb
mkdir -p /var/lib/docker/volumes
mount /dev/sdb /var/lib/docker/volumes
echo "/dev/sdb /var/lib/docker/volumes ext4 defaults 0 2" >> /etc/fstab
```

### Multi-Container Application Stack

```yaml
# /opt/apps/docker-compose.yml
version: '3.9'

services:
  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static_volume:/static:ro
    depends_on:
      - web
    networks:
      - frontend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G

  # Python/Django Application
  web:
    build:
      context: ./django-app
      dockerfile: Dockerfile.prod
    container_name: django-app
    command: gunicorn config.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    env_file:
      - .env.production
    depends_on:
      - db
      - redis
    networks:
      - frontend
      - backend
    restart: unless-stopped
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 4G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3

  # PostgreSQL Database
  db:
    image: postgres:15-alpine
    container_name: postgres-db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    environment:
      - POSTGRES_DB=app_db
      - POSTGRES_USER=app_user
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8
    networks:
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    command: redis-server --appendonly yes --maxmemory 2gb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G

  # Celery Worker
  celery:
    build:
      context: ./django-app
      dockerfile: Dockerfile.prod
    container_name: celery-worker
    command: celery -A config worker -l info -c 4
    volumes:
      - media_volume:/app/media
    env_file:
      - .env.production
    depends_on:
      - db
      - redis
    networks:
      - backend
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '2'
          memory: 2G

  # Celery Beat Scheduler
  celery-beat:
    build:
      context: ./django-app
      dockerfile: Dockerfile.prod
    container_name: celery-beat
    command: celery -A config beat -l info
    env_file:
      - .env.production
    depends_on:
      - db
      - redis
    networks:
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G

  # Monitoring - Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  # Monitoring - Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_INSTALL_PLUGINS=redis-datasource
    ports:
      - "3000:3000"
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
  monitoring:
    driver: bridge

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/postgres
  redis_data:
    driver: local
  static_volume:
    driver: local
  media_volume:
    driver: local
  prometheus_data:
    driver: local
  grafana_data:
    driver: local
```

### Production Dockerfile

```dockerfile
# django-app/Dockerfile.prod
FROM python:3.11-slim as builder

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# Final stage
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    libpq-dev \
    netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache /wheels/*

COPY . .

RUN python manage.py collectstatic --noinput

RUN useradd -m -u 1000 django && chown -R django:django /app
USER django

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "--threads", "2", "--timeout", "60", "config.wsgi:application"]
```

## Example 4: Docker Swarm Cluster

### Initialize Swarm Mode

```bash
# On first Docker host (manager)
docker swarm init --advertise-addr 10.0.200.101

# Deploy stack
docker stack deploy -c docker-compose.yml production-app

# Scale services
docker service scale production-app_web=5
docker service scale production-app_celery=3

# Rolling updates
docker service update \
  --update-parallelism 2 \
  --update-delay 30s \
  --image myapp:v2.0 \
  production-app_web
```

### Docker Swarm Monitoring

```yaml
# monitoring-stack.yml
version: '3.8'

services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    deploy:
      mode: global
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    deploy:
      mode: global
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring

networks:
  monitoring:
    driver: overlay
```

## Load Balancing Configuration

### HAProxy for Application Load Balancing

```bash
# Create Load Balancer VM (Infrastructure tier - ID range 120-139)
qm create 121 \
  --name "lb-haproxy-prod01" \
  --description "HAProxy Load Balancer for Production Applications" \
  --memory 4096 \
  --cores 4 \
  --net0 virtio,bridge=vmbr0,tag=200 \
  --scsi0 local-lvm:20 \
  --ide2 local:iso/ubuntu-22.04.3-live-server-amd64.iso,media=cdrom \
  --ostype l26 \
  --boot order=scsi0 \
  --agent enabled=1 \
  --onboot 1

# Install HAProxy on the load balancer VM
apt install -y haproxy

cat > /etc/haproxy/haproxy.cfg <<EOF
global
    maxconn 4096
    log 127.0.0.1 local0
    user haproxy
    group haproxy
    daemon

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog

frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/combined.pem
    redirect scheme https if !{ ssl_fc }
    
    # Rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny if { sc_http_req_rate(0) gt 20 }
    
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health/
    
    # CakePHP servers (VM 201, 202 if multiple)
    server cakephp-01 10.0.200.51:80 check maxconn 100  # VM 201
    server cakephp-02 10.0.200.52:80 check maxconn 100  # VM 202 (if created)
    
    # Django servers (VM 221, 222 if multiple)  
    server django-01 10.0.200.71:443 check ssl verify none maxconn 100  # VM 221
    server django-02 10.0.200.72:443 check ssl verify none maxconn 100  # VM 222 (if created)
    
    # Docker Swarm ingress (VM 501, 502)
    server swarm-01 10.0.200.251:80 check maxconn 200  # VM 501
    server swarm-02 10.0.200.252:80 check maxconn 200  # VM 502

stats enable
stats uri /haproxy-stats
stats auth admin:SecurePassword123
EOF

systemctl restart haproxy
```

## Performance Optimization

### VM Performance Tuning

```bash
# /etc/sysctl.d/99-performance.conf
# Network optimizations
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 400000

# Memory optimizations
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# File system
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
```

### Application Monitoring Integration

```yaml
# Prometheus scrape config for applications
scrape_configs:
  - job_name: 'cakephp'
    static_configs:
      - targets: ['10.0.200.201:9090']
        labels:
          app: 'ecommerce'
          env: 'production'
          
  - job_name: 'django'
    static_configs:
      - targets: ['10.0.200.301:8000']
    metrics_path: '/metrics'
    
  - job_name: 'docker'
    static_configs:
      - targets: ['10.0.200.401:9323']
```

## Backup Strategy for Applications

```bash
#!/bin/bash
# /opt/scripts/backup-applications.sh

# Backup databases
mysqldump -h 10.0.210.202 -u backup_user -p$MYSQL_PASS \
  --single-transaction --routines --triggers \
  ecommerce_prod | gzip > /backup/mysql/ecommerce_$(date +%Y%m%d).sql.gz

pg_dump -h 10.0.210.204 -U backup_user -d saas_prod \
  | gzip > /backup/postgres/saas_$(date +%Y%m%d).sql.gz

# Backup application files
rsync -avz /var/www/ecommerce/ /backup/apps/ecommerce/
rsync -avz /var/www/saas-app/ /backup/apps/django/

# Backup Docker volumes
docker run --rm -v production_postgres_data:/data \
  -v /backup/docker:/backup \
  alpine tar czf /backup/postgres_data_$(date +%Y%m%d).tar.gz -C /data .

# Upload to S3
aws s3 sync /backup/ s3://company-proxmox-backups/applications/ --delete

# Cleanup old backups
find /backup -type f -mtime +7 -delete
```

## Security Best Practices

### Application Security Checklist

```yaml
Web Applications:
  - Use HTTPS everywhere with proper certificates
  - Implement rate limiting at multiple layers
  - Enable WAF rules in reverse proxy
  - Regular dependency updates
  - Implement CSP headers
  - Use secure session management
  - Enable audit logging

Database Security:
  - Restrict network access by IP
  - Use SSL for connections
  - Regular security updates
  - Implement query monitoring
  - Use read-only replicas
  - Encrypt data at rest

Container Security:
  - Scan images for vulnerabilities
  - Use minimal base images
  - Run as non-root user
  - Implement resource limits
  - Use secrets management
  - Regular image updates
  - Network segmentation
```

This comprehensive guide provides production-ready configurations for deploying various application stacks on Proxmox, from traditional LAMP/LEMP setups to modern containerized microservices.