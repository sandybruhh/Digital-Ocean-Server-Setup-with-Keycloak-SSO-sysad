# 3. Integrate KeyCloak SSO
## Install Drupal & Configure Apache

### Install Drupal using Composer

```bash
# Install Drupal using Composer for better dependency management
cd /var/www/
sudo dnf install composer -y
sudo composer create-project drupal/recommended-project drupal
sudo chown -R apache:apache /var/www/drupal
sudo chmod -R 755 /var/www/drupal/web
```

### Create Apache Virtual Host

Create an Apache virtual host at `/etc/httpd/conf.d/drupal.conf`:

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

