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


-- 3. Grant privileges to user on the database
GRANT ALL PRIVILEGES ON DATABASE my_django_db TO django_db_user;

-- For PostgreSQL 15+, you may also need to grant schema permissions:
\c my_django_db
GRANT ALL ON SCHEMA public TO django_db_user;

-- 4. Exit PostgreSQL prompt
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

### 5. Local Project Configuration & Push to Git

Before configuring the server, return to your **local PC** (where your project code is located) to install production packages, update settings, and push the updated codebase to Git.

### 5.1 Update Database Credentials Locally
Open your project's configuration (or `.env` file) on your local machine and update it with the credentials of the database you just created on the server:
```env
DB_NAME=my_django_db
DB_USER=django_db_user
DB_PASSWORD=SuperSecurePassword123
DB_HOST=your_server_ip_address
DB_PORT=5432
```

### 5.2 Install Production Dependencies
Run the following commands in your local project terminal to install Gunicorn (the WSGI server) and WhiteNoise (for serving static styling/admin CSS files in production):
```bash
pip install gunicorn whitenoise
```

### 5.3 Modify `settings.py` for Production
Open your Django `settings.py` and modify the following configuration values:

```python
# settings.py

# 1. Disable debug mode
DEBUG = False

# 2. Add your server domain name and IP address
ALLOWED_HOSTS = ['api.example.com', 'your_server_ip_address', 'localhost', '127.0.0.1']

# 3. Add WhiteNoise middleware right after SecurityMiddleware
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # WhiteNoise Middleware
    # ... other middlewares ...
]

# 4. Configure WhiteNoise static file storage settings
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

### 5.4 Export Requirements and Push Code
Export the updated list of dependencies to `requirements.txt`, commit, and push the updates to your Git repository (GitHub/GitLab):
```bash
# Export packages
pip freeze > requirements.txt

# Commit and Push to Git
git add .
git commit -m "Configure production database, allowed hosts, and whitenoise"
git push origin main
```

---

## 6. Server Repository Setup, Pull, & Dependencies

Now, return to your server terminal session.

### 6.1 Create Directory & Clone Project (First-time setup)
If you haven't cloned the project on the server yet:
```bash
sudo mkdir -p /home/ubuntu/backend
sudo chown -R $USER:$USER /home/ubuntu/backend
cd /home/ubuntu/backend

# Clone your project repository
git clone git@github.com:username/backend-repo.git
```

### 6.2 Pull the Code on Server
Navigate into the repository directory and pull your latest changes:
```bash
cd /home/ubuntu/backend/project_repo_name
git pull
```

### 6.3 Create Environment File (`.env`)
Create a `.env` file in the folder to securely pass configuration to Django:
```bash
nano .env
```
Paste your production settings:
```env
DEBUG=False
SECRET_KEY=your_django_production_secret_key
ALLOWED_HOSTS=api.example.com,your_server_ip_address,localhost,127.0.0.1

DB_NAME=my_django_db
DB_USER=django_db_user
DB_PASSWORD=SuperSecurePassword123
DB_HOST=127.0.0.1
DB_PORT=5432
```
Press `Ctrl + O`, `Enter` to save, and `Ctrl + X` to exit.

### 6.4 Configure Virtual Environment & Install Requirements
Create a clean environment, activate it, and install python dependencies:
```bash
# Create venv
python3 -m venv venv

# Activate venv
source venv/bin/activate

# Install dependencies (make sure psycopg2 or psycopg2-binary, whitenoise, and gunicorn are installed)
pip install --upgrade pip
pip install -r requirements.txt
```

### 6.5 Run Migrations and Collect Static Files
Compile your static admin styling sheets and migrate the Postgres database:
```bash
# Apply migrations to PostgreSQL
python manage.py migrate

# Gather static CSS/JS files (including django admin styles)
python manage.py collectstatic --no-input
```

---

## 7. Install and Configure PM2 for Backend

Just like the frontend, we will use **PM2** as the process manager to run the Gunicorn backend server in the background and keep it alive.

### 7.1 Verify or Install PM2 Globally
Check if PM2 is installed on the server. If not installed, install it:
```bash
# Check if pm2 exists
pm2 -v

# If not installed, install it:
sudo npm install pm2 -g
```

### 7.2 Start Django Backend with PM2
Create an `ecosystem.config.js` file in your backend project directory:
```bash
nano ecosystem.config.js
```

Paste the configuration below (replace `myproject.wsgi:application` with the directory name containing your `wsgi.py` file):
```javascript
module.exports = {
  apps: [{
    name: "backend",
    script: "venv/bin/gunicorn",
    args: "--workers 3 --bind 127.0.0.1:8000 myproject.wsgi:application"
  }]
};
```
Press `Ctrl + O`, `Enter` to save, and `Ctrl + X` to exit.

Now, start your backend application using PM2:
```bash
pm2 start ecosystem.config.js
```

### 7.3 Save PM2 Process List and Verify Status
To ensure that PM2 restarts your backend automatically when the Ubuntu server restarts:
```bash
# Save the current list of running PM2 applications
pm2 save

