# Backend Deployment Guide (Django & PostgreSQL)

This step-by-step guide is designed to help you deploy a Django Python backend application on an Ubuntu server, configure a **PostgreSQL** database, enable remote access to PostgreSQL, connect it to the project, and serve it via **Gunicorn** and **Nginx** reverse proxy.

---

## Table of Contents
1. [Prerequisites & Gathering Credentials](#1-prerequisites--gathering-credentials)
2. [Login to Server](#2-login-to-server)
3. [Install and Configure PostgreSQL](#3-install-and-configure-postgresql)
4. [Enable Remote Database Connections (Access from Anywhere)](#4-enable-remote-database-connections-access-from-anywhere)
5. [Git Repository Setup](#5-git-repository-setup)
6. [Configure Python Virtual Environment & Install Dependencies](#6-configure-python-virtual-environment--install-dependencies)
7. [Configure Django Settings (`settings.py`)](#7-configure-django-settings-settingspy)
8. [Run Django Migrations and Collect Static Files](#8-run-django-migrations-and-collect-static-files)
9. [Configure Gunicorn Daemon (Systemd Service)](#9-configure-gunicorn-daemon-systemd-service)
10. [Configure Nginx Reverse Proxy](#10-configure-nginx-reverse-proxy)
11. [Enable SSL (HTTPS) with Certbot](#11-enable-ssl-https-with-certbot)

---

## 1. Prerequisites & Gathering Credentials

Collect the following details before starting:

| Credential / Info | Description | Example |
|---|---|---|
| **Server IP Address** | Public IP address of the backend Ubuntu server | `203.0.113.10` |
| **SSH Key or Password** | Private SSH Key (`.pem`) or password to log in | `my-backend-key.pem` |
| **SSH Username** | Usually `ubuntu` or `root` | `ubuntu` |
| **Database Name** | Name of the PostgreSQL database to create | `my_django_db` |
| **Database Username** | Username of the PostgreSQL user | `django_db_user` |
| **Database Password** | Strong password for the database user | `SuperSecurePassword123` |
| **Domain / Subdomain** | DNS A record pointed to the backend IP | `api.example.com` |

---

## 2. Login to Server

Open your terminal or command prompt and log into your server:

```bash
# If using an SSH key:
ssh -i /path/to/your-key.pem ubuntu@your_backend_ip

# If using a password:
ssh ubuntu@your_backend_ip
```

---

## 3. Install and Configure PostgreSQL

### 3.1 Install PostgreSQL
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

### 3.2 Access PostgreSQL Prompt
Log in as the default administrative `postgres` Linux user and launch the SQL shell:
```bash
sudo -i -u postgres psql
```

### 3.3 Create Database, User, and Grant Privileges
In the PostgreSQL prompt (`postgres=#`), execute the following commands:
```sql
-- 1. Create the database
CREATE DATABASE my_django_db;

-- 2. Create database user with a secure password
CREATE USER django_db_user WITH PASSWORD 'SuperSecurePassword123';

-- 3. Set standard optimized configurations for Django
ALTER ROLE django_db_user SET client_encoding TO 'utf8';
ALTER ROLE django_db_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE django_db_user SET timezone TO 'UTC';

-- 4. Grant privileges to user on the database
GRANT ALL PRIVILEGES ON DATABASE my_django_db TO django_db_user;

-- For PostgreSQL 15+, you may also need to grant schema permissions:
\c my_django_db
GRANT ALL ON SCHEMA public TO django_db_user;

-- 5. Exit PostgreSQL prompt
\q
```

<!-- SCREENSHOT_PLACEHOLDER: PostgreSQL CLI shell showing successful creation of DB, User, and Granting permissions -->
<!-- Please paste your screenshot of the PostgreSQL SQL prompt commands below: -->
<!-- <img src="./screenshots/postgres_setup.png" alt="PostgreSQL Setup" width="700"/> -->

---

## 4. Enable Remote Database Connections (Access from Anywhere)

> [!WARNING]
> Allowing database access from anywhere (`0.0.0.0/0`) presents a security risk. Make sure your database user has a very strong password. If possible, restrict access to the Frontend server IP only.

### 4.1 Edit `postgresql.conf`
Find the configuration file path (usually under `/etc/postgresql/<version>/main/postgresql.conf`).
```bash
# Check installed PostgreSQL version first
psql --version

# Edit postgresql.conf (Replace '15' or '16' with your installed version)
sudo nano /etc/postgresql/15/main/postgresql.conf
```
Find the line `#listen_addresses = 'localhost'` (around line 60), uncomment it (remove `#`), and change it to listen on all interfaces:
```ini
listen_addresses = '*'
```
Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

### 4.2 Edit `pg_hba.conf`
`pg_hba.conf` manages client authentication.
```bash
sudo nano /etc/postgresql/15/main/pg_hba.conf
```
Scroll to the end of the file and add the remote access rule:

```conf
# To allow access from ANYWHERE (0.0.0.0/0):
host    all             all             0.0.0.0/0               scram-sha-256

# OR (RECOMMENDED) To allow access ONLY from your Frontend Server IP:
# host    all             all             <frontend_server_ip>/32      scram-sha-256
```

### 4.3 Open Port 5432 on Ubuntu Firewall (UFW)
Ensure external systems can connect via port 5432:
```bash
sudo ufw allow 5432/tcp
```

### 4.4 Restart PostgreSQL Service
Apply changes by restarting PostgreSQL:
```bash
sudo systemctl restart postgresql
```

<!-- SCREENSHOT_PLACEHOLDER: Terminal showing postgresql.conf, pg_hba.conf updates, and service restart commands -->
<!-- Please paste your screenshot of editing configurations and restarting postgresql below: -->
<!-- <img src="./screenshots/postgres_remote_config.png" alt="PostgreSQL Remote Config" width="700"/> -->

---

## 5. Git Repository Setup

### 5.1 System dependencies
Install Python development headers, pip, venv, and PostgreSQL client library compilation requirements:
```bash
sudo apt install python3-pip python3-venv python3-dev libpq-dev git curl -y
```

### 5.2 Clone the Repository
```bash
sudo mkdir -p /var/www/django-backend
sudo chown -R $USER:$USER /var/www/django-backend
cd /var/www/django-backend

# Generate deployment keys if needed, copy to GitHub, and clone repo
git clone git@github.com:username/backend-repo.git .
```

---

## 6. Configure Python Virtual Environment & Install Dependencies

Create and activate virtual environment to isolate backend dependencies:

```bash
# Create venv
python3 -m venv venv

# Activate venv
source venv/bin/activate

# Install dependencies (make sure psycopg2 or psycopg2-binary and gunicorn are in requirements)
pip install --upgrade pip
pip install -r requirements.txt
```

<!-- SCREENSHOT_PLACEHOLDER: Terminal showing successful pip install -r requirements.txt output -->
<!-- Please paste your screenshot of package installations below: -->
<!-- <img src="./screenshots/pip_install.png" alt="Pip Install Dependencies" width="700"/> -->

---

## 7. Configure Django Settings (`settings.py`)

Make edits inside your Django project's `settings.py` file to prepare it for production.

```bash
nano path/to/your/project/settings.py
```

### 7.1 Security Settings
Ensure `DEBUG` is set to `False` and configure `ALLOWED_HOSTS` with your server domain or IP:
```python
# settings.py

# CRITICAL: Turn off Debug in production
DEBUG = False

# Add your domain name and server IP address
ALLOWED_HOSTS = ['api.example.com', 'your_backend_server_ip', 'localhost', '127.0.0.1']
```

### 7.2 Database Configuration
Change `DATABASES` settings to use PostgreSQL instead of default SQLite:
```python
# settings.py

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'my_django_db',
        'USER': 'django_db_user',
        'PASSWORD': 'SuperSecurePassword123',
        'HOST': 'localhost',  # Or 'your_database_server_ip' if database is hosted on a separate server
        'PORT': '5432',
    }
}
```

### 7.3 Static and Media Files Path Configuration
Ensure paths are configured so Nginx can locate and serve static assets directly:
```python
# settings.py
import os

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

---

## 8. Run Django Migrations and Collect Static Files

Before starting the server, initialize the database schema and gather all static CSS/JS/images.

```bash
# Make sure virtual environment is active
source venv/bin/activate

# 8.1 Apply database migrations
python manage.py migrate

# 8.2 Create superuser (interactive command)
python manage.py createsuperuser

# 8.3 Collect static files
python manage.py collectstatic --no-input
```

<!-- SCREENSHOT_PLACEHOLDER: Database migrations running successfully and collectstatic completion in terminal -->
<!-- Please paste your screenshot of migrations and collectstatic output below: -->
<!-- <img src="./screenshots/django_migrations.png" alt="Django Migrations and Static Files" width="700"/> -->

---

## 9. Configure Gunicorn Daemon (Systemd Service)

Gunicorn runs the WSGI server application. We will manage Gunicorn using Ubuntu systemd services so it starts automatically at boot.

### 9.1 Create Gunicorn Socket
Create a systemd socket file to listen for connections:
```bash
sudo nano /etc/systemd/system/gunicorn.socket
```
Paste this configuration:
```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

### 9.2 Create Gunicorn Service
Create the service configuration file:
```bash
sudo nano /etc/systemd/system/gunicorn.service
```
Paste this configuration (adjust paths and usernames to match your setup):
```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/var/www/django-backend
ExecStart=/var/www/django-backend/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```
> [!NOTE]
> Replace `myproject.wsgi:application` with the directory name containing your `wsgi.py` file.

### 9.3 Start and Enable Gunicorn
```bash
# Start socket
sudo systemctl start gunicorn.socket

# Enable socket to run at boot
sudo systemctl enable gunicorn.socket

# Reload daemon to apply changes
sudo systemctl daemon-reload

# Restart gunicorn service
sudo systemctl restart gunicorn
```

Verify service status:
```bash
sudo systemctl status gunicorn.socket
sudo systemctl status gunicorn.service
```

<!-- SCREENSHOT_PLACEHOLDER: Systemctl output showing active (running) status of Gunicorn socket and service -->
<!-- Please paste your screenshot of systemctl status output below: -->
<!-- <img src="./screenshots/gunicorn_status.png" alt="Gunicorn Service Status" width="700"/> -->

---

## 10. Configure Nginx Reverse Proxy

Nginx will handle requests for static and media assets and proxy all other backend queries to the Gunicorn socket.

### 10.1 Install Nginx (if not installed)
```bash
sudo apt install nginx -y
```

### 10.2 Create Nginx Server Block
```bash
sudo nano /etc/nginx/sites-available/django-backend
```
Paste the configuration below (replace `api.example.com` with your actual domain):
```nginx
server {
    listen 80;
    server_name api.example.com;

    # Serve favicon, if it exists
    location = /favicon.ico { access_log off; log_not_found off; }

    # Django Static Files
    location /static/ {
        alias /var/www/django-backend/staticfiles/;
    }

    # Django Media Uploads
    location /media/ {
        alias /var/www/django-backend/media/;
    }

    # Proxy rest of requests to Gunicorn socket
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

### 10.3 Enable Configuration and Reload Nginx
```bash
# Link the site configuration to enable it
sudo ln -s /etc/nginx/sites-available/django-backend /etc/nginx/sites-enabled/

# Test syntax correctness
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

---

## 11. Enable SSL (HTTPS) with Certbot

Secure the backend endpoint with Let's Encrypt SSL:

### 11.1 Install Certbot Nginx plugin
```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 11.2 Request Certificate
```bash
sudo certbot --nginx -d api.example.com
```

### 11.3 Verify Renewal Service
Check automatic renewal:
```bash
sudo certbot renew --dry-run
```

Your Django backend and PostgreSQL database are now deployed securely!

<!-- SCREENSHOT_PLACEHOLDER: Terminal/Browser showing HTTPS response or Django Rest Framework API page running securely -->
<!-- Please paste your screenshot of active API / admin page securely running below: -->
<!-- <img src="./screenshots/backend_ssl_verify.png" alt="Backend Live Verification" width="700"/> -->
