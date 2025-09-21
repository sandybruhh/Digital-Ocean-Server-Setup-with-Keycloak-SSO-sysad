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
