# FOSSEE System Administrator Task

This repository is my submission for the FOSSEE Semester Long Internship - Autumn 2025.
Name of Participants: Sandeep Solanki and Myisha Porwal
GitHub Repository Submission: https://github.com/sandybruhh/Digital-Ocean-Server-Setup-with-Keycloak-SSO-sysad
## Table of Contents

- **Project Overview**
- **Live Environment Access**
- **Documentation Files**

## Project Overview

This project outlines a comprehensive guide for setting up a **Digital Ocean droplet** with **Rocky Linux 10**, configuring **Keycloak** for Single Sign-On (SSO), and deploying three separate applications: **Drupal 11**, a **Django** project, and a generic **PHP** application.

## Live Environment Access

The entire environment is live and publicly accessible for evaluation. You can test the SSO flow by logging into any of the applications with a Keycloak user.

**Droplet IP:** [Your Droplet IP]

**Keycloak Admin Console:** [Your Keycloak URL]

**Drupal Application:** [Your Drupal URL]

**Django Application:** [Your Django URL]

**PHP Application:** [Your PHP URL]

## Documentation Files

This repository contains all the necessary documentation and proof of work for the project.

| **File Name** | **Description** |
|---------------|----------------|
| [01-server-setup.md](./01-server-setup.md) | Details the initial steps of creating the Digital Ocean droplet, securing the server with user management and SSH keys, and configuring the firewall. |
| [02-keycloak-install.md](./02-keycloak-install.md) | Covers the installation and setup of the Keycloak authentication server, including Java installation and creating a production-ready systemd service. |
| [03-drupal-integration.md](./03-drupal-integration.md) | Describes the process of setting up Drupal 11, configuring its database, and integrating it with Keycloak for SSO using the Drupal Keycloak module. |
| [04-django-integration.md](./04-django-integration.md) | Provides the steps for deploying a Django project and integrating it with Keycloak for SSO. |
| [05-php-integration.md](./05-php-integration.md) | Outlines the process of deploying a generic PHP application and enabling Keycloak SSO. |

---

## Quick Setup Overview

### 1. Digital Ocean Droplet & Initial Setup

#### A. Create the Droplet & Initial Server Hardening

1. **Create Droplet:** Log in to Digital Ocean, create a droplet with **Rocky Linux 10**, add your **SSH key**, and set the hostname.

2. **Initial Connection:** Connect to the droplet as the root user.

```bash
ssh root@your_droplet_ip
```

3. **Create New User:** Create a new administrative user with sudo privileges and copy your SSH key.

```bash
adduser your_username
passwd your_username
usermod -aG wheel your_username
rsync --archive --chown=your_username:your_username ~/.ssh /home/your_username
```

4. **Disable Root Login:** Edit /etc/ssh/sshd_config and change PermitRootLogin yes to PermitRootLogin no. Then restart the SSH service.

```bash
sudo vi /etc/ssh/sshd_config
# Apply the change
sudo systemctl restart sshd
```

#### B. Configure the Firewall

Install and configure firewalld to allow essential services.

```bash
# Enable and start the firewall
sudo systemctl enable --now firewalld

# Allow essential services permanently
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp # For Keycloak
sudo firewall-cmd --permanent --add-service=ssh

# Apply the new rules
sudo firewall-cmd --reload
```

#### C. Update System & Install Core Components

Install all necessary packages, including Apache, PHP, and MariaDB.

```bash
# Update all system packages
sudo dnf update -y

# Install repositories for up-to-date packages
sudo dnf install epel-release -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y
sudo dnf module enable php:remi-8.3 -y

# Install Apache, PHP, MariaDB, Python, and other tools
sudo dnf install httpd php php-cli php-mysqlnd php-gd php-xml php-mbstring php-json php-fpm mariadb-server python3 python3-pip unzip wget -y

# Enable and start core services
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
sudo systemctl enable --now mariadb

# Secure your database installation
sudo mysql_secure_installation
```

### 2. Keycloak Installation & Configuration

#### A. Install Java & Keycloak

```bash
# Install Java JDK 17
sudo dnf install java-17-openjdk-devel -y

# Navigate to /opt, download Keycloak and create a dedicated user
cd /opt
sudo wget https://github.com/keycloak/keycloak/releases/download/24.0.4/keycloak-24.0.4.zip
sudo unzip keycloak-24.0.4.zip
sudo mv keycloak-24.0.4 keycloak
sudo groupadd keycloak
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
sudo chown -R keycloak:keycloak /opt/keycloak
```

#### B. Create a Systemd Service for Production

Create the service file /etc/systemd/system/keycloak.service to manage Keycloak as a background service.

**File: /etc/systemd/system/keycloak.service**

```ini
[Unit]
Description=Keycloak Authorization Server
After=network.target

[Service]
Type=idle
User=keycloak
Group=keycloak
ExecStart=/opt/keycloak/bin/kc.sh start --optimized --http-port=8080
LimitNOFILE=102400
LimitNPROC=102400
TimeoutStartSec=600
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

**Enable and Start Service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now keycloak
sudo systemctl status keycloak
```

### 3. Drupal 11 Setup & SSO

#### A. Create Database

