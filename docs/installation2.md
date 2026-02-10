SkyNet Bangladesh আইএসপি হিসেবে বেশ নাম করেছে। ঢাকা নর্থ জোনে মূলতঃ তাদের নেটওয়ার্ক। বর্তমানে মিরপুর এবং উত্তরায় দুটো পপ, প্রায় ৮ হাজার কাস্টমার। তারা সিদ্ধান্ত নিয়েছে এক্সেল শিট/গুগল শিট এবং হাতে আঁকা ডায়াগ্রাম ছেড়ে একটা প্রপার Network Source of Truth সিস্টেম বানাবে। তাদের লক্ষ্য পরের দুই বছরে ৫০ হাজার কাস্টমারে পৌঁছানো, আর তারপর ১ লক্ষ। এজন্য দরকার একটা শক্ত ফাউন্ডেশন। সেই ফাউন্ডেশনের নাম Nautobot 3.0।

এই চ্যাপ্টারে আমরা দেখব কীভাবে SkyNets Bangladesh তাদের নেটওয়ার্কের জন্য Nautobot 3.0 ইনস্টল এবং সেটআপ করেছে। আপনিও একইভাবে আপনার ISP-এর জন্য সেটআপ করতে পারবেন।

## কেন Nautobot 3.0?

প্রথম প্রশ্ন - কেন Nautobot 3.0? কেন Nautobot 2.x না? কেন NetBox না?

**Nautobot 3.0-এর মূল সুবিধা:**

১. **Flexible Location Hierarchy:** আগে Sites একটা ফ্ল্যাট স্ট্রাকচার ছিল। এখন Locations দিয়ে যত খুশি লেভেলের hierarchy বানাতে পারবেন। Zone → Cluster → POP → Building → Floor - যেভাবে চান সাজান।

২. **Unified Role System:** ডিভাইস রোল, র‍্যাক রোল, IPAM রোল - সব আলাদা আলাদা ছিল। এখন একটা unified Roles system। এক জায়গা থেকে সব ম্যানেজ করুন।

৩. **Custom Relationships:** দুটো যেকোনো অবজেক্টের মধ্যে custom relationship তৈরি করতে পারবেন। যেমন "Device supports Customer" - এরকম relationship।

৪. **Better Performance:** ডেটাবেস optimization, faster queries, improved UI responsiveness।

৫. **GraphQL API:** REST API-এর পাশাপাশি GraphQL support। জটিল query সহজে করা যায়।

৬. **Plugin Ecosystem:** আরো শক্তিশালী plugin system। Golden Config, Device Lifecycle Management - এসব plugin সহজে integrate করা যায়।

SkyNets Bangladesh এই কারণে 3.0 বেছে নিয়েছে। তারা জানে ভবিষ্যতে scaling করতে হবে, তাই শুরু থেকেই সবচেয়ে আধুনিক ভার্সন দিয়ে শুরু করছে।

## সিস্টেম রিকোয়ারমেন্ট

Nautobot চালাতে হলে একটা লিনাক্স সার্ভার দরকার। SkyNet Bangladesh-এর ক্ষেত্রে তারা একটা ডেডিকেটেড Ubuntu server সেটআপ করেছে।

### হার্ডওয়্যার রিকোয়ারমেন্ট (SkyNet-এর সাইজের জন্য):

```
CPU: 4 cores (Intel/AMD)
RAM: 8 GB (minimum), 16 GB (recommended)
Storage: 100 GB SSD
Network: 1 Gbps ethernet
```

যদি আপনার নেটওয়ার্ক খুব ছোট হয় (১-২ হাজার কাস্টমার), তাহলে 2 cores এবং 4 GB RAM দিয়েও চলবে। কিন্তু future-proofing-এর জন্য একটু বেশি নেওয়া ভালো।

### সফটওয়্যার রিকোয়ারমেন্ট:

```
Operating System: Ubuntu 24.04 LTS (recommended)
                  অথবা Ubuntu 22.04 LTS
                  অথবা Debian 12

Python: 3.11 বা তার উপরে
PostgreSQL: 13 বা তার উপরে
Redis: 6.0 বা তার উপরে
```

