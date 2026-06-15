# Frontend Deployment Guide (Next.js)

This step-by-step guide is designed to help you deploy a Next.js frontend application to an Ubuntu server using **PM2**, **PMX (PM2 Plus)**, and **Nginx** as a reverse proxy.

---

## Table of Contents
1. [Prerequisites & Gathering Credentials](#1-prerequisites--gathering-credentials)
2. [Login to Server](#2-login-to-server)
3. [System Preparation & Node.js Installation](#3-system-preparation--nodejs-installation)
4. [Project Directory & Git Setup](#4-project-directory--git-setup)
5. [Environment Variables Setup](#5-environment-variables-setup)
6. [Build and Test the Application](#6-build-and-test-the-application)
7. [Install and Configure PM2 & PMX (PM2 Plus)](#7-install-and-configure-pm2--pmx-pm2-plus)
8. [Configure Nginx Reverse Proxy](#8-configure-nginx-reverse-proxy)
9. [Enable SSL (HTTPS) with Certbot](#9-enable-ssl-https-with-certbot)

---

## 1. Prerequisites & Gathering Credentials

Before starting, make sure you collect the following credentials and access details:

| Credential / Info | Description | Example |
|---|---|---|
| **Server IP Address** | The public IP of your Ubuntu Server | `192.168.1.100` or `203.0.113.5` |
| **SSH Key or Password** | SSH Private Key file (`.pem` / `.ppk`) or root/sudo password | `my-server-key.pem` |
| **SSH Username** | Standard Ubuntu admin username (usually `ubuntu` or `root`) | `ubuntu` |
| **Git Repository URL** | HTTPS or SSH link to the frontend repository | `git@github.com:username/repo.git` |
| **Domain Name** | Domain name pointed to the Server IP (DNS A Record) | `frontend.example.com` |
| **Environment Variables** | All production environment variables (`.env.production`) | `NEXT_PUBLIC_API_URL=...` |

---

## 2. Login to Server

Open your terminal (on macOS/Linux) or PowerShell/Git Bash (on Windows) and SSH into your Ubuntu server.

```bash
# If using an SSH key file:
ssh -i /path/to/your-key.pem ubuntu@your_server_ip

# If using a password:
ssh ubuntu@your_server_ip
```

> [!NOTE]
> Replace `ubuntu` with your server's username and `your_server_ip` with your actual server IP.

<!-- SCREENSHOT_PLACEHOLDER: Terminal showing successful SSH login connection -->
<!-- Please paste your screenshot of the successful login below: -->
<!-- <img src="./screenshots/ssh_login.png" alt="SSH Login" width="700"/> -->

---

## 3. System Preparation & Node.js Installation

Update the package registry and install Node.js. It is highly recommended to use **NVM (Node Version Manager)** to easily manage Node versions.

### 3.1 Update System Packages
```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2 Install NVM and Node.js
```bash
# Download and install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Load NVM into the current session
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# Install the recommended Node.js LTS version (e.g., v20)
nvm install 20

# Verify installation
node -v
npm -v
```

<!-- SCREENSHOT_PLACEHOLDER: Terminal showing output of node -v and npm -v -->
<!-- Please paste your screenshot showing installed node and npm versions below: -->
<!-- <img src="./screenshots/node_version.png" alt="Node & NPM Versions" width="700"/> -->

---

## 4. Project Directory & Git Setup

We will create a directory for the application, clone the repository, and pull the latest code.

### 4.1 Create App Directory
```bash
# Create directory under /var/www (best practice) or home folder
sudo mkdir -p /var/www/nextjs-app
sudo chown -R $USER:$USER /var/www/nextjs-app
cd /var/www/nextjs-app
```

### 4.2 Initialize Git and Clone/Pull Code
If it is a private repository, you need to generate an SSH key on the server and add it to your GitHub/GitLab repository settings:
```bash
# Generate SSH key (press Enter to accept defaults)
ssh-keygen -t ed25519 -C "server-deploy@example.com"

# View the public key to copy it
cat ~/.ssh/id_ed25519.pub
```
*Add this public key to GitHub Deploy Keys on your repository settings.*

<!-- SCREENSHOT_PLACEHOLDER: GitHub Deploy Keys interface or terminal displaying SSH public key -->
<!-- Please paste your screenshot of adding SSH deploy key to GitHub below: -->
<!-- <img src="./screenshots/github_deploy_key.png" alt="GitHub Deploy Keys" width="700"/> -->

Clone the repository:
```bash
# Clone the repository directly into the folder
git clone git@github.com:username/repo.git .

# If repository is already cloned, navigate into it and pull latest changes
git pull origin main
```

<!-- SCREENSHOT_PLACEHOLDER: Git clone / git pull command success in terminal -->
<!-- Please paste your screenshot of git clone/pull success below: -->
<!-- <img src="./screenshots/git_pull.png" alt="Git Pull Success" width="700"/> -->

---

## 5. Environment Variables Setup

Create the `.env.production` file to hold client-side and server-side environment configurations.

```bash
nano .env.production
```

Inside the file, paste your environment variables:
```env
PORT=3000
NODE_ENV=production
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
# Add other environment variables here
```
Press `Ctrl + O`, `Enter` to save, and `Ctrl + X` to exit.

---

## 6. Build and Test the Application

Now we will install dependencies, build the Next.js production bundle, and run it locally to verify.

### 6.1 Install Dependencies
```bash
npm install --production=false
# Or if using yarn: yarn install
```

### 6.2 Build the Project
Next.js compiles HTML, CSS, and optimized assets into the `.next` folder.
```bash
npm run build
```

<!-- SCREENSHOT_PLACEHOLDER: Terminal showing successful npm run build output -->
<!-- Please paste your screenshot of the successful build output below: -->
<!-- <img src="./screenshots/npm_build.png" alt="NextJS Build Output" width="700"/> -->

### 6.3 Test Locally
Ensure the build runs correctly on the server:
```bash
npm run start
```
*Test via another terminal or curl, then stop it using `Ctrl + C`.*

---

## 7. Install and Configure PM2 & PMX (PM2 Plus)

PM2 is a production process manager that keeps your Next.js application alive 24/7 and restarts it if it crashes.

### 7.1 Install PM2 Globally
```bash
npm install pm2 -g
```

### 7.2 Start Application with PM2
We start the production Next.js server using PM2. You can define a configuration file `ecosystem.config.js` or start it directly.

```bash
# Start directly:
pm2 start npm --name "nextjs-app" -- start

# Or start on specific port:
PORT=3000 pm2 start npm --name "nextjs-app" -- start
```

### 7.3 Enable Startup Script
To make sure PM2 restarts and boots your app automatically when the Ubuntu server restarts:
```bash
pm2 startup systemd
```
*Copy and paste the command generated by the output of the command above and run it (it starts with `sudo env PATH=...`).*

Save the PM2 current process list:
```bash
pm2 save
```

<!-- SCREENSHOT_PLACEHOLDER: PM2 process list showing "nextjs-app" is online -->
<!-- Please paste your screenshot showing pm2 status and list below: -->
<!-- <img src="./screenshots/pm2_list.png" alt="PM2 Process List" width="700"/> -->

---

### 7.4 Connect to PMX (PM2 Plus / Keymetrics) for Monitoring

PM2 Plus (PMX) allows you to monitor memory leaks, exceptions, CPU utilization, and requests in a real-time web dashboard.

1. Go to [PM2 Plus Dashboard](https://pm2.io/) and create an account.
2. Click on **Create Bucket** and name it (e.g. `NextJS-Production`).
3. Under the bucket options, you will receive a linking command with a Secret Key and Public Key:
   ```bash
   pm2 link <secret_key> <public_key>
   ```
4. Copy and execute that command on your Ubuntu server terminal.

Once connected, your dashboard on pm2.io will immediately start displaying real-time metrics.

<!-- SCREENSHOT_PLACEHOLDER: PM2 Plus web interface dashboard showing connected server metrics -->
<!-- Please paste your screenshot of PM2 Plus (PMX) dashboard below: -->
<!-- <img src="./screenshots/pm2_plus_dashboard.png" alt="PM2 Plus Dashboard" width="700"/> -->

---

## 8. Configure Nginx Reverse Proxy

Nginx acts as a reverse proxy, listening on port 80/443 (HTTP/HTTPS) and forwarding requests to your Next.js application running on port 3000.

### 8.1 Install Nginx
```bash
sudo apt install nginx -y
```

### 8.2 Create Nginx Block Configuration
```bash
sudo nano /etc/nginx/sites-available/nextjs-app
```

Paste the configuration below (replace `frontend.example.com` with your actual domain):
```nginx
server {
    listen 80;
    server_name frontend.example.com;

    # Gzip settings for Next.js performance
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    location / {
        proxy_pass http://localhost:3000;
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

### 8.3 Enable Nginx Configuration and Restart
```bash
# Enable the configuration by creating a symlink
sudo ln -s /etc/nginx/sites-available/nextjs-app /etc/nginx/sites-enabled/

# Test Nginx syntax
sudo nginx -t
```
If test is successful, restart Nginx:
```bash
sudo systemctl restart nginx
```

<!-- SCREENSHOT_PLACEHOLDER: Nginx -t output showing syntax is OK and successful test -->
<!-- Please paste your screenshot of Nginx configuration test success below: -->
<!-- <img src="./screenshots/nginx_test.png" alt="Nginx Test Success" width="700"/> -->

---

## 9. Enable SSL (HTTPS) with Certbot

To secure your site with free Let's Encrypt SSL:

### 9.1 Install Certbot
```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 9.2 Request SSL Certificate
```bash
sudo certbot --nginx -d frontend.example.com
```
*Follow the on-screen prompts. Enter your email, agree to terms, and choose whether to redirect HTTP traffic to HTTPS (highly recommended).*

### 9.3 Test Automatic SSL Renewal
Let's Encrypt certificates last 90 days. The certbot package automatically handles renewals, check it with:
```bash
sudo certbot renew --dry-run
```

Your Next.js frontend is now securely deployed and monitored via PM2 Plus (PMX)!

<!-- SCREENSHOT_PLACEHOLDER: Web browser loading your domain name with the secure padlock icon (HTTPS) -->
<!-- Please paste your screenshot of your live website running securely below: -->
<!-- <img src="./screenshots/ssl_verification.png" alt="SSL Verification" width="700"/> -->
