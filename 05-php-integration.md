## Setup PHP Directory

```bash
sudo mkdir /var/www/php_app
sudo chown -R apache:apache /var/www/php_app
sudo chmod -R 755 /var/www/php_app
```

## Create Virtual Host for PHP App

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
