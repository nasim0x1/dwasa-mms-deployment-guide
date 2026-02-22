# Dhaka Water Supply and Sewerage Authority (DWASA)  
# Meter Management System (MMS) — Deployment Guide

**Target:** Oracle Linux 8.8 (or RHEL 8 / Rocky Linux 8)  
**Prepared for:** Syntech Solution Limited

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Server Preparation](#3-server-preparation)
4. [Oracle Database Setup](#4-oracle-database-setup)
5. [Backend (Spring Boot) Deployment](#5-backend-spring-boot-deployment)
6. [Frontend (Vue.js) Build & Deployment](#6-frontend-vuejs-build--deployment)
7. [Nginx Configuration](#7-nginx-configuration)
8. [Firewall & SELinux](#8-firewall--selinux)
9. [Verification & Troubleshooting](#9-verification--troubleshooting)

---

## 1. Overview

| Component        | Technology        | Port | Path / Service        |
|-----------------|-------------------|------|------------------------|
| Frontend        | Vue.js (SPA)      | 80   | `/var/www/wasa-app`    |
| Backend API     | Spring Boot JAR   | 9090 | systemd `wasa-mms`      |
| Database        | Oracle 19c        | 1521 | (existing/remote)       |
| Reverse proxy   | Nginx             | 80   | `nginx`                 |

**Assumptions:**

- Oracle Linux 8.8 is freshly installed with root/sudo access.
- Oracle Database 19c is either on the same server or reachable (host, port, service name, user, password known).
- You have the built artifacts: `wasa-backend.jar`, `application.properties`, and the Vue production build (e.g. `wasa-app/dist/` or similar).

---

## 2. Prerequisites

- **OS:** Oracle Linux 8.8 (or compatible RHEL 8.x)
- **Java:** OpenJDK 17 or Oracle JDK 17 (for running the JAR only on server)
- **Node.js:** v14.19.3+ (only needed on build machine for `npm run build`)
- **Maven:** Not required if the project has Maven Wrapper (`mvnw` / `mvnw.cmd`); use `./mvnw` (Linux/macOS) or `.\mvnw.cmd` (Windows) to build.
- **Oracle Database:** 19c installed and running; schema created for the application
- **Network:** Server IP/hostname and DNS (e.g. `meter.dhakawasa.org`) pointing to this server

---

## 3. Server Preparation

### 3.1 Update system

```bash
sudo dnf update -y
```

### 3.2 Create application user (for running the JAR)

```bash
sudo useradd -r -s /bin/false springbootapp
```

### 3.3 Install Java 17

```bash
# Oracle Linux / RHEL 8
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Verify
java -version
# Should show: openjdk version "17.x.x"
```

### 3.4 Create directory structure

```bash
# Backend
sudo mkdir -p /opt/wasa-mms
sudo mkdir -p /opt/wasa-mms/uploads

# Frontend
sudo mkdir -p /var/www/wasa-app

# Ownership for backend (application user)
sudo chown -R springbootapp:springbootapp /opt/wasa-mms
```

---

## 4. Oracle Database Setup

- Ensure Oracle 19c is installed and the listener is running.
- Create a dedicated schema/user for MMS and note:
  - **Host**
  - **Port** (default 1521)
  - **Service name** (e.g. `ORCL`, `XEPDB1`)
  - **Username**
  - **Password**

Use these values in `application.properties` (see next section). No extra steps on the app server are needed if the DB is remote; only network connectivity and firewall (port 1521 if DB is on another host) are required.

---

## 5. Backend (Spring Boot) Deployment

### 5.0 Build the backend (on build machine)

From the backend project root (where `pom.xml` and `mvnw` are). **Java 17** is required. The project uses **Maven Wrapper** (`mvnw`), so you do not need to install Maven.

**Linux / macOS:**

```bash
# Navigate to backend project root
cd /path/to/wasa-backend

# Build JAR (skip tests for faster build; remove -DskipTests to run tests)
./mvnw clean package -DskipTests

# Output JAR location
# target/wasa-backend.jar
```

**Windows (PowerShell or CMD):**

```bash
cd \path\to\wasa-backend
.\mvnw.cmd clean package -DskipTests
```

To run tests during build:

```bash
./mvnw clean package          # Linux/macOS
.\mvnw.cmd clean package      # Windows
```

The built JAR will be at **`target/wasa-backend.jar`**. Copy `src/main/resources/application.properties` to the server as well (or create it on the server).

### 5.1 Copy backend files to server

From your build machine, copy to the server (replace `USER` and `SERVER`):

```bash
scp target/wasa-backend.jar USER@SERVER:/opt/wasa-mms/
scp src/main/resources/application.properties USER@SERVER:/opt/wasa-mms/
```

Or use rsync/sftp as per your policy.

### 5.2 Configure application.properties on server

Edit on the server:

```bash
sudo nano /opt/wasa-mms/application.properties
```

Set at least:

- **Oracle URL:**  
  `spring.datasource.url=jdbc:oracle:thin:@//HOST:PORT/SERVICE_NAME`  
  Example: `jdbc:oracle:thin:@//192.168.115.50:1521/ORCL`
- **Username:** `spring.datasource.username=your_db_user`
- **Password:** `spring.datasource.password=your_db_password`
- **Server port (optional):** `server.port=9090`
- **Upload path (if used):** e.g. `file.upload-dir=/opt/wasa-mms/uploads`

Save and secure the file:

```bash
sudo chown springbootapp:springbootapp /opt/wasa-mms/application.properties
sudo chmod 600 /opt/wasa-mms/application.properties
```

### 5.3 Create systemd service

```bash
sudo nano /etc/systemd/system/wasa-mms.service
```

Paste (adjust paths if different):

```ini
[Unit]
Description=WASA Meter Management System - Spring Boot Backend
After=syslog.target network.target

[Service]
User=springbootapp
ExecStart=/usr/bin/java -jar /opt/wasa-mms/wasa-backend.jar --spring.config.location=/opt/wasa-mms/application.properties
SuccessExitStatus=143
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=wasa-mms

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wasa-mms
sudo systemctl start wasa-mms
sudo systemctl status wasa-mms
```

Check logs:

```bash
sudo journalctl -u wasa-mms -f
```

---

## 6. Frontend (Vue.js) Build & Deployment

### 6.0 Build the frontend (on build machine)

On a machine with **Node.js v14.19.3+** installed:

```bash
# Navigate to Vue project directory
cd /path/to/wasa-app

# Install dependencies
npm install

# Production build (output goes to dist/ by default)
npm run build
```

**Output:** Production files are in **`dist/`** (or the `outputDir` set in `vue.config.js`). Deploy the *contents* of this folder to the server.

**Before building:** Set the production API URL in `wasa-app/src/config/api_config.js`, e.g.:

```javascript
export const authServiceBaseUrl = 'http://192.168.115.51:9090/api/'
// Or, if using Nginx proxy: export const authServiceBaseUrl = '/api/'
```

### 6.1 Deploy build to server

Copy the **contents** of the build output (usually `dist/`) to the server:

   ```bash
   rsync -avz dist/ USER@SERVER:/var/www/wasa-app/
   # or
   scp -r dist/* USER@SERVER:/var/www/wasa-app/
   ```

### 6.2 Set ownership and permissions on server

```bash
sudo chown -R nginx:nginx /var/www/wasa-app/
sudo find /var/www/wasa-app/ -type d -exec chmod 755 {} \;
sudo find /var/www/wasa-app/ -type f -exec chmod 644 {} \;
```

If SELinux is enabled (default on Oracle Linux 8):

```bash
sudo chcon -Rt httpd_sys_content_t /var/www/wasa-app/
```

Restart Nginx after any change:

```bash
sudo systemctl restart nginx
```

---

## 7. Nginx Configuration

### 7.1 Install Nginx

```bash
sudo dnf install -y nginx
sudo systemctl enable nginx
```

### 7.2 Create site configuration

```bash
sudo nano /etc/nginx/conf.d/wasa.conf
```

Paste (adjust `server_name` if needed):

```nginx
server {
    listen 80;
    server_name 192.168.115.51 wasa.local meter.dhakawasa.org 123.49.33.97;

    root /var/www/wasa-app;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /uploads/ {
        alias /opt/wasa-mms/uploads/;
        autoindex off;
    }
}
```

**Optional — proxy API to backend (same origin, no CORS):**

If you prefer the frontend to call `/api/` on the same host instead of `http://IP:9090/api/`, add inside the same `server` block:

```nginx
    location /api/ {
        proxy_pass http://127.0.0.1:9090/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
```

Then in `api_config.js` use: `export const authServiceBaseUrl = '/api/'`

### 7.3 Test and reload Nginx

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## 8. Firewall & SELinux

### 8.1 Firewall (firewalld)

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

If the backend is only used via Nginx proxy on the same host, you can omit opening 9090 publicly and only allow 80.

### 8.2 SELinux

- Frontend directory must be readable by Nginx (already done above):

  ```bash
  sudo chcon -Rt httpd_sys_content_t /var/www/wasa-app/
  ```

- If Nginx serves files from `/opt/wasa-mms/uploads/`, allow read access:

  ```bash
  sudo chcon -Rt httpd_sys_content_t /opt/wasa-mms/uploads/
  ```

- If you use `proxy_pass` to 9090 and see permission errors, check:

  ```bash
  sudo setsebool -P httpd_can_network_connect 1
  ```

---

## 9. Verification & Troubleshooting

### 9.1 Backend

```bash
# Service status
sudo systemctl status wasa-mms

# Logs
sudo journalctl -u wasa-mms -n 100 --no-pager

# Test API (from server)
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:9090/api/
# Expect 200 or 401, not connection refused
```

### 9.2 Frontend & Nginx

```bash
# Nginx status
sudo systemctl status nginx

# Test config
sudo nginx -t

# Test from server
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/
# Expect 200
```

### 9.3 From browser

- Open `http://192.168.115.51` or `http://meter.dhakawasa.org`.
- Confirm login and that API calls work (Network tab: requests to `/api/` or `:9090/api/` succeed).

### 9.4 Common issues

| Issue | Check |
|-------|--------|
| 502 Bad Gateway | Backend not running or wrong `proxy_pass` port; `journalctl -u wasa-mms` |
| 403 Forbidden on static files | Permissions and SELinux: `chown nginx:nginx`, `chcon httpd_sys_content_t` |
| API connection refused | Firewall, backend listening on 9090, correct URL in `api_config.js` |
| Blank page / wrong route | Nginx `try_files ... /index.html` and correct `root` |
| Uploads not found | `location /uploads/` alias path and SELinux on `/opt/wasa-mms/uploads/` |

---

## Quick reference — commands summary

```bash
# Backend
sudo systemctl start wasa-mms
sudo systemctl stop wasa-mms
sudo systemctl restart wasa-mms
sudo journalctl -u wasa-mms -f

# Frontend permissions after deploy
sudo chown -R nginx:nginx /var/www/wasa-app/
sudo find /var/www/wasa-app/ -type d -exec chmod 755 {} \;
sudo find /var/www/wasa-app/ -type f -exec chmod 644 {} \;
sudo chcon -Rt httpd_sys_content_t /var/www/wasa-app/
sudo systemctl restart nginx

# Firewall
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
```

---

**Document version:** 1.0  
**Target:** Oracle Linux 8.8  
**Last updated:** February 2025
