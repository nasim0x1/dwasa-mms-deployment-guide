# Dhaka Water Supply and Sewerage Authority (DWASA)  
# Meter Management System (MMS) — Deployment Guide

**Target:** Oracle Linux 8.8 (or RHEL 8 / Rocky Linux 8)  
**Prepared for:** Syntech Solution Limited

---

## Table of Contents

1. [Overview](#1-overview)
2. [Step 1 — Install all tools](#2-step-1--install-all-tools)
3. [Step 2 — Apply configuration](#3-step-2--apply-configuration)
4. [Verification & troubleshooting](#4-verification--troubleshooting)

---

## 1. Overview

| Component        | Technology        | Port | Path / Service        |
|-----------------|-------------------|------|------------------------|
| Frontend        | Vue.js (SPA)      | 80   | `/var/www/wasa-app`    |
| Backend API     | Spring Boot JAR   | 9090 | systemd `wasa-mms`      |
| Database        | Oracle 19c        | 1521 | (existing/remote)       |
| Reverse proxy   | Nginx             | 80   | `nginx`                 |

**Assumptions:** Oracle Linux 8.8 with root/sudo; Oracle 19c reachable; built artifacts (`wasa-backend.jar`, `application.properties`, Vue `dist/`) ready.

---

## 2. Step 1 — Install all tools

Do these in order on the **server**. Install every required tool and open firewall ports before configuring the application.

### 2.1 Update system

```bash
sudo dnf update -y
```

### 2.2 Install Java 17

```bash
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Verify
java -version
# Should show: openjdk version "17.x.x"
```

### 2.3 Install Nginx

```bash
sudo dnf install -y nginx
sudo systemctl enable nginx
```

### 2.4 Create application user (for running the JAR)

```bash
sudo useradd -r -s /bin/false springbootapp
```

### 2.5 Create directory structure

```bash
# Backend
sudo mkdir -p /opt/wasa-mms
sudo mkdir -p /opt/wasa-mms/uploads

# Frontend
sudo mkdir -p /var/www/wasa-app

# Ownership for backend
sudo chown -R springbootapp:springbootapp /opt/wasa-mms
```

### 2.6 Open firewall ports

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

*(If the backend is only reached via Nginx on the same host, you can skip opening 9090 and only allow 80.)*

---

## 3. Step 2 — Apply configuration

After all tools are installed, apply configuration and deploy the application.

### 3.1 Oracle Database (create user, permissions, quota, then export/import)

*Assume Oracle 19c is already installed and the pluggable database exists. Run as the Oracle OS user (e.g. `oracle`) or with SYSDBA.*

#### 1. Create OS directory for dump files

On the database server:

```bash
sudo mkdir -p /home/oracle/exp_dir
sudo chown oracle:oinstall /home/oracle/exp_dir
```

#### 2. Create new DB user (schema)

```bash
sqlplus / as sysdba
```

In SQL*Plus:

```sql
-- Switch to pluggable database (e.g. WASADB)
ALTER SESSION SET CONTAINER = WASADB;

-- Create new user (schema)
CREATE USER WASA_MMS_LIVE IDENTIFIED BY WasaMeetingX;
```

#### 3. Grant import/export permissions

Still in SQL*Plus:

```sql
-- Allow Data Pump import (needed to load from DMP)
GRANT IMP_FULL_DATABASE TO WASA_MMS_LIVE;

-- Allow Data Pump export (if this user will export; optional)
GRANT EXP_FULL_DATABASE TO WASA_MMS_LIVE;

-- Connect and resource for normal use
GRANT CONNECT, RESOURCE TO WASA_MMS_LIVE;
```

#### 4. Set quota (tablespace space)

```sql
ALTER USER WASA_MMS_LIVE QUOTA UNLIMITED ON USERS;
```

#### 5. Create directory object and grant read/write for Data Pump

```sql
CREATE OR REPLACE DIRECTORY exp_dir AS '/home/oracle/exp_dir';
GRANT READ, WRITE ON DIRECTORY exp_dir TO WASA_MMS_LIVE;
```

Use **Host**, **Port** (1521), **Service name** (e.g. `WASADB`), **Username** (`WASA_MMS_LIVE`), **Password** (`WasaMeetingX`) in `application.properties`.

---

#### Export command (expdp)

Run on the **source** server. Place the DMP in the path that `exp_dir` points to (e.g. `/home/oracle/exp_dir/`). Replace `SPRING_USER2` and password with the source schema.

```bash
expdp SPRING_USER2/<source_password>@//localhost:1521/WASADB \
  DIRECTORY=exp_dir \
  DUMPFILE=WASA_MMS_MASTER_FINAL.DMP \
  LOGFILE=export_log.log \
  SCHEMAS=SPRING_USER2
```

Full database export (privileged user):

```bash
expdp system/<password>@//localhost:1521/WASADB \
  DIRECTORY=exp_dir \
  DUMPFILE=full_backup.DMP \
  LOGFILE=export_log.log \
  FULL=Y
```

Copy the DMP to the target server under the same path used by `exp_dir` (e.g. `/home/oracle/exp_dir/`).

#### Import command (impdp)

Place the DMP in the directory that `exp_dir` points to (e.g. `/home/oracle/exp_dir/WASA_MMS_MASTER_FINAL.DMP`). Then run:

```bash
impdp WASA_MMS_LIVE/WasaMeetingX@//localhost:1521/WASADB \
  DIRECTORY=exp_dir \
  DUMPFILE=WASA_MMS_MASTER_FINAL.DMP \
  LOGFILE=import_log.log \
  REMAP_SCHEMA=SPRING_USER2:WASA_MMS_LIVE \
  TABLE_EXISTS_ACTION=REPLACE
```

- **REMAP_SCHEMA=source:target** — objects from dump schema go into the user you created.
- **TABLE_EXISTS_ACTION=REPLACE** — replace existing tables (or use `SKIP` / `APPEND`).

For remote DB, use `//HOST:PORT/SERVICE_NAME` instead of `//localhost:1521/WASADB`.

---

### 3.2 Backend (Spring Boot)

**Build (on your build machine):** From backend project root (Java 17 + Maven Wrapper):

```bash
./mvnw clean package -DskipTests          # Linux/macOS
.\mvnw.cmd clean package -DskipTests      # Windows
# Output: target/wasa-backend.jar
```

**Copy to server:**

```bash
scp target/wasa-backend.jar USER@SERVER:/opt/wasa-mms/
scp src/main/resources/application.properties USER@SERVER:/opt/wasa-mms/
```

**Configure `application.properties` on server:**

```bash
sudo nano /opt/wasa-mms/application.properties
```

Set at least:

- `spring.datasource.url=jdbc:oracle:thin:@//HOST:PORT/SERVICE_NAME`
- `spring.datasource.username=your_db_user`
- `spring.datasource.password=your_db_password`
- `server.port=9090` (optional)
- `file.upload-dir=/opt/wasa-mms/uploads` (if used)

Then:

```bash
sudo chown springbootapp:springbootapp /opt/wasa-mms/application.properties
sudo chmod 600 /opt/wasa-mms/application.properties
```

**Create and start systemd service:**

```bash
sudo nano /etc/systemd/system/wasa-mms.service
```

Paste:

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

```bash
sudo systemctl daemon-reload
sudo systemctl enable wasa-mms
sudo systemctl start wasa-mms
sudo systemctl status wasa-mms
```

---

### 3.3 Frontend (Vue.js)

**Build (on build machine):** Set API URL in `wasa-app/src/config/api_config.js`, then:

```bash
cd /path/to/wasa-app
npm install
npm run build
```

**Deploy to server:** Copy contents of `dist/` to `/var/www/wasa-app/`:

```bash
rsync -avz dist/ USER@SERVER:/var/www/wasa-app/
# or: scp -r dist/* USER@SERVER:/var/www/wasa-app/
```

**Permissions and SELinux on server:**

```bash
sudo chown -R nginx:nginx /var/www/wasa-app/
sudo find /var/www/wasa-app/ -type d -exec chmod 755 {} \;
sudo find /var/www/wasa-app/ -type f -exec chmod 644 {} \;
sudo chcon -Rt httpd_sys_content_t /var/www/wasa-app/
```

---

### 3.4 Nginx configuration

Create the site config:

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

**Optional — proxy API (same origin):** Add inside the same `server` block:

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

**Test and start Nginx:**

```bash
sudo nginx -t
sudo systemctl start nginx
# or if already running: sudo systemctl restart nginx
```

---

### 3.5 SELinux (if enabled)

```bash
# Frontend (usually done above)
sudo chcon -Rt httpd_sys_content_t /var/www/wasa-app/

# Uploads (if Nginx serves /uploads/)
sudo chcon -Rt httpd_sys_content_t /opt/wasa-mms/uploads/

# If Nginx proxy to backend fails
sudo setsebool -P httpd_can_network_connect 1
```

---

## 4. Verification & troubleshooting

**Backend:**

```bash
sudo systemctl status wasa-mms
sudo journalctl -u wasa-mms -f
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:9090/api/
```

**Nginx & frontend:**

```bash
sudo systemctl status nginx
sudo nginx -t
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/
```

**Browser:** Open `http://192.168.115.51` or `http://meter.dhakawasa.org` and check login and API calls.

| Issue | Check |
|-------|--------|
| 502 Bad Gateway | Backend not running; `journalctl -u wasa-mms` |
| 403 on static files | `chown nginx:nginx`, `chcon httpd_sys_content_t` |
| API refused | Firewall, backend on 9090, correct URL in `api_config.js` |
| Blank page | Nginx `try_files ... /index.html`, correct `root` |
| Uploads not found | `location /uploads/` alias, SELinux on `/opt/wasa-mms/uploads/` |

---

## Quick reference

```bash
# Backend
sudo systemctl restart wasa-mms
sudo journalctl -u wasa-mms -f

# Frontend after deploy
sudo chown -R nginx:nginx /var/www/wasa-app/
sudo find /var/www/wasa-app/ -type d -exec chmod 755 {} \;
sudo find /var/www/wasa-app/ -type f -exec chmod 644 {} \;
sudo chcon -Rt httpd_sys_content_t /var/www/wasa-app/
sudo systemctl restart nginx
```

---

**Document version:** 1.0 · **Target:** Oracle Linux 8.8 · **Last updated:** February 2026