```sql
sudo mysql -u root -p

CREATE DATABASE drupaldb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'drupaluser'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON drupaldb.* TO 'drupaluser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

#### B. Install Drupal & Configure Apache

```bash
cd /var/www/
sudo dnf install composer -y
sudo composer create-project drupal/recommended-project drupal
sudo chown -R apache:apache /var/www/drupal
sudo chmod -R 755 /var/www/drupal/web
```

**Virtual Host File: /etc/httpd/conf.d/drupal.conf**

```apache
<VirtualHost *:80>
    ServerName your_drupal_domain.com
    DocumentRoot /var/www/drupal/web
    
    <Directory /var/www/drupal/web>
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog /var/log/httpd/drupal_error.log
    CustomLog /var/log/httpd/drupal_access.log combined
</VirtualHost>
```

Restart Apache: `sudo systemctl restart httpd`.

### 4. Django Project Setup & SSO

#### A. Create Database

```bash
sudo mysql -u root -p

CREATE DATABASE djangodb;
CREATE USER 'djangouser'@'localhost' IDENTIFIED BY 'another_secure_password';
GRANT ALL PRIVILEGES ON djangodb.* TO 'djangouser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

#### B. Set Up Django Project with Gunicorn

Create the project, install dependencies, and configure the database.

**File: /var/www/django_project/mysite/settings.py** (Database Configuration Snippet)

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'djangodb',
        'USER': 'djangouser',
        'PASSWORD': 'another_secure_password',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
```

**Gunicorn Service File: /etc/systemd/system/gunicorn.service**

```ini
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=your_username
Group=your_username
WorkingDirectory=/var/www/django_project
ExecStart=/var/www/django_project/venv/bin/gunicorn --workers 3 --bind unix:/var/www/django_project/mysite.sock mysite.wsgi:application

[Install]
WantedBy=multi-user.target
```

**Apache Virtual Host File: /etc/httpd/conf.d/django.conf**

```apache
<VirtualHost *:80>
    ServerName your_django_domain.com
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
</VirtualHost>
```

#### C. Integrate Keycloak SSO with mozilla-django-oidc

**File: /var/www/django_project/mysite/settings.py** (OIDC Configuration)

```python
INSTALLED_APPS = [
    # ...
    'mozilla_django_oidc',
]

AUTHENTICATION_BACKENDS = [
    'mozilla_django_oidc.auth.OIDCAuthenticationBackend',
    'django.contrib.auth.backends.ModelBackend',
]

OIDC_RP_CLIENT_ID = "django"
OIDC_RP_CLIENT_SECRET = "your_client_secret_from_keycloak"
OIDC_OP_AUTHORIZATION_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/auth"
OIDC_OP_TOKEN_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/token"
OIDC_OP_USER_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/userinfo"
OIDC_RP_SIGN_ALGO = "RS256"
OIDC_OP_JWKS_ENDPOINT = "http://your_droplet_ip:8080/realms/master/protocol/openid-connect/certs"

LOGIN_REDIRECT_URL = "/"
LOGOUT_REDIRECT_URL = "/"
```

**File: /var/www/django_project/mysite/urls.py** (URL Configuration)

```python
from django.urls import path, include
from django.contrib import admin

urlpatterns = [
    path('admin/', admin.site.urls),
    path('oidc/', include('mozilla_django_oidc.urls')),
    # ... your other app urls
]
```

### 5. Generic PHP Application & SSO

#### A. Deploy Application & Configure Apache

```bash
sudo mkdir /var/www/php_app
# Copy your PHP application files here
sudo chown -R apache:apache /var/www/php_app
sudo chmod -R 755 /var/www/php_app
```

**Virtual Host File: /etc/httpd/conf.d/php_app.conf**

```apache
<VirtualHost *:80>
    ServerName your_php_app_domain.com
    DocumentRoot /var/www/php_app
    
    <Directory /var/www/php_app>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Restart Apache: `sudo systemctl restart httpd`.

#### B. PHP Keycloak SSO Integration

Install the jumbojett/openid-connect-php library and create the necessary files.

**Install Library:**

```bash
cd /var/www/php_app
sudo composer require jumbojett/openid-connect-php
```

**File: /var/www/php_app/login.php** (SSO Initiator)

```php
<?php
require 'vendor/autoload.php';

use Jumbojett\OpenIDConnectClient;

session_start();

$oidc = new OpenIDConnectClient(
    'http://your_droplet_ip:8080/realms/master',
    'php-app',
    'your_client_secret_from_keycloak'
);

$oidc->authenticate();
$_SESSION['user_info'] = $oidc->requestUserInfo();

header("Location: /profile.php");
exit();
?>
```

**File: /var/www/php_app/profile.php** (Protected Page)

```php
<?php
session_start();

if (empty($_SESSION['user_info'])) {
    header("Location: /login.php");
    exit();
}

$userInfo = $_SESSION['user_info'];

echo "<h1>Welcome, " . htmlspecialchars($userInfo->name) . "</h1>";
echo "<p>Email: " . htmlspecialchars($userInfo->email) . "</p>";
?>
```

**File: /var/www/php_app/callback.php** (SSO Callback Handler)

```php
<?php
require 'vendor/autoload.php';

use Jumbojett\OpenIDConnectClient;

session_start();

$oidc = new OpenIDConnectClient(
    'http://your_droplet_ip:8080/realms/master',
    'php-app',
    'your_client_secret_from_keycloak'
);

$oidc->authenticate();
$_SESSION['user_info'] = $oidc->requestUserInfo('sub', true);

header("Location: /profile.php");
exit();
?>
```
