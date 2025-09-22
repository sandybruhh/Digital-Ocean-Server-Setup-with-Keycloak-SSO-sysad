# Django Project Setup and Keycloak SSO Integration

This guide provides a step-by-step process for setting up a Django application, deploying it with Gunicorn and Apache, and integrating it with Keycloak for Single Sign-On (SSO).

---

## 1. Database Configuration
Start by creating a dedicated database and user account for your Django project.

**Access MySQL/MariaDB CLI as root:**

```bash
sudo mysql -u root -p
```

**Create database and user:**

```sql
CREATE DATABASE djangodb;
CREATE USER 'djangouser'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON djangodb.* TO 'djangouser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

⚠ Note: Replace 'your_secure_password' with a secure, unique password.

---

## 2. Django Project Initialization

**Install necessary development libraries:**

```bash
sudo dnf install python3-devel mariadb-devel gcc -y
```

**Create project directory and virtual environment:**

```bash
sudo mkdir /var/www/django_project
sudo chown your_username:your_username /var/www/django_project
cd /var/www/django_project
python3 -m venv venv
source venv/bin/activate
```

**Install Django and dependencies:**

```bash
pip install django gunicorn mozilla-django-oidc mysqlclient
```

**Create Django project:**

```bash
django-admin startproject mysite .
```

**Configure settings (mysite/settings.py):**

```python
ALLOWED_HOSTS = ['your_django_domain', 'your_droplet_ip', 'localhost']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'djangodb',
        'USER': 'djangouser',
        'PASSWORD': 'your_secure_password',
        'HOST': 'localhost',
        'PORT': '3306'
    }
}

STATIC_URL = 'static/'

import os
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

**Complete initial setup:**

```bash
python manage.py collectstatic
python manage.py migrate
python manage.py createsuperuser
deactivate
```

---

## 3. Apache Reverse Proxy Configuration

**Create Apache config at /etc/httpd/conf.d/django.conf:**

```apache
<VirtualHost *:80>
    ServerName your_django_domain
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
</VirtualHost>
```

**Secure domain with SSL:**

```bash
sudo certbot --apache -d your_django_domain
```

**Modify SSL virtual host to include X-Forwarded-Proto:**

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName your_django_domain
    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
    RequestHeader set X-Forwarded-Proto "https"
    ErrorLog /var/log/httpd/django_error.log
    CustomLog /var/log/httpd/django_access.log combined
    SSLCertificateFile /etc/letsencrypt/live/your_django_domain/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/your_django_domain/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```

**Update settings.py for HTTPS:**

```python
SECURE_SSL_REDIRECT = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

---

## 4. Gunicorn Service Creation

**Create systemd service at /etc/systemd/system/gunicorn.service:**

```ini
[Unit]
Description=Gunicorn for Django mysite
After=network.target
[Service]
User=your_username
Group=apache
WorkingDirectory=/var/www/django_project
Environment="PATH=/var/www/django_project/venv/bin"
ExecStart=python3 /var/www/django_project/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 mysite.wsgi:application
Restart=always
[Install]
WantedBy=multi-user.target
```

**Reload systemd and restart services:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn
sudo systemctl status gunicorn
sudo systemctl restart httpd
```

---

## 5. Keycloak SSO Integration

### A. Create a Keycloak Client
- Client ID: `django`
- Enable Client Authentication
- Add valid redirect URI: `https://your_django_domain/oidc/callback/`
- Copy client secret from Credentials tab

### B. Update Django Settings (mysite/settings.py)

```python
INSTALLED_APPS += ['mozilla_django_oidc']

AUTHENTICATION_BACKENDS = [
    'mozilla_django_oidc.auth.OIDCAuthenticationBackend',
    'django.contrib.auth.backends.ModelBackend',
]

OIDC_RP_CLIENT_ID = "django"
OIDC_RP_CLIENT_SECRET = "your_client_secret"
OIDC_OP_AUTHORIZATION_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/auth"
OIDC_OP_TOKEN_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/token"
OIDC_OP_USER_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/userinfo"
OIDC_RP_SIGN_ALGO = "RS256"
OIDC_OP_JWKS_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/certs"
OIDC_OP_LOGOUT_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/logout"
OIDC_STORE_ID_TOKEN = True
LOGIN_URL = 'oidc_authentication_init'
LOGIN_REDIRECT_URL = '/profile/'
LOGOUT_REDIRECT_URL = '/'
```

---

## 6. Create Django App for User Pages

**Create app:**

```bash
cd /var/www/django_project
source venv/bin/activate
python manage.py startapp home
```

**Add to INSTALLED_APPS:**

```python
INSTALLED_APPS += ['home']
```

**Define views in home/views.py:**

```python
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from django.contrib.auth import logout as django_logout
from django.conf import settings
import urllib.parse

def login(request):
    return render(request, 'login.html')

@login_required
def profile(request):
    return render(request, 'profile.html')

@login_required
def logout(request):
    django_logout(request)
    id_token = request.session.get('oidc_id_token')
    post_logout_redirect = request.build_absolute_uri(settings.LOGOUT_REDIRECT_URL)
    if id_token:
        logout_url = (
            f"{settings.OIDC_OP_LOGOUT_ENDPOINT}"
            f"?id_token_hint={id_token}"
            f"&post_logout_redirect_uri={urllib.parse.quote(post_logout_redirect)}"
        )
    else:
        logout_url = post_logout_redirect
    return redirect(logout_url)
```

**Create HTML templates:**

`home/templates/profile.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Django Profile</title>
  </head>
  <body>
    <h1>Welcome, Sandeep!</h1>
    <p>You have successfully logged in with SSO.</p>
    <p>Username: {{ user.get_username }}</p>
    <p>Email: {{ user.email }}</p>
    <form action="{% url 'logout' %}" method="post">
      {% csrf_token %}
      <button type="submit">Logout</button>
    </form>
  </body>
</html>
```

`home/templates/login.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Django Login</title>
  </head>
  <body>
    <h1>Login Page</h1>
    <a href="{% url 'oidc_authentication_init' %}">Login with SSO</a>
  </body>
</html>
```

**Setup URL routing:**

`home/urls.py`

```python
from django.urls import path
from .views import login, profile, logout

urlpatterns = [
    path('', login, name='login'),
    path('profile/', profile, name='profile'),
    path('logout/', logout, name='logout'),
]
```

Include in `mysite/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('oidc/', include('mozilla_django_oidc.urls')),
    path('', include('home.urls')),
]
```

**Restart services:**

```bash
sudo systemctl restart gunicorn
sudo systemctl restart httpd
```

---

## 7. Test SSO Login
- Navigate to `https://your_django_domain`  
- Click **Login with SSO**  
- Authenticate via Keycloak  
- You should be redirected to the Django profile page after login
