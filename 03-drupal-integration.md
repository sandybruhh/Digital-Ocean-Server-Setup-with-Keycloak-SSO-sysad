# Drupal 11 Setup and Keycloak SSO Integration

This guide covers the installation of Drupal 11 and its integration with Keycloak for Single Sign-On (SSO) using the miniOrange OAuth Client module.

## 1. Drupal Database Configuration

Begin by creating a dedicated database and user for your Drupal installation.

Access your MySQL/MariaDB server as the root user.

```bash
sudo mysql -u root -p
```

Run the following SQL commands to create the database (drupaldb) and a user (drupaluser) with full privileges on it.

```sql
CREATE DATABASE drupaldb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'drupaluser'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON drupaldb.* TO 'drupaluser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

⚠️ Note: Replace 'your_secure_password' with a strong, unique password.

## 2. Drupal Installation

+ Install Drupal and configure the server environment to run the application.

Navigate to the /var/www/ directory and use Composer to install Drupal's recommended project structure.

```bash
cd /var/www/
sudo dnf install composer -y
sudo composer create-project drupal/recommended-project drupal
sudo chown -R apache:apache /var/www/drupal
sudo chmod -R 755 /var/www/drupal/web
```
+ Create the necessary directories and files for drupal to store its data and configuration.
```bash
sudo mkdir /var/www/drupal/web/sites/default/files
sudo cp /var/www/drupal/web/sites/default/default.settings.php /var/www/drupal/web/sites/default/settings.php
```

+ Edit the settings.php file at /var/www/drupal/web/sites/default/settings.php. Add your domain to the trusted_host_patterns array. This is a security measure to prevent HTTP Host header attacks.

```php
// ...existing code...
$settings['trusted_host_patterns'] = [
  '^drupal\.swaruph\.tech$',
];
// ...existing code...
```

+ Replace drupal.swaruph.tech with your actual domain.

Configure SELinux to allow Apache to write to the Drupal directories. This is crucial for the installation and subsequent operations.

```bash
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/drupal(/.*)?"
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/drupal/web/sites/default/settings.php'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/drupal/web/sites/default/files'
sudo restorecon -Rv /var/www/drupal/
```

+ Create an Apache virtual host configuration file for your Drupal site at /etc/httpd/conf.d/drupal.conf.

```apache
<VirtualHost *:80>
    ServerName your_drupal_domain
    DocumentRoot /var/www/drupal/web
    <Directory /var/www/drupal/web>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog /var/log/httpd/drupal_error.log
    CustomLog /var/log/httpd/drupal_access.log combined
</VirtualHost>
```
+ Replace your_drupal_domain with your domain name.

Restart the Apache web server to apply the changes.

```bash
sudo systemctl restart httpd
```

+ Complete the Drupal web installation by navigating to your domain in a web browser. Follow the on-screen prompts, providing the database credentials you configured in the first step.

Secure your site with SSL using Certbot. This will automatically set up HTTPS.

```bash
sudo certbot --apache -d your_drupal_domain
```

## 3. Keycloak SSO Integration

Integrate Drupal with your Keycloak instance to enable SSO.

### A. Install the OAuth Client Module

The miniOrange OAuth Client module is the recommended choice for OIDC integration due to its active maintenance and compatibility. It provides seamless integration with Keycloak, supports advanced configurations, and ensures regular security updates. This makes it a reliable solution for production-grade Single Sign-On (SSO) in PHP and Drupal applications. Install it using Composer:

```bash
cd /var/www/drupal
sudo composer require 'drupal/miniorange_oauth_client'
```

### B. Enable the Drupal Module

- Log into your Drupal admin dashboard.
- Go to the Extend menu.
- Search for miniOrange OAuth Client and enable the module.

### C. Configure Drupal as an OAuth Client

- Navigate to Configuration > People > miniOrange OAuth Client Configuration.
- Under the Client Configuration tab, click + Add New.
- Select Keycloak as the application and set a custom name, such as "Keycloak."
- Copy the provided Callback/Redirect URL; you'll need this for Keycloak setup.

### D. Configure Keycloak

- Access your Keycloak admin console.
- Switch to your desired realm (e.g., sso-apps).
- Go to Clients and click Create client.
- Set the Client type to OpenID Connect and the Client ID to drupal.
- On the Capability config screen, turn on Client Authentication.
- Under Login Settings, paste the Callback/Redirect URL you copied from Drupal into the Valid redirect URIs field.
- Save the client, then go to the Credentials tab and copy the generated Client Secret.

### E. Integrate Drupal with Keycloak

- Return to the miniOrange OAuth Client Configuration page in Drupal.
Enter the following details:
- Client ID: ```drupal```
- Client Secret: The secret you copied from Keycloak.
- Authorization Endpoint: ```{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/auth```
- Token Endpoint: ```{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/token```
- User Info Endpoint: ```{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/userinfo```
- Make sure to replace ```{your_keycloak_domain} with your actual Keycloak domain.```
- Click Save Configuration.

### F. Test the Integration

- On the Drupal miniOrange OAuth Client Configuration page, click Perform Test Configuration.
- A new window will open, prompting you to log in to Keycloak.
- Upon successful login, you'll