SkyNet Ubuntu 24.04 LTS বেছে নিয়েছে কারণ এটা long-term support পাবে ২০২৯ সাল পর্যন্ত।

## ইনস্টলেশন পদ্ধতি - কোনটা বেছে নেবেন?

Nautobot ইনস্টল করার দুটো প্রধান উপায়:

**১. Docker দিয়ে (সহজ, দ্রুত)**

- ১০-১৫ মিনিটে চালু করা যায়
- Development/testing-এর জন্য পারফেক্ট
- Production-এও চলে কিন্তু কাস্টমাইজেশন কম

**২. Traditional Installation (সম্পূর্ণ নিয়ন্ত্রণ নিজ হাতে)**

- একটু সময় লাগে (৩০-৪৫ মিনিট)
- Production-এর জন্য best
- সব কিছু নিজের মতো সাজানো যায়

SkyNet Bangladesh দুটো পদ্ধতিই অনুসরণ করেছে:

- প্রথমে Docker দিয়ে টেস্ট করেছে
- তারপর Production-এর জন্য Traditional Installation করেছে 

আমরা দুটোই দেখব।

---

## পদ্ধতি ১: Docker দিয়ে দ্রুত শুরু

### স্টেপ ১: সার্ভার প্রস্তুত করা

একটা নতুন Ubuntu 24.04 সার্ভার নিন। SkyNet একটা VM ব্যবহার করেছে তাদের NOC-এর ভেতরে।

SSH করে সার্ভারে লগইন করুন:

```bash
ssh admin@nautobot-server.skynet.bd
```

সিস্টেম আপডেট করুন:

```bash
sudo apt update
sudo apt upgrade -y
```

### স্টেপ ২: Docker ইনস্টল করা

Docker এবং Docker Compose ইনস্টল করুন:

```bash
# Docker এর official GPG key যোগ করুন
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Repository যোগ করুন
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker ইনস্টল করুন
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

যাচাই করুন:

```bash
sudo docker --version
```

দেখাবে: `Docker version 24.0.7, build...`

### স্টেপ ৩: Nautobot Docker Compose সেটআপ

Nautobot-এর official docker compose repository clone করুন:

```bash
cd /opt
sudo git clone https://github.com/nautobot/nautobot-docker-compose.git
cd nautobot-docker-compose
```

Environment configuration তৈরি করুন:

```bash
sudo cp development/creds.example.env development/creds.env
sudo cp development/dev.env .env
```

এখন configuration ফাইল এডিট করুন:

```bash
sudo nano development/creds.env
```

গুরুত্বপূর্ণ সেটিংস:

```bash
# Database credentials
NAUTOBOT_DB_USER=nautobot
NAUTOBOT_DB_PASSWORD=SkyNet@2025$Secure  # শক্তিশালী পাসওয়ার্ড দিন
POSTGRES_PASSWORD=SkyNet@2025$PostgreSQL  # আলাদা শক্তিশালী পাসওয়ার্ড

# Nautobot superuser
NAUTOBOT_CREATE_SUPERUSER=true
NAUTOBOT_SUPERUSER_USERNAME=admin
NAUTOBOT_SUPERUSER_PASSWORD=SkyNet@Admin2025!  # শক্তিশালী পাসওয়ার্ড
NAUTOBOT_SUPERUSER_EMAIL=admin@skynet.bd
NAUTOBOT_SUPERUSER_API_TOKEN=0123456789abcdef0123456789abcdef01234567  # Random 40-char token

# Redis password
NAUTOBOT_REDIS_PASSWORD=SkyNet@Redis2025$
```

`.env` ফাইল এডিট করুন:

```bash
sudo nano .env
```

```bash
NAUTOBOT_IMAGE=networktocode/nautobot:3.0-py3.11
NAUTOBOT_VERSION=3.0.0

# SkyNet specific settings
NAUTOBOT_ALLOWED_HOSTS=nautobot.skynet.bd,localhost,127.0.0.1
```

সেভ করুন (Ctrl+O, Enter, Ctrl+X)।

### স্টেপ ৪: Nautobot চালু করা

Docker containers শুরু করুন:

```bash
sudo docker compose up -d
```

প্রথমবার চালালে সব images download হবে। ৫-১০ মিনিট লাগতে পারে (ইন্টারনেট স্পিডের উপর নির্ভর করে)।

আউটপুট দেখাবে:

```
[+] Running 5/5
 ✔ Container nautobot-redis-1     Started
 ✔ Container nautobot-db-1        Started
 ✔ Container nautobot-nautobot-1  Started
 ✔ Container nautobot-celery-worker-1  Started
 ✔ Container nautobot-celery-beat-1    Started
