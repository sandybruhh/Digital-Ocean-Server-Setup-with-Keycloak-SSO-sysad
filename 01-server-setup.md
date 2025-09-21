# Digital Ocean Server Implementation
  1. Digital Ocean Droplet & Initial Setup Implementation
This section documents the implementation steps to set up the Digital Ocean droplet.
## A.Droplet Creation
The following configuration is implemented for the droplet:
  1. Log in to Digital Ocean Control Panel.
  2. Navigate to Manage > Droplets > Create
  3. Region: Banglore & Data Centre: BR1
  4. OS Image: Rocky Linux Version: 10x64
  5. Plan: Baisc Shared CPU with 2GB/1CPU Plan.
  6. Authenthication: Add SSH key for secure access.
  7. IPv6: Configure IPv6 by checking Advanced Options > Enable IPv6.
  8. Hostname: Set Hostname as fossee-sso-task.



B. Initial Server Hardening:

Connect to the droplet as root user for initial setup:
```
ssh root@droplet_ip 
```
Create a new administrative user with sudo priviledges:
```bash
# Create the user and set a strong password
adduser your_username
passwrd your_username

# Add the user to the 'wheel' group to grant sudo access
usermod -aG wheel your_username
```
Copy SSH key to the new user for secure access:
```bash
# Copy SSH key for passwordless login
rsync --archive --chown=your_username:your_username ~/.ssh /home/your_username
```
Enchance security by disabling root SSH login. Modify ```/etc/ssh/sshd_config``` changing ```PermitRootLogin yes``` to ```PermitRootLogin no```:


# Digital Ocean Server Implementation Documentation

## 1. Digital Ocean Droplet & Initial Setup Implementation

This section documents the implementation steps to set up the Digital Ocean droplet.

### A. Droplet Creation

The following configuration is implemented for the droplet:

1. Log in to Digital Ocean control panel
2. Navigate to Manage > Droplets > Create
3. Region: Bangalore & Data center: BR1
4. OS Image: Rocky Linux Version: 10x64
5. Plan: Basic Shared CPU with 2GB/1 CPU Plan
6. Authentication: Add SSH key for secure access
7. IPv6: Configure IPv6 by checking Advanced Options > Enable IPv6
8. Hostname: Set hostname as fossee-sso-task

![Droplet Creation](./screenshots/01-images/droplet-overview.png)
*Digital Ocean droplet configuration and creation*

### B. Initial Server Hardening

Connect to the droplet as root user for initial setup:

```bash
ssh root@droplet_ip
```

Create a new administrative user with sudo privileges:

```bash
# Create the user and set a strong password
adduser your_username
passwd your_username

# Add the user to the 'wheel' group to grant sudo access
usermod -aG wheel your_username
```

Copy SSH key to the new user for secure access:

```bash
# Copy SSH key for passwordless login
rsync --archive --chown=your_username:your_username ~/.ssh /home/your_username
```

Enhance security by disabling root SSH login. Modify /etc/ssh/sshd_config changing PermitRootLogin yes to PermitRootLogin no:

```bash
# Change PermitRootLogin yes to PermitRootLogin no, save and exit
sudo vi /etc/ssh/sshd_config
```

```bash
# Apply the security change
sudo systemctl restart sshd
```

Verify new user access by logging out and reconnecting: `ssh your_username@your_droplet_ip`

![SSH Login](./screenshots/01-images/ssh-login.png)
*New user ssh login*

### C. Firewall Configuration

Implement firewall rules using firewalld:

```bash
# Install, enable and start the firewall service
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld

# Configure permanent rules for essential services
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp # For Keycloak access
sudo firewall-cmd --permanent --add-service=ssh

# Apply the new firewall rules
sudo firewall-cmd --reload
```

![Firewall Configuration](./screenshots/01-images/firewall-status.png)
*Firewall rules configuration*

### D. System Updates & Core Component Installation

Perform comprehensive system setup and package installation:

```bash
# Update all system packages to latest versions
sudo dnf update -y

# Install EPEL and Remi repositories for current packages
sudo dnf install epel-release -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y
sudo dnf module enable php:remi-8.3 -y

# Install required services: Apache, PHP, MariaDB, Python, and utilities
sudo dnf install httpd php php-cli php-mysqlnd php-gd php-xml php-mbstring php-json php-fpm mariadb-server python3 python3-pip unzip wget -y

# Enable and start all core services
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
sudo systemctl enable --now mariadb

# Secure the database installation
sudo mysql_secure_installation
```

![System Update and Core Packages Installation](./screenshots/01-images/update.png)
*System updates and core package installation*

![HTTPd Service Status](./screenshots/01-images/httpd-status.png)

![MariaDB Service Status](./screenshots/01-images/mariadb-status.png)

![PHP Service Status](./screenshots/01-images/php-status.png)
*Service status verification*

### Security Considerations

- Ensure SSH keys are properly configured
- Disable password authentication if using SSH keys only
- Regularly update system packages
- Monitor firewall rules and adjust as needed
- Use strong passwords for database and user accounts

**Next Steps:** Proceed with Keycloak installation and configuration as documented in [02-keycloak-setup.md](02-keycloak-setup.md).
