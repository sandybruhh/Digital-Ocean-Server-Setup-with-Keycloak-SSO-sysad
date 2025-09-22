# PHP Application Integration with Keycloak SSO

## Overview

This guide demonstrates how to integrate a generic PHP application with Keycloak for Single Sign-On (SSO) authentication. The implementation uses OpenID Connect protocol to authenticate users through Keycloak and provides secure HTTPS access with SSL certificates.

## Prerequisites

Before proceeding with this integration, ensure you have:

-  Keycloak server running and accessible
-  Apache web server configured and running
-  PHP 8.3+ installed with required extensions
-  Composer package manager installed
-  SSL certificates configured (Let's Encrypt recommended)
-  Keycloak realm `sso-apps` created and configured
-  Basic understanding of PHP and session management

## Setup PHP Directory
**Steps**:
- Create dedicated directory `/var/www/php_app` for the PHP application
- Set Apache user and group ownership for proper web server access
- Configure read and execute permissions for all users, write access for owner

```bash
sudo mkdir /var/www/php_app
sudo chown -R apache:apache /var/www/php_app
sudo chmod -R 755 /var/www/php_app
```

## Create Virtual Host for PHP App
**Steps**:
- Define server name for your PHP application domain
- Set document root to the PHP application directory
- Allow .htaccess file overrides for URL rewriting and redirects
- Grant access permissions to all users for the application directory

Create `/etc/httpd/conf.d/php_app.conf`:

```apache
<VirtualHost *:80>
    ServerName your_php_app_domain
    DocumentRoot /var/www/php_app
    <Directory /var/www/php_app>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

## SSL Configuration (HTTPS)
**Steps**:
- Configure SSL virtual host listening on port 443
- Set same server name and document root as HTTP configuration
- Specify SSL certificate file path from Let's Encrypt
- Define SSL private key file location
- Include Let's Encrypt SSL security options and configurations
- Restart Apache: `sudo systemctl restart httpd`

Add SSL configuration to the same file:

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName your_php_app_domain
    DocumentRoot /var/www/php_app
    <Directory /var/www/php_app>
        AllowOverride All
        Require all granted
    </Directory>
    SSLCertificateFile /etc/letsencrypt/live/your_php_app_domain/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/your_php_app_domain/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```

## Keycloak SSO Integration

### Install PHP OIDC Library

```bash
cd /var/www/php_app
sudo composer require jumbojett/openid-connect-php
```

### Configure Keycloak Client

1. Open Keycloak Admin console
2. Switch to sso-apps realm
3. Navigate to Manage > Clients > Create client
4. Set client name: `php-app`
5. Turn Client Authentication: `on`
6. Valid redirect urls: `https://your_php_app_domain/callback.php`
7. Save and copy Client Secret from Credentials tab