```

স্ট্যাটাস চেক করুন:

```bash
sudo docker compose ps
```

সব containers "healthy" বা "running" দেখাবে।

### স্টেপ ৫: প্রথম লগইন

ব্রাউজারে যান:

```
http://your-server-ip:8080
```

SkyNet-এর ক্ষেত্রে:

```
http://192.168.10.50:8080
```

লগইন পেজ দেখাবে। আপনার সুপারইউজার ক্রেডেনশিয়াল দিয়ে লগইন করুন:

```
Username: admin
Password: SkyNet@Admin2025!
```

অভিনন্দন! Nautobot 3.0 চালু হয়ে গেছে।

### Docker সেটআপের সীমাবদ্ধতা

Docker দিয়ে শুরু করা সহজ, কিন্তু কিছু সীমাবদ্ধতা আছে:

- Custom plugins ইনস্টল করা জটিল
- Performance tuning সীমিত
- Backup/restore complex
- Production-grade SSL/Nginx setup লাগে extra কাজ

এজন্য SkyNet Bangladesh production-এর জন্য Traditional Installation বেছে নিয়েছে।

---

## পদ্ধতি ২: Traditional Installation (Production-Ready)

এখন দেখব কীভাবে SkyNet Bangladesh তাদের production Nautobot server সেটআপ করেছে। এটা একটু সময় সাপেক্ষ কিন্তু সম্পূর্ণ নিয়ন্ত্রণ পাবেন।

### স্টেপ ১: নতুন Ubuntu 24.04 Server

একটা ফ্রেশ Ubuntu 24.04 LTS সার্ভার নিন। SSH করে লগইন করুন:

```bash
ssh admin@nautobot.skynet.bd
```

সিস্টেম আপডেট:

```bash
sudo apt update && sudo apt upgrade -y
```

প্রয়োজনীয় প্যাকেজ ইনস্টল:

```bash
sudo apt install -y git python3.11 python3.11-venv python3-pip python3-dev \
  build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev \
  zlib1g-dev redis-server postgresql postgresql-contrib nginx
```

### স্টেপ ২: PostgreSQL ডেটাবেস সেটআপ

PostgreSQL ডেটাবেস এবং ইউজার তৈরি করুন:

```bash
# PostgreSQL-এ যান
sudo -u postgres psql
```

PostgreSQL প্রম্পটে:

```sql
-- Nautobot database তৈরি
CREATE DATABASE nautobot;

-- Nautobot user তৈরি
CREATE USER nautobot WITH PASSWORD 'SkyNet@DB2025$Secure';

-- User-কে database access দিন
ALTER ROLE nautobot SET client_encoding TO 'utf8';
ALTER ROLE nautobot SET default_transaction_isolation TO 'read committed';
ALTER ROLE nautobot SET timezone TO 'Asia/Dhaka';

-- সব privileges দিন
GRANT ALL PRIVILEGES ON DATABASE nautobot TO nautobot;

-- Database owner করুন
ALTER DATABASE nautobot OWNER TO nautobot;

-- বের হন
\q
```

যাচাই করুন:

```bash
psql -U nautobot -d nautobot -h localhost
# Password দিন: SkyNet@DB2025$Secure
```

যদি কানেক্ট হয়, তাহলে ঠিক আছে। `\q` দিয়ে বের হন।

### স্টেপ ৩: Redis সেটআপ

Redis configuration এডিট করুন:

```bash
sudo nano /etc/redis/redis.conf
```

এই লাইনগুলো খুঁজে পরিবর্তন করুন:

```conf
# Password সেট করুন (line খুঁজে আনকমেন্ট করুন)
requirepass SkyNet@Redis2025$

# Bind করুন localhost-এ
bind 127.0.0.1

