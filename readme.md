# Production Server Setup Guide

> **Next.js + Express.js + MySQL on Ubuntu — Complete Deployment Handbook**

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Detailed Port Architecture](#2-detailed-port-architecture)
3. [Recommended Production Server Configuration](#3-recommended-production-server-configuration)
4. [Ubuntu Server Initial Setup](#4-ubuntu-server-initial-setup)
5. [SSH Security Configuration](#5-ssh-security-configuration)
6. [Firewall Configuration (UFW)](#6-firewall-configuration-ufw)
7. [Installing Node.js Runtime](#7-installing-nodejs-runtime)
8. [Installing MySQL Database](#8-installing-mysql-database)
9. [MySQL Database Configuration](#9-mysql-database-configuration)
10. [Application Environment Configuration](#10-application-environment-configuration)
11. [Application Deployment](#11-application-deployment)
12. [Process Management using PM2](#12-process-management-using-pm2)
13. [Nginx Reverse Proxy Setup](#13-nginx-reverse-proxy-setup)
14. [Nginx Configuration](#14-nginx-configuration)
15. [SSL Certificate Setup (HTTPS)](#15-ssl-certificate-setup-https)
16. [Performance Optimization](#16-performance-optimization)
17. [Monitoring and Logs](#17-monitoring-and-logs)
18. [Backup Strategy](#18-backup-strategy)
19. [Security Best Practices](#19-security-best-practices)
20. [Final Production Architecture](#20-final-production-architecture)
21. [Docker Deployment (Optional)](#21-docker-deployment-optional)
22. [CI/CD Pipeline Setup](#22-cicd-pipeline-setup)
23. [Health Check Endpoint](#23-health-check-endpoint)
24. [Rate Limiting & DDoS Protection](#24-rate-limiting--ddos-protection)
25. [Log Rotation](#25-log-rotation)
26. [Zero-Downtime Deployment](#26-zero-downtime-deployment)
27. [Environment-Specific Configurations](#27-environment-specific-configurations)
28. [Troubleshooting Guide](#28-troubleshooting-guide)

---

## 1. System Architecture Overview

The application uses a **three-tier architecture**, separating the frontend, backend logic, and database to ensure scalability, maintainability, and security.

### Architecture Layers

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Client Layer** | Browser | User interface access |
| **Reverse Proxy Layer** | Nginx | Handles requests, SSL termination |
| **Application Layer** | Node.js (Next.js + Express.js) | Business logic & API |
| **Data Layer** | MySQL | Persistent data storage |

### Request Flow

```
User Browser
     │
     │  HTTPS (443)
     ▼
Nginx Reverse Proxy
     │
     │  Proxy Pass
     ▼
Node.js Application Server
  Next.js + Express.js
       Port 3000
     │
     │  SQL Queries
     ▼
MySQL Database
   Port 3306
```

---

## 2. Detailed Port Architecture

| Service | Port | Description |
|---------|------|-------------|
| SSH | `22` | Secure remote server access |
| HTTP | `80` | Default web traffic (redirects to 443) |
| HTTPS | `443` | Secure web traffic |
| Node.js Application | `3000` | Application runtime |
| MySQL Database | `3306` | Database service |
| Nginx Internal | Reverse Proxy | Handles traffic routing |

### Port Flow

```
Public Internet
     │
     ▼
Port 443 (HTTPS)
     │
     ▼
Nginx Reverse Proxy
     │
     ▼
localhost:3000
(Node.js Application)
     │
     ▼
localhost:3306
(MySQL Database)
```

> ⚠️ **Security Note:** Only ports **80** and **443** are exposed publicly. All other ports remain internal.

---

## 3. Recommended Production Server Configuration

### Standard Configuration

| Component | Recommended |
|-----------|-------------|
| OS | Ubuntu 22.04 LTS |
| CPU | 2 vCPU or higher |
| RAM | 4–8 GB |
| Storage | 40 GB SSD |
| Node.js | v18+ |
| MySQL | v8+ |
| Nginx | Latest stable |

### High-Traffic Configuration

| Component | Recommended |
|-----------|-------------|
| CPU | 4 vCPU |
| RAM | 8–16 GB |
| Storage | 80 GB SSD |
| Load Balancer | Yes |
| CDN | Recommended (Cloudflare / AWS CloudFront) |

---

## 4. Ubuntu Server Initial Setup

**Update system packages:**

```bash
sudo apt update && sudo apt upgrade -y
```

**Install essential tools:**

```bash
sudo apt install curl git unzip build-essential ufw htop net-tools -y
```

**Create a non-root deploy user:**

```bash
adduser deploy
usermod -aG sudo deploy
```

**Switch to the deploy user:**

```bash
su - deploy
```

**Set the timezone:**

```bash
sudo timedatectl set-timezone Asia/Kolkata
```

---

## 5. SSH Security Configuration

**Edit SSH configuration:**

```bash
sudo nano /etc/ssh/sshd_config
```

**Recommended settings:**

```
PermitRootLogin no
PasswordAuthentication yes
Port 22
MaxAuthTries 5
LoginGraceTime 60
ClientAliveInterval 120
ClientAliveCountMax 3
```

**Restart SSH:**

```bash
sudo systemctl restart ssh
```

> **Tip:** For stronger security, consider using SSH key-based authentication and disabling `PasswordAuthentication`.

### Setting Up SSH Key Authentication (Recommended)

```bash
# On your LOCAL machine, generate a key pair
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy the public key to the server
ssh-copy-id deploy@your_server_ip

# Then disable password auth on the server
# PasswordAuthentication no
```

---

## 6. Firewall Configuration (UFW)

**Enable firewall:**

```bash
sudo ufw enable
```

**Allow required ports:**

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

**Check firewall rules:**

```bash
sudo ufw status verbose
```

**Expected output:**

```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

---

## 7. Installing Node.js Runtime

**Install Node.js using NodeSource repository:**

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y
```

**Verify installation:**

```bash
node -v   # Expected: v18.x
npm -v    # Expected: 9.x
```

**Install Yarn (optional but recommended):**

```bash
sudo npm install -g yarn
```

---

## 8. Installing MySQL Database

**Install MySQL:**

```bash
sudo apt install mysql-server -y
```

**Secure installation:**

```bash
sudo mysql_secure_installation
```

**Recommended responses during secure installation:**

| Prompt | Answer |
|--------|--------|
| Validate password component | Yes |
| Password strength | Medium or Strong |
| Remove anonymous users | Yes |
| Disallow root remote login | Yes |
| Remove test database | Yes |
| Reload privilege tables | Yes |

---

## 9. MySQL Database Configuration

**Login to MySQL:**

```bash
sudo mysql
```

**Create production database:**

```sql
CREATE DATABASE production_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Create database user:**

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

**Verify connection:**

```bash
mysql -u app_user -p production_db -e "SELECT 1;"
```

---

## 10. Application Environment Configuration

**Create `.env` file in your project root:**

```env
# Server
NODE_ENV=production
PORT=3000

# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=app_user
DB_PASSWORD=StrongPassword123!
DB_NAME=production_db

# Session / Auth (if applicable)
SESSION_SECRET=your_super_secret_session_key
JWT_SECRET=your_jwt_secret_key

# Domain
NEXT_PUBLIC_BASE_URL=https://yourdomain.com
```

**Example MySQL connection in Node.js:**

```javascript
const mysql = require('mysql2');

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
});

module.exports = pool.promise();
```

> ⚠️ **Never commit `.env` files to Git.** Add `.env` to your `.gitignore`.

---

## 11. Application Deployment

**Clone repository:**

```bash
git clone https://github.com/yourrepo/project.git
cd project
```

**Install dependencies:**

```bash
npm install --production
```

**Build Next.js application:**

```bash
npm run build
```

**Start application:**

```bash
npm run start
```

**Application runs on:**

```
http://localhost:3000
```

---

## 12. Process Management using PM2

PM2 ensures that the Node.js application:

- Runs continuously in the background
- Restarts automatically on crash
- Starts automatically on server reboot
- Provides built-in monitoring

**Install PM2:**

```bash
sudo npm install -g pm2
```

**Start application:**

```bash
pm2 start npm --name "webapp" -- start
```

**Cluster mode (recommended for multi-core servers):**

```bash
pm2 start npm --name "webapp" -i max -- start
```

**Save process list:**

```bash
pm2 save
```

**Enable auto start on reboot:**

```bash
pm2 startup
```

**Other useful PM2 commands:**

```bash
pm2 list              # List all processes
pm2 restart webapp    # Restart the app
pm2 stop webapp       # Stop the app
pm2 delete webapp     # Remove from PM2
pm2 logs webapp       # View logs
pm2 monit             # Real-time monitoring dashboard
```

---

## 13. Nginx Reverse Proxy Setup

**Install Nginx:**

```bash
sudo apt install nginx -y
```

**Start and enable the service:**

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

**Verify Nginx is running:**

```bash
sudo systemctl status nginx
```

---

## 14. Nginx Configuration

**Create configuration file:**

```bash
sudo nano /etc/nginx/sites-available/webapp
```

**Configuration:**

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    # SSL certificates (managed by Certbot)
    # ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Static file caching
    location /_next/static/ {
        proxy_pass http://localhost:3000;
        expires 365d;
        access_log off;
        add_header Cache-Control "public, immutable";
    }

    # Favicon & robots
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
}
```

**Enable site:**

```bash
sudo ln -s /etc/nginx/sites-available/webapp /etc/nginx/sites-enabled/
```

**Remove default site (optional):**

```bash
sudo rm /etc/nginx/sites-enabled/default
```

**Test configuration:**

```bash
sudo nginx -t
```

**Restart Nginx:**

```bash
sudo systemctl restart nginx
```

---

## 15. SSL Certificate Setup (HTTPS)

**Install Certbot:**

```bash
sudo apt install certbot python3-certbot-nginx -y
```

**Generate certificate:**

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

**Auto-renewal test:**

```bash
sudo certbot renew --dry-run
```

**Traffic flow after SSL:**

```
User Browser
     │
  HTTPS 443
     │
Nginx Reverse Proxy
  (SSL Termination)
     │
  HTTP (internal)
     │
localhost:3000
  Node.js App
```

> **Certbot automatically sets up a cron job** for certificate renewal (every 90 days).

---

## 16. Performance Optimization

### Enable Gzip Compression

Edit `/etc/nginx/nginx.conf`:

```nginx
http {
    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;
    gzip_min_length 256;
}
```

### Increase Worker Connections

```nginx
worker_processes auto;

events {
    worker_connections 2048;
    multi_accept on;
    use epoll;
}
```

### Client Body & Timeout Settings

```nginx
http {
    client_max_body_size 10M;
    client_body_timeout 12;
    client_header_timeout 12;
    keepalive_timeout 15;
    send_timeout 10;
}
```

### Node.js Memory Optimization

```bash
# Set memory limit for Node.js (adjust based on server RAM)
pm2 start npm --name "webapp" --node-args="--max-old-space-size=2048" -- start
```

---

## 17. Monitoring and Logs

### Nginx Logs

```bash
# Access log
tail -f /var/log/nginx/access.log

# Error log
tail -f /var/log/nginx/error.log
```

### PM2 Monitoring

```bash
pm2 logs             # Stream all logs
pm2 logs webapp      # Stream app-specific logs
pm2 monit            # Real-time monitoring dashboard
pm2 status           # Quick status overview
```

### System Monitoring

```bash
htop                 # Interactive process viewer
df -h                # Disk usage
free -m              # Memory usage
uptime               # Server uptime and load
```

### Install Monitoring Tools (Optional)

```bash
# Install Netdata for real-time server monitoring
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
# Access at: http://your_server_ip:19999
```

---

## 18. Backup Strategy

### Manual Database Backup

```bash
mysqldump -u app_user -p production_db > /home/deploy/backups/db_backup_$(date +%Y%m%d).sql
```

### Create Backup Directory

```bash
mkdir -p /home/deploy/backups
```

### Automate Backups using Cron

```bash
crontab -e
```

**Add daily backup at 2:00 AM:**

```cron
0 2 * * * mysqldump -u app_user -pStrongPassword123! production_db > /home/deploy/backups/db_backup_$(date +\%Y\%m\%d).sql 2>/dev/null
```

### Backup Rotation (keep last 7 days)

```bash
# Add this to cron as well
0 3 * * * find /home/deploy/backups/ -name "db_backup_*.sql" -mtime +7 -delete
```

### Application Code Backup

```bash
# Backup the entire project
tar -czf /home/deploy/backups/app_backup_$(date +%Y%m%d).tar.gz /home/deploy/project/
```

---

## 19. Security Best Practices

### Checklist

- [x] Use HTTPS only (redirect HTTP → HTTPS)
- [x] Store secrets in environment variables
- [x] Disable root login via SSH
- [x] Enable UFW firewall
- [x] Use strong database passwords
- [x] Restrict database access to `localhost`
- [ ] Enable Fail2Ban
- [ ] Set up automatic security updates
- [ ] Configure Content Security Policy (CSP)

### Install Fail2Ban

```bash
sudo apt install fail2ban -y
```

**Configure Fail2Ban for SSH:**

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 3600
findtime = 600

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log
maxretry = 5
bantime = 3600
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
```

### Automatic Security Updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

---

## 20. Final Production Architecture

```
                    INTERNET
                       │
                       ▼
               Port 443 (HTTPS)
                       │
                       ▼
              ┌─────────────────┐
              │      NGINX      │
              │  Reverse Proxy  │
              │  SSL + Gzip +   │
              │  Security Hdrs  │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   Node.js App   │
              │  Next.js +      │
              │  Express.js     │
              │  Port 3000      │
              │  (PM2 Managed)  │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  MySQL Database │
              │  Port 3306      │
              │  (localhost)    │
              └─────────────────┘
```

---

## 21. Docker Deployment (Optional)

For containerized deployments, use Docker and Docker Compose.

### Install Docker

```bash
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker deploy
```

### `Dockerfile`

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      - db
    restart: always

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: production_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: StrongPassword123!
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    restart: always

volumes:
  mysql_data:
```

**Run with Docker Compose:**

```bash
docker-compose up -d --build
```

---

## 22. CI/CD Pipeline Setup

### GitHub Actions Example

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build

      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/project
            git pull origin main
            npm install --production
            npm run build
            pm2 restart webapp
```

---

## 23. Health Check Endpoint

Add a health check route to your Express.js application:

```javascript
// routes/health.js
const express = require('express');
const router = express.Router();
const pool = require('../config/database');

router.get('/api/health', async (req, res) => {
  try {
    // Check database connection
    await pool.query('SELECT 1');

    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      memory: process.memoryUsage(),
      database: 'connected',
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      database: 'disconnected',
      error: error.message,
    });
  }
});

module.exports = router;
```

**Monitor with cron:**

```bash
# Check health every 5 minutes
*/5 * * * * curl -s http://localhost:3000/api/health | jq . >> /home/deploy/logs/health.log
```

---

## 24. Rate Limiting & DDoS Protection

### Nginx Rate Limiting

Add to your Nginx configuration:

```nginx
http {
    # Define rate limit zone
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

    server {
        # Apply to API routes
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://localhost:3000;
        }

        # Stricter limit on auth routes
        location /api/auth/ {
            limit_req zone=login burst=5 nodelay;
            proxy_pass http://localhost:3000;
        }
    }
}
```

### Node.js Rate Limiting (Express)

```bash
npm install express-rate-limit
```

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per window
  message: { error: 'Too many requests, please try again later.' },
});

app.use('/api/', limiter);
```

---

## 25. Log Rotation

### Configure Logrotate for Nginx

```bash
sudo nano /etc/logrotate.d/nginx
```

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

### PM2 Log Rotation

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
```

---

## 26. Zero-Downtime Deployment

### Using PM2 Reload

```bash
# Graceful reload (zero downtime)
pm2 reload webapp

# vs restart (has brief downtime)
pm2 restart webapp
```

### Deployment Script

Create `deploy.sh`:

```bash
#!/bin/bash
set -e

echo "Starting deployment..."

cd /home/deploy/project

echo "Pulling latest code..."
git pull origin main

echo "Installing dependencies..."
npm install --production

echo "Building application..."
npm run build

echo "Reloading application (zero downtime)..."
pm2 reload webapp

echo "Deployment complete!"
echo "Current status:"
pm2 status
```

```bash
chmod +x deploy.sh
```

---

## 27. Environment-Specific Configurations

### Development

```env
NODE_ENV=development
PORT=3000
DB_HOST=localhost
DB_NAME=dev_db
```

### Staging

```env
NODE_ENV=staging
PORT=3000
DB_HOST=localhost
DB_NAME=staging_db
```

### Production

```env
NODE_ENV=production
PORT=3000
DB_HOST=localhost
DB_NAME=production_db
```

---

## 28. Troubleshooting Guide

### Common Issues & Solutions

| Issue | Command | Solution |
|-------|---------|----------|
| App not running | `pm2 list` | Restart: `pm2 restart webapp` |
| Nginx error | `sudo nginx -t` | Fix config, then `sudo systemctl restart nginx` |
| Port already in use | `sudo lsof -i :3000` | Kill process: `kill -9 <PID>` |
| MySQL won't connect | `sudo systemctl status mysql` | Restart: `sudo systemctl restart mysql` |
| SSL certificate expired | `sudo certbot certificates` | Renew: `sudo certbot renew` |
| High memory usage | `free -m && pm2 monit` | Restart app or increase swap |
| Disk full | `df -h` | Clean old backups/logs |

### Quick Debug Commands

```bash
# Check all running services
sudo systemctl list-units --type=service --state=running

# Check which process is using a port
sudo lsof -i :3000
sudo lsof -i :3306
sudo lsof -i :443

# View recent system errors
journalctl -xe --since "1 hour ago"

# Test database connection
mysql -u app_user -p -e "SELECT 1;" production_db

# Check Nginx config
sudo nginx -t

# View PM2 error logs
pm2 logs webapp --err --lines 50
```


---

> **Document Version:** 1.0  
> **Last Updated:** March 2026  
> **Author:** Santhosh Kumar R S