# Verify application status
pm2 status
```
*Make sure the status column of the process `backend` shows `online`.*

<!-- SCREENSHOT_PLACEHOLDER: PM2 status list showing both "frontend" and "backend" processes are online -->
<!-- Please paste your screenshot showing pm2 status and list below: -->
<!-- <img src="./screenshots/pm2_backend_status.png" alt="PM2 Backend Status" width="700"/> -->

---

## 8. Domain Setup & Reverse Proxy (CloudPanel vs Manual Nginx)

Next, we must configure how web traffic from the internet reaches the Gunicorn backend running on port `8000`.

### Option A: Server has CloudPanel Installed
If you are using **CloudPanel** to manage your server:
1. Log in to your CloudPanel admin dashboard.
2. Click **Add Site** -> Select **Reverse Proxy**.
3. Fill out the following:
   * **Domain Name**: `api.example.com`
   * **Reverse Proxy Port**: `8000`
4. Click **Create** or **Save**.
5. Navigate to the site settings in CloudPanel, go to the **SSL/TLS** tab, and generate a free Let's Encrypt certificate.
*If using CloudPanel, you can skip Option B (Manual Nginx Setup) and Certbot manual setup entirely.*

<!-- SCREENSHOT_PLACEHOLDER: CloudPanel dashboard showing site creation for reverse proxy on port 8000 -->
<!-- Please paste your screenshot of CloudPanel Reverse Proxy settings below: -->
<!-- <img src="./screenshots/cloudpanel_backend_proxy.png" alt="CloudPanel Reverse Proxy" width="700"/> -->

---

### Option B: Manual Nginx Setup (No CloudPanel)
If CloudPanel is not used, follow these manual configuration steps:

#### 8.1 Verify or Install Nginx
Check if Nginx is installed on the server. If not, install it:
```bash
# Check if nginx exists
nginx -v

# If not installed, install it:
sudo apt update && sudo apt install nginx -y
```

#### 8.2 Create Nginx Server Block Configuration
Create a configuration file using your domain name as the filename:
```bash
sudo nano /etc/nginx/sites-available/api.example.com
```

Paste the configuration below (replace `api.example.com` with your actual domain):

```nginx
server {
    listen 80;
    server_name api.example.com;


    # Django Media Uploads (if applicable)
    location /media/ {
        alias /home/ubuntu/backend/project_repo_name/media/;
    }

    max_body_size 10M;

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        
        # Real IP headers
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Press `Ctrl + O`, `Enter` to save, and `Ctrl + X` to exit.

#### 8.3 Create a Soft Link to Enable the Site
Create a symbolic link from `sites-available` to `sites-enabled`:
```bash
sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
```

#### 8.4 Test Nginx Syntax and Restart
Always verify the configuration syntax is correct before restarting:
```bash
sudo nginx -t
```
*Make sure it prints `syntax is ok` and `test is successful`.*

If the test is successful, reload Nginx to apply changes:
```bash
sudo systemctl restart nginx
```

<!-- SCREENSHOT_PLACEHOLDER: Terminal showing sudo nginx -t output and systemctl restart nginx command execution -->
<!-- Please paste your screenshot of Nginx configuration test success below: -->
<!-- <img src="./screenshots/nginx_backend_test.png" alt="Nginx Test Success" width="700"/> -->

---

## 9. Manual SSL (HTTPS) Configuration with Certbot

If you configured Nginx manually (Option B), secure the domain with free Let's Encrypt SSL.

### 9.1 Install Certbot Nginx Extension
```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 9.2 Request and Configure SSL
Run the interactive wizard to generate the SSL certificate and let Certbot automatically update Nginx to redirect HTTP to HTTPS:
```bash
sudo certbot --nginx -d api.example.com
```
*Follow the terminal instructions (enter email, agree to terms, choose redirect option).*

### 9.3 Verify Auto-renewal
Let's Encrypt certificates expire in 90 days. Run a test to verify the automated renewal cronjob is working:
```bash
sudo certbot renew --dry-run
```

<!-- SCREENSHOT_PLACEHOLDER: Web browser URL bar showing the secure padlock icon on your domain next to your site URL -->
<!-- Please paste your screenshot of your live website running securely below: -->
<!-- <img src="./screenshots/ssl_backend_verification.png" alt="SSL Verification" width="700"/> -->

---

## 10. Automate Deployment with GitHub Actions

To automate your backend deployment, you can set up a GitHub Actions workflow to pull the latest code, run database migrations, collect static files, and restart the backend service.

### 10.1 Set Up Secrets on GitHub
1. Go to your backend repository on GitHub.
2. Navigate to **Settings** -> **Secrets and variables** -> **Actions**.
3. Add the following repository secrets by clicking **New repository secret**:
   * `SSH_HOST`: Your backend server IP address (e.g. `203.0.113.10`).
   * `SSH_USER`: The SSH username (e.g. `ubuntu`).
   * `SSH_KEY`: The server's SSH Private Key (contents of your `.pem` key file). Make sure it includes the `BEGIN` and `END` lines.

### 10.2 Create the Workflow Configuration
On your local computer, create a new directory and configuration file at `.github/workflows/deploy.yml` in the root of your backend project:

```bash
mkdir -p .github/workflows
nano .github/workflows/deploy.yml
```

Paste the following YAML configuration inside the file:
```yaml
name: Deploy Django Backend

on:
  push:
    branches:
      - main  # Trigger workflow when pushing to main branch

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Deploy to Ubuntu Server via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            # Navigate to project folder on server
            cd /home/ubuntu/backend/project_repo_name

            # Pull latest changes from the repository
            git pull origin main

            # Activate Python Virtual Environment
            source venv/bin/activate

            # Upgrade pip and install dependencies
            pip install --upgrade pip
            pip install -r requirements.txt

            # Run Django migrations (if any are available)
            python manage.py makemigrations
            python manage.py migrate

            # Collect static files (like admin CSS/JS assets)
            python manage.py collectstatic --no-input

            # Restart the backend application using PM2 config
            pm2 restart ecosystem.config.js
```
*(Note: Replace `project_repo_name` with your actual project directory name).*

### 10.3 Commit and Push
Commit the new workflow file and push it:
```bash
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Actions CD deployment workflow"
git push origin main
```
Your backend application is now fully automated! Whenever you push code, GitHub Actions will trigger, connect to the server, install new packages, run database migrations, update static files, and restart the backend PM2 process automatically.

