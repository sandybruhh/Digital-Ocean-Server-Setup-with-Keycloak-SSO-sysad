## 1. Create Database
```bash
sudo mysql -u root -p
```sql
CREATE DATABASE myappdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'myappuser'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON myappdb.* TO 'myappuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
