# LAMP Stack Project – StegHub DevOps Training

# Introduction

**LAMP** is an acronym representing a stack of open-source technologies used to develop and deploy web applications. It stands for:

- **Linux** – Operating system
- **Apache** – Web server
- **MySQL** – Database server
- **PHP** – Server-side scripting language

LAMP is one of the best choices for web development due to its **reliability**, **flexibility**, and **cost-effectiveness**, making it ideal for hosting dynamic websites and applications.


# Prerequisites

To implement this project, the following were used:

- **AWS EC2 Ubuntu 24.04 LTS instance**
- **Security Group**: `steghub-SG` with inbound rules allowing:
  . **22 (SSH)** for secure terminal access
  . **80 (HTTP)** for web traffic
- **Key Pair**: `steghub.pem` private key for SSH authentication
- **Visual Studio Code (VSCode)**- for SSH access and code editing
- **GitHub**- for storing and documenting the project
- **Stable Internet Connection** - for package downloads and remote access
- **AWS CLI** - for quick AWS resource management
- **Basic Linux Terminal Knowledge** to run commands effectively
- **Web Browser** (Chrome) for verification of Apache/PHP pages

# Implementation Steps

```bash
cd /c/Users/H.P\\.I5\\8TH\\GEN/Downloads/

chmod 400 steghub.pem
# This sets the private key file permission to read-only for the user, preventing unauthorized access.

ssh -i steghub.pem ubuntu@16.170.238.160

sudo apt update && sudo apt upgrade
# This updates package list and upgrades outdated packages to ensure system stability and security.

# Update & Install Apache

sudo apt install apache2
# Installs Apache, the web server that handles HTTP requests and serves web pages.

sudo systemctl status apache2
# Checks the status of the Apache service to confirm it is running.

curl http://localhost:80
# Tests if the Apache server is serving content locally via port 80.
```

Browser Check (Apache Test Page):
```bash
In Browser: http://<Your-Public-IP>
Example: http://16.170.238.160
Expected Result: Default Apache Ubuntu landing page.
```
# Another means to Get Public IP Address
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds:21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/public-ipv4
# Retrieves the public IP of the EC2 instance using IMDSv2, enhancing metadata access security.
```
# To Install MySql(Database)
```bash
sudo apt install mysql-server

sudo mysql

Inside the MySQL prompt:
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Password.1';

mysql> exit

sudo mysql_secure_installation
```
Prompts & Responses (your inputs in bold):
```bash
Would you like to setup VALIDATE PASSWORD plugin? Y/n  **Y**
There are three levels of password validation policy:
LOW    Length >= 8
MEDIUM Length >= 8, numeric, mixed case, special chars
STRONG Length >= 8, numeric, mixed case, special chars, dictionary file
Please enter 0 = LOW, 1 = MEDIUM, 2 = STRONG:  **1**

New password:  **Chidimma123!**
Re-enter new password:  **Chidimma123!**

Estimated strength of the password: 100 
Do you wish to continue with the password provided? (Press y|Y for Yes, any other key for No):  **Y**

Remove anonymous users? (Press y|Y for Yes, any other key for No):  **Y**
Disallow root login remotely? (Press y|Y for Yes, any other key for No):  **Y**
Remove test database and access to it? (Press y|Y for Yes, any other key for No):  **Y**
Reload privilege tables now? (Press y|Y for Yes, any other key for No):  **Y**
```

```bash
sudo apt install php libapache2-mod-php php-mysql
# Installs PHP and required modules for Apache and MySQL integration.

php -v
# Verifies PHP installation.
```
Create Project Directory & Set Permissions
```bash
sudo mkdir /var/www/lampstack
# Creates a project directory where web files will be served from.

sudo chown -R $USER:$USER /var/www/lampstack
# Transfers ownership of the project directory to the current user for easier management.
```
Configure Apache Virtual Host
```bash
sudo vi /etc/apache2/sites-available/lampstack.conf
```
Paste:
```bash
<VirtualHost *:80>
       ServerName lampstack
       ServerAlias www.lampstack
       ServerAdmin webmaster@localhost
       DocumentRoot /var/www/lampstack
<Directory /var/www/lampstack>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

sudo ls /etc/apache2/sites-available

sudo a2ensite lampstack.conf

sudo a2dissite 000-default.conf

sudo apache2ctl configtest
```
Create Custom Web Page with IP Info
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds:21600")

echo "<html><body><h1>Hello LAMP from $(hostname) with public IP $(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)</h1></body></html>" | sudo tee /var/www/lampstack/index.html
```
Browser Check (Custom HTML Page):
In Browser: http://<Your-Public-IP>
Example: http://16.170.238.160
Expected Result:
```bash
Hello LAMP from <hostname> with public IP <your-ip>
```
Prioritize PHP Files
```bash
sudo vim /etc/apache2/mods-enabled/dir.conf
# Add index.php first in the DirectoryIndex list.
```
 Reload Apache
 ```bash
 sudo systemctl reload apache2
```
Test PHP Configuration
```bash
vim /var/www/lampstack/index.php
```
Paste:
```php
<?php
phpinfo();
?>
```
 Browser Check (PHP Info Page):
In Browser: http://<Your-Public-IP>
Example: http://16.170.238.160
Expected Result: A PHP configuration page.

```bash
sudo rm /var/www/lampstack/index.php
# Removes the PHP info file after verification for security reasons.
```

End of LAMP Stack Setup – Fully functional with Apache, MySQL, and PHP ready for deployment.

By: ANI CHINECHEREM CHIDIMMA
