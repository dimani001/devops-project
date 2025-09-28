# LEMP Stack Deployment on AWS EC2 (Ubuntu 24.04 LTS)

## Introduction
The **LEMP Stack** is a group of open-source software used to serve dynamic websites and web applications.  
LEMP stands for:
- **L**inux – The operating system (Ubuntu in our case).
- **E**ngine-X (Nginx) – Web server.
- **M**ySQL – Relational database management.
- **P**HP – Scripting language for dynamic content.

This guide will show step-by-step how to deploy LEMP on an AWS EC2 Ubuntu 24.04 instance, with clear explanations for each command.


## Prerequisites
- AWS account
- Security group (`steghub-sg`) allowing:
  - Port 22 (SSH)
  - Port 80 (HTTP)
  - EC2 instance: Ubuntu 24.04 LTS  
 <img width="1866" height="765" alt="LEMP INSTANCE" src="https://github.com/user-attachments/assets/30865c05-b06b-4f2d-a39b-db15c85ce34e" />
- Private key: `steghub.pem`
- Basic familiarity with Linux commands (cd, ls, nano, etc.).

## Step 1 — Connect to EC2 Instance

Navigate to the folder containing your `.pem` key:
```bash
cd /c/Users/H.P\ I5\ 8TH\ GEN/Downloads
````

Reason: Change directory to where the SSH key file is stored.

SSH into your EC2 instance:

```bash
ssh -i steghub.pem ubuntu@16.170.224.78
```
<img width="1920" height="1080" alt="ssh'd into server" src="https://github.com/user-attachments/assets/aa9a19bc-074a-4c64-89d7-b1d7b659e974" />

Reason: Securely connect to the EC2 instance using your private key.

**Note:** We do **not** run `chmod 400 steghub.pem` here because it was already done for a previous project (LAMP). That command is only needed to restrict key permissions when first using the key.


## Step 2 — Install and Configure Nginx

Update system package lists:

```bash
sudo apt update
```
<img width="1920" height="1080" alt="updated server" src="https://github.com/user-attachments/assets/50d7dd4e-1c8c-42d2-ab40-3aec119b13d6" />

Reason: Ensures Ubuntu knows the latest available versions of packages.

Install Nginx:

```bash
sudo apt install nginx
```
Reason: Installs the Nginx web server.
<img width="1920" height="1080" alt="Installed Nginx" src="https://github.com/user-attachments/assets/587a5542-6e69-44d0-af35-e7420b6ba003" />

Check Nginx status:

```bash
sudo systemctl status nginx
```
<img width="1920" height="1080" alt="Nginx Server Running" src="https://github.com/user-attachments/assets/217c50dd-8b63-4d73-8c12-00600ed755a0" />

Reason: Confirms Nginx is running after installation.

Test HTTP on localhost:

```bash
curl http://localhost:80
```
<img width="1920" height="1080" alt="Curl LocalHost" src="https://github.com/user-attachments/assets/e9286dd1-39b8-4e2e-85bf-b1cdb5e290f2" />

Reason: Confirms Nginx is serving locally.

Test via public IP in browser:

```
http://16.170.224.78
```
<img width="1920" height="1080" alt="testing nginx publicly" src="https://github.com/user-attachments/assets/35f8f16f-6147-4b52-b9c6-35231a4fd901" />

Reason: Confirms the site is accessible from the internet.



## Step 3 — Install MySQL

Install MySQL server:

```bash
sudo apt install mysql-server
```
<img width="1920" height="1080" alt="mysql installed" src="https://github.com/user-attachments/assets/6bd67cc4-7cf2-4c35-b2cf-a7ec0ccf5f09" />

Reason: Installs the database server.

Log in as root:

```bash
sudo mysql
```
<img width="1920" height="1080" alt="sudo mysql" src="https://github.com/user-attachments/assets/0fd8118f-58b3-4d14-acda-3f9fe6faecce" />

Reason: Opens MySQL shell as root.

Set a password for MySQL root user:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Dimma.1';
EXIT;
```
<img width="1920" height="1080" alt="Alter user" src="https://github.com/user-attachments/assets/6164500d-2af2-4dbe-9a69-58bca530ad9b" />