# Persistence enable করুন
save 900 1
save 300 10
save 60 10000
```

সেভ করুন এবং Redis restart করুন:

```bash
sudo systemctl restart redis-server
sudo systemctl enable redis-server
```

যাচাই করুন:

```bash
redis-cli
> AUTH SkyNet@Redis2025$
> PING
```

দেখাবে: `PONG`

### স্টেপ ৪: Nautobot ইউজার তৈরি

Nautobot-এর জন্য একটা dedicated system user তৈরি করুন:

```bash
sudo useradd --system --shell /bin/bash --create-home --home-dir /opt/nautobot nautobot
```

এই ইউজারে switch করুন:

```bash
sudo -iu nautobot
```

### স্টেপ ৫: Python Virtual Environment

Virtual environment তৈরি করুন:

```bash
cd /opt/nautobot
python3.11 -m venv venv
source venv/bin/activate
```

Pip upgrade করুন:

```bash
pip install --upgrade pip wheel
```

### স্টেপ ৬: Nautobot 3.0 ইনস্টল করা

Nautobot ইনস্টল করুন:

```bash
pip install nautobot==3.0.0
```

এটা কয়েক মিনিট লাগবে। অনেক dependencies install হবে।

ইনস্টল verify করুন:

```bash
nautobot-server --version
```

দেখাবে: `3.0.0`

### স্টেপ ৭: Nautobot Configuration

Configuration ডিরেক্টরি তৈরি:

```bash
mkdir -p /opt/nautobot/nautobot_config
```

Configuration file তৈরি করুন:

```bash
nano /opt/nautobot/nautobot_config/nautobot_config.py
```

এই configuration টা পেস্ট করুন (SkyNet Bangladesh-এর জন্য কাস্টমাইজড):

```python
"""
Nautobot Configuration for SkyNet Bangladesh
Production Setup
"""

import os
from nautobot.core.settings import *  # noqa: F403

# ==============================
# Basic Settings
# ==============================

ALLOWED_HOSTS = [
    'nautobot.skynet.bd',
    'localhost',
    '127.0.0.1',
    '192.168.10.50',  # SkyNet internal IP
]

# Database Configuration
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'nautobot',
        'USER': 'nautobot',
        'PASSWORD': 'SkyNet@DB2025$Secure',
        'HOST': 'localhost',
        'PORT': '',
        'CONN_MAX_AGE': 300,
    }
}

# Redis Configuration
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PASSWORD': 'SkyNet@Redis2025$',
        },
    }
}

# Celery Configuration (Background Tasks)
CELERY_BROKER_URL = 'redis://:SkyNet@Redis2025$@127.0.0.1:6379/1'
CELERY_RESULT_BACKEND = 'redis://:SkyNet@Redis2025$@127.0.0.1:6379/1'

# ==============================
# Security Settings
# ==============================

# SECRET_KEY - Generate করুন এভাবে:
# python3 -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
SECRET_KEY = 'ab#cd$ef@12*34&56^78(90)gh!ij~kl+mn=op_qr-st[uv]wx{yz}'  # এটা বদলান!

# Session security
SESSION_COOKIE_SECURE = True  # HTTPS-এর জন্য
CSRF_COOKIE_SECURE = True

# ==============================
# Time Zone
# ==============================

TIME_ZONE = 'Asia/Dhaka'

# ==============================
# Static Files এবং Media
# ==============================

STATIC_ROOT = '/opt/nautobot/static'
MEDIA_ROOT = '/opt/nautobot/media'

# ==============================
# Logging
# ==============================

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
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/opt/nautobot/logs/nautobot.log',
            'maxBytes': 1024 * 1024 * 50,  # 50 MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'nautobot': {
            'handlers': ['file', 'console'],
            'level': 'INFO',
        },
    },
}

# ==============================
# SkyNet Bangladesh Custom Settings
# ==============================

# Organization name
BANNER_TOP = 'SkyNet Bangladesh - Network Source of Truth'
BANNER_BOTTOM = 'Managed by NOC Team'

# Support contact
SUPPORT_EMAIL = 'noc@skynet.bd'
SUPPORT_MESSAGE = 'For support, contact NOC team at noc@skynet.bd'
```

সেভ করুন।

**গুরুত্বপূর্ণ:** SECRET_KEY একটা নতুন জেনারেট করুন:

```bash
python3 -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

