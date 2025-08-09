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
  <img width="1905" height="789" alt="AWS INSTANCE (LAMP)" src="https://github.com/user-attachments/assets/700c2845-93fd-4ca8-85a5-4498659643bd" />
- **Security Group**: `steghub-SG` with inbound rules allowing:
  . **22 (SSH)** for secure terminal access
  . **80 (HTTP)** for web traffic
  <img width="1905" height="789" alt="SecurityGroup (LAMP)" src="https://github.com/user-attachments/assets/bfb45ff4-3897-40e5-9286-fa071dc1dca8" />

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
```
```bash
chmod 400 steghub.pem
# This sets the private key file permission to read-only for the user, preventing unauthorized access.
```
```bash
ssh -i steghub.pem ubuntu@16.170.238.160
```
<img width="1920" height="1080" alt="SSH'D into server" src="https://github.com/user-attachments/assets/e2488920-3020-4b21-9846-00278c038a72" />

```bash
sudo apt update && sudo apt upgrade
# This updates package list and upgrades outdated packages to ensure system stability and security.
```
# Update & Install Apache
```bash
sudo apt install apache2
# Installs Apache, the web server that handles HTTP requests and serves web pages.
```
<img width="1920" height="1080" alt="apache installed" src="https://github.com/user-attachments/assets/b21fc0aa-587d-43b0-bbd2-3284e4ca8248" />

```bash
sudo systemctl status apache2
# Checks the status of the Apache service to confirm it is running.
```
<img width="1920" height="1080" alt="apache server running" src="https://github.com/user-attachments/assets/d2ebd078-21df-461f-a564-ffa57c3f380f" />

```bash
curl http://localhost:80
# Tests if the Apache server is serving content locally via port 80.
```
<img width="1920" height="1080" alt="curl  localhost" src="https://github.com/user-attachments/assets/d02c11f9-6ccc-4527-835e-12676ab20067" />

Browser Check (Apache Test Page):
```bash
In Browser: http://<Your-Public-IP>
Example: http://16.170.238.160
Expected Result: Default Apache Ubuntu landing page.
```
<img width="1905" height="2013" alt="Apache2-Ubuntu-Default-Page-It-works-08-06-2025_09_11_PM" src="https://github.com/user-attachments/assets/361ba041-1b84-4165-b2f1-76f1b7476707" />

# Another means to Get Public IP Address
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds:21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/public-ipv4
# Retrieves the public IP of the EC2 instance using IMDSv2, enhancing metadata access security.
```
<img width="1920" height="1080" alt="IP address retrieval" src="https://github.com/user-attachments/assets/a19ae9c1-9a14-461c-8aa8-b7aa62e95ee8" />

# To Install MySql(Database)
```bash
sudo apt install mysql-server
```
<img width="1920" height="1080" alt="mysql installed" src="https://github.com/user-attachments/assets/017d0b64-6569-4a29-9b53-5ff5735b5ed7" />

```bash
sudo mysql
```
<img width="1920" height="1080" alt="Screenshot (337)" src="https://github.com/user-attachments/assets/b7ac7928-be1f-4595-9737-2302d5f53738" />

Inside the MySQL prompt:
```mysql
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
<img width="1920" height="1080" alt="Screenshot (338)" src="https://github.com/user-attachments/assets/a6fdb9b8-4e0e-40b7-a01c-d46a52e6495e" />
<img width="1920" height="1080" alt="Screenshot (339)" src="https://github.com/user-attachments/assets/a027e8d7-5857-4405-9368-7b4e7876e50e" />

```bash
sudo apt install php libapache2-mod-php php-mysql
# Installs PHP and required modules for Apache and MySQL integration.

php -v
# Verifies PHP installation.
```
<img width="1920" height="1080" alt="Screenshot (340)" src="https://github.com/user-attachments/assets/bf689168-a7da-446f-9f49-179f28e5adb4" />

Create Project Directory & Set Permissions
```bash
sudo mkdir /var/www/lampstack
# Creates a project directory where web files will be served from.

sudo chown -R $USER:$USER /var/www/lampstack
# Transfers ownership of the project directory to the current user for easier management.
```
<img width="1920" height="1080" alt="Screenshot (341)" src="https://github.com/user-attachments/assets/b8618477-8644-4134-bcd2-0eb70cb35838" />

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
```
<img width="1920" height="1080" alt="Screenshot (342)" src="https://github.com/user-attachments/assets/09cf811b-d76c-4f4c-b347-0cc7a8b3c5e4" />

```bash
sudo ls /etc/apache2/sites-available
```
<img width="1920" height="1080" alt="Screenshot (343)" src="https://github.com/user-attachments/assets/4d3092da-fab8-4ad5-8786-4a7a0f056bb2" />

```bash
sudo a2ensite lampstack.conf

sudo a2dissite 000-default.conf

sudo apache2ctl configtest
```
<img width="1920" height="1080" alt="Screenshot (345)" src="https://github.com/user-attachments/assets/5febca09-0b40-47ec-ab8f-88935ac0c9e3" />

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
<img width="1905" height="789" alt="LAMP (Apache reloaded)" src="https://github.com/user-attachments/assets/8c443a9a-3466-441f-aba0-7dfa47bb6721" />

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
<img width="1905" height="27084" alt="PHP-8-3-6-phpinfo--08-06-2025_10_47_PM" src="https://github.com/user-attachments/assets/5175158b-8ed2-4f13-8465-e2ff15392a9d" />

```bash
sudo rm /var/www/lampstack/index.php
# Removes the PHP info file after verification for security reasons.
```
<img width="1920" height="1080" alt="Screenshot (349)" src="https://github.com/user-attachments/assets/5fd4b6dd-4b43-452e-afea-0c9a806abe5d" />


End of LAMP Stack Setup – Fully functional with Apache, MySQL, and PHP ready for deployment.

By: ANI CHINECHEREM CHIDIMMA