Reason: Changes authentication method to `mysql_native_password` for PHP compatibility.

Secure MySQL:

```bash
sudo mysql_secure_installation
```

Reason: Improves MySQL security (root password, anonymous users, test DB).

* Password validation level: `1` (MEDIUM)
-Enter root password: Dimma.1
-Password validation policy: 1 (MEDIUM)
-Keep current root password: n
-Remove anonymous users: y
-Disallow root login remotely: y
-Remove test database: y
-Reload privilege tables: y
* Keep current root password: `n` (I kept Dimma.1) which is optional
<img width="1920" height="1080" alt="all set with secured password" src="https://github.com/user-attachments/assets/06583bc5-e1b7-4ea1-9ad5-b9be30332e6a" />


## Step 4 — Install PHP

```bash
sudo apt install php-fpm php-mysql
```
<img width="1920" height="1080" alt="install php" src="https://github.com/user-attachments/assets/5ef09314-d9f9-46c7-b338-b17d66b11dd9" />

Reason: Installs PHP processor and MySQL extension for PHP.


## Step 5 — Configure Nginx for PHP

Create project directory:

```bash
sudo mkdir /var/www/projectLEMP
sudo chown -R $USER:$USER /var/www/projectLEMP
```
<img width="1920" height="1080" alt="mkdir var" src="https://github.com/user-attachments/assets/e267fac9-707d-46ed-ab07-be2df017dea0" />

Reason: Sets up document root and gives current user ownership.

Edit Nginx server block:

```bash
sudo nano /etc/nginx/sites-available/projectLEMP
```

Paste:

```nginx
server {
    listen 80;
    server_name projectLEMP www.projectLEMP;
    root /var/www/projectLEMP;

    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Enable config and disable default:

```bash
sudo ln -s /etc/nginx/sites-available/projectLEMP /etc/nginx/sites-enabled/
sudo unlink /etc/nginx/sites-enabled/default
```

Test syntax and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```
<img width="1920" height="1080" alt="test and reload nginx" src="https://github.com/user-attachments/assets/05f1942a-7bc3-49a9-9fa2-f94741e30a33" />

Reason: Validates config and applies changes.


## Step 6 — Test HTML and PHP

Create dynamic HTML:

```bash
cd /var/www/projectLEMP
sudo bash -c 'echo "Hello LEMP from hostname $(TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600") && curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/public-hostname) with public IP $(TOKEN=$(curl -s -X PUT \
"http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600") && \
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4)" \
> index.html'
```
<img width="1920" height="1080" alt="echo lemp instance" src="https://github.com/user-attachments/assets/f708de7b-1424-4d8e-b9bb-cbdfb49d6e4a" />

Reason: Generates an index.html file showing EC2 hostname and public IP.

Test in browser:

```
http://16.170.224.78
```
<img width="1920" height="1080" alt="local testing" src="https://github.com/user-attachments/assets/48275012-28b6-47ed-8008-e7c796e6e45f" />

Create PHP info page:

```bash
sudo nano /var/www/projectLEMP/info.php
```

Paste:

```php
<?php
phpinfo();
```
<img width="1920" height="1080" alt="php config file" src="https://github.com/user-attachments/assets/c9673fd0-0b7a-4035-b061-1e55772585c6" />


Test:

```
http://16.170.224.78/info.php
```
<img width="1920" height="1080" alt="php infopage" src="https://github.com/user-attachments/assets/42ff1915-1af1-4d57-a97e-5798dfeeba60" />

Delete PHP info file:

```bash
sudo rm /var/www/projectLEMP/info.php
```

Reason: Removes sensitive system info.
<img width="1920" height="1080" alt="rmv php info file" src="https://github.com/user-attachments/assets/1f0acd62-6d61-47b3-b826-18ac0e23f505" />


## Step 7 — MySQL + PHP Integration

### Create Database and User

```bash
sudo mysql
```
<img width="1920" height="1080" alt="sudo mysql with scured user" src="https://github.com/user-attachments/assets/01b142b4-8cf6-4900-a129-4583156d6416" />

