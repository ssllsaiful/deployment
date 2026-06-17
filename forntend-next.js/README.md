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
sudo mkdir -p /home/ubuntu/frontend
sudo chown -R $USER:$USER /home/ubuntu/frontend
cd /home/ubuntu/frontend
```
![alt text](imagejjjj.png)
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
Create a new file name in the project directory (next to the folder you just cloned)  `.env` and copy all the environment variables from `.env` file.

```bash
nano .env
```

Inside the file, paste your environment variables Example:
```env

NODE_ENV=production
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
```
Press `Ctrl + O`, `Enter` to save, and `Ctrl + X` to exit.

---

## 6. Project Setup, Build, and PM2 Process Management

In this section, we will install **PM2** globally, install project dependencies, build the Next.js bundle, run the app using PM2, and verify its running status.

### 6.1 Install PM2 Globally
PM2 is a production process manager that keeps your Next.js application alive 24/7.
```bash
sudo npm install pm2 -g
```

### 6.2 Install Project Dependencies
Navigate to your project directory (if not already there) and install the necessary node modules:
```bash
# Make sure you are in the project folder
cd /home/ubuntu/frontend/projectname

# Install dependencies
npm install
```

### 6.3 Build the Project
Next.js compiles HTML, CSS, and optimized assets into the `.next` folder for production:
```bash
npm run build
```

<!-- SCREENSHOT_PLACEHOLDER: Terminal showing successful npm run build output -->
<!-- Please paste your screenshot of the successful build output below: -->
<!-- <img src="./screenshots/npm_build.png" alt="NextJS Build Output" width="700"/> -->

### 6.4 Start Next.js with PM2
Create an `ecosystem.config.js` file in your project directory:
```bash
nano ecosystem.config.js
```

Paste the configuration  below (Example):
```javascript
module.exports = {
  apps: [{
    name: "frontend",
    script: "npm",
    args: "start"
  }]
};
```
Press `Ctrl + O`, `Enter` to save, and `Ctrl + X` to exit.

Now, start the application using PM2:
```bash
pm2 start ecosystem.config.js
```

### 6.5 Save PM2 Process List and Verify Status
To ensure that PM2 starts your application automatically when the Ubuntu server restarts:
```bash
# Generate startup configuration
pm2 startup systemd
# (Note: Copy and run the command printed by the command output starting with 'sudo env PATH=...')

# Save the current list of running PM2 applications
pm2 save

# Verify application status
pm2 status
```
*Make sure the status column of the process `frontend` shows `online`.*

<!-- SCREENSHOT_PLACEHOLDER: PM2 status list showing the "frontend" app is active and online -->
<!-- Please paste your screenshot showing pm2 status and list below: -->
<!-- <img src="./screenshots/pm2_status.png" alt="PM2 Status List" width="700"/> -->


## 7. Domain Setup & Reverse Proxy (CloudPanel vs Manual Nginx)

Next, we must configure how web traffic from the internet reaches the Next.js app running on port `3000`.

### Option A: Server has CloudPanel Installed
If you are using **CloudPanel** to manage your server:
1. Log in to your CloudPanel admin dashboard.
2. Click **Add Site** -> Select **Reverse Proxy**.
3. Fill out the following:
   * **Domain Name**: `frontend.example.com`
   * **Reverse Proxy Port**: `3000`
4. Click **Create** or **Save**.
5. Navigate to the site settings in CloudPanel, go to the **SSL/TLS** tab, and generate a free Let's Encrypt certificate.
*If using CloudPanel, you can skip Option B (Manual Nginx Setup) and Certbot manual setup entirely.*

<!-- SCREENSHOT_PLACEHOLDER: CloudPanel dashboard showing site creation for reverse proxy on port 3000 -->
<!-- Please paste your screenshot of CloudPanel Reverse Proxy settings below: -->
<!-- <img src="./screenshots/cloudpanel_proxy.png" alt="CloudPanel Reverse Proxy" width="700"/> -->

---

### Option B: Manual Nginx Setup (No CloudPanel)
If CloudPanel is not used, follow these manual configuration steps:

#### 7.1 Verify or Install Nginx
Check if Nginx is installed on the server. If not, install it:
```bash
# Check if nginx exists
nginx -v

