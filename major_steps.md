# Production Server Setup Guide

> **Node.js + MySQL on Ubuntu — Quick Deployment Guide**

---

## 1. Update Ubuntu Server

```bash
sudo apt update && sudo apt upgrade -y
```

**Install required packages:**

```bash
sudo apt install curl git unzip build-essential -y
```

---

## 2. Install Node.js

**Install Node.js using NodeSource:**

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y
```

**Verify installation:**

```bash
node -v
npm -v
```

---

## 3. Install MySQL Database

```bash
sudo apt install mysql-server -y
```

**Secure MySQL:**

```bash
sudo mysql_secure_installation
```

---

## 4. Create Database

**Login to MySQL:**

```bash
sudo mysql
```

**Create database:**

```sql
CREATE DATABASE production_db;
```

**Create user:**

```sql
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
```

**Grant privileges:**

```sql
GRANT ALL PRIVILEGES ON production_db.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
```

**Exit MySQL:**

```sql
EXIT;
```

---

## 5. Deploy Application

**Clone your project:**

```bash
git clone https://github.com/yourrepo/project.git
cd project
```

**Install dependencies:**

```bash
npm install
```

**Build project:**

```bash
npm run build
```

**Start project:**

```bash
npm run start
```

**Application runs at:**

```
http://localhost:3000
```

---

## 6. Install PM2 (Process Manager)

**Install PM2 globally:**

```bash
sudo npm install -g pm2
```

**Start application with PM2:**

```bash
pm2 start npm --name "webapp" -- start
```

**Save PM2 processes:**

```bash
pm2 save
```

**Enable startup on reboot:**

```bash
pm2 startup
```

---

## 7. Install Nginx

```bash
sudo apt install nginx -y
```

**Start Nginx:**

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

## 8. Configure Nginx Reverse Proxy

**Create configuration file:**

```bash
sudo nano /etc/nginx/sites-available/webapp
```

**Add configuration:**

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }
}
```

**Enable site:**

```bash
sudo ln -s /etc/nginx/sites-available/webapp /etc/nginx/sites-enabled/
```

**Restart Nginx:**

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## 9. Install SSL (HTTPS)

**Install Certbot:**

```bash
sudo apt install certbot python3-certbot-nginx -y
```

**Generate SSL certificate:**

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

**Test renewal:**

```bash
sudo certbot renew --dry-run
```

---

## Final Architecture

```
User
  │
  ▼
HTTPS (443)
  │
  ▼
Nginx
  │
  ▼
Node.js App (PM2)
  │
  ▼
MySQL Database
```

---

> **Document Version:** 2.0
> **Last Updated:** March 2026
> **Author:** Server Admin Team
