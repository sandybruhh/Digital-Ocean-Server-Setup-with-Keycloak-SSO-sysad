# Keycloak Installation & Configuration

This document covers the complete installation and configuration of Keycloak on Rocky Linux 10 and securing using MariaDB and Apache with SSL.

## 1. Prerequisites

- Linux server (Rocky Linux 10)
- Apache installed with SSL support (mod_proxy, mod_ssl)

## 2. Java Installation

Keycloak requires Java. Install OpenJDK 21:

```bash
# Install Java JDK 21
sudo dnf install java-21-openjdk-devel -y

# Verify Java installation
java -version
```
# Keycloak Setup Guide

## 2. Download & Configure Keycloak

```bash
cd /opt  
# Download latest stable Keycloak
sudo wget https://github.com/keycloak/keycloak/releases/download/26.3.3/keycloak-26.3.3.zip  
# Extract
sudo unzip keycloak-26.3.3.zip  
sudo mv keycloak-26.3.3 keycloak  
# Create keycloak user
sudo groupadd keycloak  
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak  
sudo chown -R keycloak:keycloak /opt/keycloak  
# SELinux context
sudo chcon -R -u system_u -t usr_t /opt/keycloak
```

## 3. Bootstrap Admin User

```bash
# Build Keycloak
sudo ./bin/kc.sh build  
# Create temporary bootstrap admin
export ADMIN_PASS=sandeep@123  
sudo --preserve-env=ADMIN_PASS ./bin/kc.sh bootstrap-admin user --username sandeep_admin --password:env ADMIN_PASS
```

## 4. MariaDB Setup

```bash
sudo mysql -u root -p
```

Inside MariaDB:
```sql
CREATE DATABASE sandeep_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'sandeep_user'@'localhost' IDENTIFIED BY 'sandeep@pass';
GRANT ALL PRIVILEGES ON sandeep_db.* TO 'sandeep_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Add JDBC driver:
```bash
cd /opt/keycloak/providers  
sudo wget https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/3.5.5/mariadb-java-client-3.5.5.jar  
sudo chown keycloak:keycloak mariadb-java-client-3.5.5.jar  
sudo chcon -u system_u -t usr_t /opt/keycloak/providers/mariadb-java-client-3.5.5.jar
```

## 5. Keycloak Config File

Edit `/opt/keycloak/conf/keycloak.conf`:

```ini
# DB settings
db=mariadb
db-username=sandeep_user
db-password=sandeep@pass
db-url=jdbc:mariadb://localhost:3306/sandeep_db  

# Proxy & Host
proxy=edge
proxy-headers=xforwarded
hostname=sandeep-keycloak.local

# Ports
http-enabled=true
https-enabled=false
http-port=8080
```

## 6. Systemd Service

File: `/etc/systemd/system/keycloak.service`

```ini
[Unit]
Description=Keycloak Server
After=network.target

[Service]
User=keycloak
Group=keycloak
WorkingDirectory=/opt/keycloak
ExecStart=/opt/keycloak/bin/kc.sh start --optimized
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

Enable service:
```bash
sudo chcon -u system_u -t systemd_unit_file_t /etc/systemd/system/keycloak.service  
sudo systemctl daemon-reload  
sudo systemctl enable --now keycloak  
sudo systemctl status keycloak
```

📸 Screenshot: `screenshots/service-running.png`

## 7. Apache Reverse Proxy + SSL

```bash
sudo dnf install -y mod_ssl certbot python3-certbot-apache
```

File: `/etc/httpd/conf.d/keycloak.conf`

```apache
<VirtualHost *:80>
    ServerName sandeep-keycloak.local
    RewriteEngine On
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>
```

SSL certificate:
```bash
sudo certbot --apache -d sandeep-keycloak.local
```

## 8. Access Keycloak

👉 **URL:** [https://sandeep-keycloak.local](https://sandeep-keycloak.local)  

### Step 1: Login with Bootstrap Admin
- Open the above URL in your browser.  
- Click on **Administration Console**.  
- Enter the bootstrap admin credentials:

```
Username: sandeep_admin
Password: sandeep@123
```

📸 **Screenshots:**  
- `screenshots/keycloak-login.png`  
- `screenshots/admin-console.png`  

---

### Step 2: Create Permanent Admin User
The bootstrap admin is only meant for the initial setup. We will now create a permanent admin user.

1. Go to **Users → Add User**.  
2. Enter details for the new admin user (e.g., `permanent_admin`).  
3. In the **Credentials tab**, set a strong password and **disable "Temporary"**.  
4. Go to **Role Mappings** → Assign the `realm-admin` role under **Realm Management**.  
5. Save changes.  
6. **Logout** from the bootstrap user (`sandeep_admin`).  
7. Log back in with the new permanent admin credentials.  
8. Once confirmed, **delete the bootstrap user** (`sandeep_admin`) for security.

---

## 9. Create Realm

### Step 1: Create a New Realm
1. In the Admin Console, click the **realm dropdown** (top-left).  
2. Click **Create Realm**.  
3. Enter the following details:
   - **Realm name:** `sandeep-realm`  
4. Click **Create**.  

📸 **Screenshot:**  
- `screenshots/realm-created.png`  

---

### Step 2: Create a Test User
1. Go to **Users → Add User**.  
2. Enter:
   - **Username:** `sandeep-user`  
3. Navigate to the **Credentials tab**:  
   - Set a password.  
   - Disable the **Temporary** option.  
   - Save.  
4. Test login:  
   - Go to [https://sandeep-keycloak.local/realms/sandeep-realm/account](https://sandeep-keycloak.local/realms/sandeep-realm/account)  
   - Log in with the test user credentials.  

---

✅ At this point, you have:  
- A **permanent admin account**.  
- A **new realm (`sandeep-realm`)**.  
- A **test user (`sandeep-user`)** to validate the setup.  