# If not installed, install it:
sudo apt update && sudo apt install nginx -y
```

#### 7.2 Create Nginx Server Block Configuration
Create a configuration file using your domain name as the filename:
```bash
sudo nano /etc/nginx/sites-available/frontend.example.com
```

Paste the configuration below (replace `frontend.example.com` with your actual domain):
```nginx
server {
    listen 80;
    server_name frontend.example.com;

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
Press `Ctrl + O`, `Enter` to save, and `Ctrl + X` to exit.

#### 7.3 Create a Soft Link to Enable the Site
Create a symbolic link from `sites-available` to `sites-enabled`:
```bash
sudo ln -s /etc/nginx/sites-available/frontend.example.com /etc/nginx/sites-enabled/
```

#### 7.4 Test Nginx Syntax and Restart
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
<!-- <img src="./screenshots/nginx_test.png" alt="Nginx Test Success" width="700"/> -->

---

## 8. Manual SSL (HTTPS) Configuration with Certbot

If you configured Nginx manually (Option B), secure the domain with free Let's Encrypt SSL.

### 8.1 Install Certbot Nginx Extension
```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 8.2 Request and Configure SSL
Run the interactive wizard to generate the SSL certificate and let Certbot automatically update Nginx to redirect HTTP to HTTPS:
```bash
sudo certbot --nginx -d frontend.example.com
```
*Follow the terminal instructions (enter email, agree to terms, choose redirect option).*

### 8.3 Verify Auto-renewal
Let's Encrypt certificates expire in 90 days. Run a test to verify the automated renewal cronjob is working:
```bash
sudo certbot renew --dry-run
```

<!-- SCREENSHOT_PLACEHOLDER: Web browser URL bar showing the secure padlock icon on your domain next to your site URL -->
<!-- Please paste your screenshot of your live website running securely below: -->
<!-- <img src="./screenshots/ssl_verification.png" alt="SSL Verification" width="700"/> -->

---

## 9. Automate Deployment with GitHub Actions

To set up continuous deployment (CD), you can use GitHub Actions to automatically deploy code updates to the server whenever you push changes to the `main` branch.

### 9.1 Set Up Secrets on GitHub
1. Go to your repository on GitHub.
2. Navigate to **Settings** -> **Secrets and variables** -> **Actions**.
3. Add the following repository secrets by clicking **New repository secret**:
   * `SSH_HOST`: Your server IP address (e.g. `203.0.113.5`).
   * `SSH_USER`: The SSH username (e.g. `ubuntu`).
   * `SSH_KEY`: The server's SSH Private Key (contents of your `.pem` key file). Make sure it includes the `BEGIN` and `END` lines.

### 9.2 Create the Workflow Configuration
On your local computer, create a new directory and configuration file at `.github/workflows/deploy.yml` in the root of your project:

```bash
mkdir -p .github/workflows
nano .github/workflows/deploy.yml
```

Paste the following YAML configuration inside the file:
```yaml
name: Deploy Next.js Frontend

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
            # Navigate to project directory
            cd /home/ubuntu/frontend/projectname

            # Pull latest changes from the repository
            git pull origin main

            # Install dependencies
            npm install

            # Build production Next.js files
            npm run build

            # Restart the application with PM2 using config
            pm2 restart ecosystem.config.js
```
*(Note: Replace `projectname` with your actual project directory name).*

### 9.3 Commit and Push
Commit the new workflow file and push it:
```bash
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Actions CD deployment workflow"
git push origin main
```
Your frontend application is now fully automated! Whenever you push code, GitHub Actions will trigger, connect to the server, build, and restart PM2 automatically.