In MySQL:

```sql
CREATE DATABASE `hairsbydimani`;
CREATE USER 'chi'@'%' IDENTIFIED WITH mysql_native_password BY 'PassWord.1';
GRANT ALL ON hairsbydimani.* TO 'chi'@'%';
EXIT;
```

Reason: Sets up database and user with PHP-compatible authentication.
<img width="1920" height="1080" alt="create user" src="https://github.com/user-attachments/assets/897d783a-3521-471b-b307-5a48663660de" />

### Verify Access

```bash
mysql -u chi -p
```

Enter password: `PassWord.1`
<img width="1920" height="1080" alt="verify access using psswd" src="https://github.com/user-attachments/assets/0e68dd33-9e8d-46f1-ab18-22ba6aaf736f" />

```sql
SHOW DATABASES;
```
<img width="1920" height="1080" alt="show db" src="https://github.com/user-attachments/assets/b9d4bebf-3397-41a1-945c-0d35b4936216" />

### Create Table and Insert Data


This is optional but makes my setup more personal.

Instead of using a generic database name, I renamed mine to match my hair business brand:

Database name: hairsbydimani

Table name: todo_list (but tailored it for hair business services)

Here’s how I customized it in MySQL:
```sql
CREATE TABLE hairsbydimani.todo_list (
    item_id INT AUTO_INCREMENT,
    content VARCHAR(255),
    PRIMARY KEY(item_id)
);

INSERT INTO hairsbydimani.todo_list (content) VALUES ("wigs");
INSERT INTO hairsbydimani.todo_list (content) VALUES ("hair extensions");
INSERT INTO hairsbydimani.todo_list (content) VALUES ("braids");
INSERT INTO hairsbydimani.todo_list (content) VALUES ("hair coloring");

SELECT * FROM hairsbydimani.todo_list;
EXIT;
```
<img width="1920" height="1080" alt="table creation" src="https://github.com/user-attachments/assets/33e497ee-47fb-4c51-84dc-1a3862a99186" />


### Create PHP Script to Display Data

```bash
nano /var/www/projectLEMP/todo_list.php
```

Paste:

```php
<?php
$user = "chi";
$password = "PassWord.1";
$database = "hairsbydimani";
$table = "todo_list";

try {
  $db = new PDO("mysql:host=localhost;dbname=$database", $user, $password);
  echo "<h2>TODO</h2><ol>";
  foreach($db->query("SELECT content FROM $table") as $row) {
    echo "<li>" . $row['content'] . "</li>";
  }
  echo "</ol>";
} catch (PDOException $e) {
    print "Error!: " . $e->getMessage() . "<br/>";
    die();
}
```
<img width="1920" height="1080" alt="config file for php todolist" src="https://github.com/user-attachments/assets/46edafb7-ef2d-41b2-85a0-3fed48dd1471" />

Test in browser:

```
http://16.170.224.78/todo_list.php
```

You should see:

```
TODO
1. wigs
2. hair extensions
3. braids
4. hair coloring
```
<img width="1920" height="1080" alt="todo-list php result" src="https://github.com/user-attachments/assets/e397f73a-6ded-438a-9800-c29482208fed" />

By following this guide, I successfully deployed a LEMP stack on an AWS EC2 Ubuntu 24.04 instance, configured Nginx to serve dynamic PHP content, secured MySQL with a dedicated user, and connected our application to a custom database.

I didn’t just stop at the basics, I tailored the database to match a real-world business case, using hairsbydimani as the database name and a service-oriented todo_list table for our offerings. This makes the setup both functional and brand-ready.

This documentation can now serve as a reference for future deployments or be adapted for other businesses. Whether it’s for a hair salon, e-commerce store, or portfolio site, the same steps can be reused with different configurations.

With the LEMP stack in place, you now have a secure, scalable foundation to build upon — from adding SSL for encrypted connections to deploying more complex PHP applications and APIs.

By : Ani Chinecherem Chidimma
LinkedIn: linkedin.com/in/ani-chinecherem
Email: chidimma516@gmail.com