যে key আসবে সেটা configuration file-এ বসান।

### স্টেপ ৮: Static Files এবং Logs Directory

প্রয়োজনীয় ডিরেক্টরি তৈরি করুন:

```bash
mkdir -p /opt/nautobot/static
mkdir -p /opt/nautobot/media
mkdir -p /opt/nautobot/logs
```

### স্টেপ ৯: Database Migration

Database tables তৈরি করুন:

```bash
nautobot-server migrate
```

এটা ৫-১০ মিনিট লাগবে। অনেকগুলো migration apply হবে।

আউটপুট শেষে দেখাবে:

```
Applying sessions.0001_initial... OK
Applying users.0001_initial... OK
...
Operations to perform:
  Apply all migrations: ... (success)
```

### স্টেপ ১০: Superuser তৈরি

Admin user তৈরি করুন:

```bash
nautobot-server createsuperuser
```

প্রম্পট আসবে:

```
Username: admin
Email address: admin@skynet.bd
Password: ********  (SkyNet@Admin2025!)
Password (again): ********
Superuser created successfully.
```

### স্টেপ ১১: Static Files Collect

Static files collect করুন:

```bash
nautobot-server collectstatic --no-input
```

এটা CSS, JavaScript, images সব একসাথে `/opt/nautobot/static`-এ কপি করবে।

এখন `nautobot` user থেকে বের হন:

```bash
exit
```

### স্টেপ ১২: Systemd Service Files

Nautobot-কে systemd service হিসেবে চালাবেন যাতে server restart হলে automatically চালু হয়।

**Nautobot Service:**

```bash
sudo nano /etc/systemd/system/nautobot.service
```

```ini
[Unit]
Description=Nautobot WSGI Service - SkyNet Bangladesh
Documentation=https://docs.nautobot.com/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=nautobot
Group=nautobot
PIDFile=/var/run/nautobot/nautobot.pid
WorkingDirectory=/opt/nautobot

Environment="NAUTOBOT_ROOT=/opt/nautobot"
ExecStart=/opt/nautobot/venv/bin/gunicorn \
    --workers 4 \
    --bind 127.0.0.1:8001 \
    --timeout 120 \
    --access-logfile /opt/nautobot/logs/access.log \
    --error-logfile /opt/nautobot/logs/error.log \
    nautobot.core.wsgi:application

Restart=on-failure
RestartSec=30
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

**Nautobot Celery Worker (Background Tasks):**

```bash
sudo nano /etc/systemd/system/nautobot-celery.service
```

```ini
[Unit]
Description=Nautobot Celery Worker - SkyNet Bangladesh
Documentation=https://docs.nautobot.com/
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
User=nautobot
Group=nautobot
WorkingDirectory=/opt/nautobot

Environment="NAUTOBOT_ROOT=/opt/nautobot"
ExecStart=/opt/nautobot/venv/bin/celery -A nautobot.core.celery worker \
    --loglevel INFO \
    --logfile /opt/nautobot/logs/celery.log

Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

**Nautobot Celery Beat (Scheduled Tasks):**

```bash
sudo nano /etc/systemd/system/nautobot-celery-beat.service
```

```ini
[Unit]
Description=Nautobot Celery Beat - SkyNet Bangladesh
Documentation=https://docs.nautobot.com/
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
User=nautobot
Group=nautobot
WorkingDirectory=/opt/nautobot

Environment="NAUTOBOT_ROOT=/opt/nautobot"
ExecStart=/opt/nautobot/venv/bin/celery -A nautobot.core.celery beat \
    --loglevel INFO \
    --logfile /opt/nautobot/logs/celery-beat.log

Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

Systemd reload করুন:

```bash
sudo systemctl daemon-reload
```

Services চালু করুন:

```bash
sudo systemctl start nautobot
sudo systemctl start nautobot-celery
sudo systemctl start nautobot-celery-beat
```

Boot-এ automatically চালু হওয়ার জন্য enable করুন:

```bash
sudo systemctl enable nautobot
sudo systemctl enable nautobot-celery
sudo systemctl enable nautobot-celery-beat
```

Status চেক করুন:

```bash
sudo systemctl status nautobot
```

দেখাবে: `Active: active (running)`

---