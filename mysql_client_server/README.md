# CLIENT-SERVER ARCHITECTURE USING MYSQL

# Introduction

A **Client-Server Architecture** is a network-based computing model where responsibilities are divided between **clients** and **servers**:

* **Client** → Sends requests (e.g., retrieve data, insert records).
* **Server** → Processes requests and returns responses.

This architecture is widely used in:

* Email systems
* Web applications
* Online banking
* E-commerce platforms

In this guide, we will **implement a client-server architecture** using **MySQL** on **two AWS EC2 instances (Ubuntu)**

##  Prerequisites

Before starting, ensure you have:

* An **AWS account**
* Two **Ubuntu EC2 instances** in the same VPC/security group

  * **Server instance** (for MySQL Server)
  * **Client instance** (for MySQL Client)
* A **Key Pair (.pem file)** for SSH access eg steghubkey
* Security group rules allowing:

  * **Port 22 (SSH)** for management
  * **Port 3306 (MySQL)** for remote DB access


##  Steps Involved

### **1. Launch and Connect to EC2 Instances**


```bash
ssh -i <key-pair-name>.pem ubuntu@<ip-address>
```

 Used to **securely connect** to the Ubuntu EC2 instance from your local machine using the private key.


### **2. Update and Upgrade Packages**

```bash
sudo apt update && sudo apt upgrade -y
```

 Ensures the system package index is updated and all packages are upgraded to their latest versions.

### **3. Install MySQL Server (on the Server Instance)**


```bash
sudo apt install mysql-server -y
```

Installs the **MySQL server software** so this machine can act as a **database server**.
<img width="1920" height="1080" alt="installing mysql server" src="https://github.com/user-attachments/assets/e1f29035-f783-4970-b1c3-0ba76f9b7143" />


### **4. Configure MySQL for Remote Access**


```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

 Opens the MySQL configuration file. Change:

```ini
bind-address = 127.0.0.1
```

To:

```ini
bind-address = 0.0.0.0
```
<img width="1920" height="1080" alt="bind address" src="https://github.com/user-attachments/assets/fa97db18-0dfd-4617-9500-ee80ef824748" />

This allows MySQL to accept connections from **any IP address**, not just localhost.

**Restart MySQL**:

```bash
sudo systemctl restart mysql
```

 Applies the configuration changes.


### **5. Create a Remote User on MySQL**


```bash
sudo mysql -u root -p
```

 Logs into MySQL as the **root user** with password authentication.

Inside MySQL shell, run:

```sql
CREATE USER 'dimma'@'%' IDENTIFIED BY 'Password.1';
GRANT ALL PRIVILEGES ON *.* TO 'dimma'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```
<img width="1920" height="1080" alt="setting mysql user" src="https://github.com/user-attachments/assets/65f1fd76-710e-4133-bc56-9b11824b9a55" />

 Explanation:

* `CREATE USER` → Creates a new MySQL user `dimma` with a password.
* `'%'` → Allows login from **any host** (remote access).
* `GRANT ALL PRIVILEGES` → Gives admin-like rights to this user.
* `FLUSH PRIVILEGES` → Reloads privilege tables so changes take effect.
* `EXIT` → Leaves the MySQL shell.


### **6. Setup the MySQL Client (on the Client Instance)**


```bash
sudo apt install mysql-client -y
```

Installs the **MySQL client utility** used to connect to remote MySQL servers.

**Verify Installation**:

```bash
mysql --version
```
<img width="1920" height="1080" alt="mysql client version" src="https://github.com/user-attachments/assets/b99862c4-f029-4d42-bff4-150bd4ae869a" />

 Confirms that the MySQL client was installed correctly.

---

### **7. Connect from Client to Server**


```bash
mysql -u dimma -p -h <server-ip>
```

Example:

```bash
mysql -u dimma -p -h 51.20.114.25
```
  <img width="1920" height="1080" alt="mysql client connection" src="https://github.com/user-attachments/assets/8ee6b10d-d83b-4d4e-aaab-5025bb46e534" />
 Connects the **client machine** to the **server’s MySQL instance** using the remote user credentials.


### **8. Perform Database Operations**

1. **Show available databases**:

   ```sql
   SHOW DATABASES;
  ```

<img width="1920" height="1080" alt="show databases" src="https://github.com/user-attachments/assets/7b16eef4-fda7-40ff-875f-fbb9c6c876d1" />


 Lists all existing databases.

2. **Create a database**:

   ```sql
   CREATE DATABASE dimani;
   ```

    Creates a new database named `dimani`.
<img width="1920" height="1080" alt="create database" src="https://github.com/user-attachments/assets/edf078ce-5037-4c10-a242-14190433d717" />

3. **Use the database**:

   ```sql
   USE dimani;
   ```

    Switches the current session to work inside the `dimani` database.

4. **Create a table**:

   ```sql
   CREATE TABLE users (
     id INT AUTO_INCREMENT PRIMARY KEY,
     name VARCHAR(255) NOT NULL,
     email VARCHAR(255) NOT NULL
   );
   ```

    Defines a `users` table with `id`, `name`, and `email` fields.

5. **Insert data**:

   ```sql
   INSERT INTO users (name, email)
   VALUES ('Ani Chinecherem Chidimma', 'chidimma516@gmail.com');
   ```

    Adds a sample user record into the `users` table.
<img width="1920" height="1080" alt="create table" src="https://github.com/user-attachments/assets/e26d89a7-0a7b-4c60-9445-b0901b059709" />

6. **Verify data**:

   ```sql
   SELECT * FROM users;
   ```

    Displays all records from the `users` table.

7. **Drop the table**:

   ```sql
   DROP TABLE users;
   ```

    Deletes the `users` table.

8. **Drop the database**:

   ```sql
   DROP DATABASE dimani;
   ```

    Deletes the `dimani` database.
<img width="1920" height="1080" alt="drop table and database" src="https://github.com/user-attachments/assets/432cfad0-6f79-47ed-a92e-0b680814a462" />

9. **Exit MySQL**:
    
   ```sql
   EXIT;
   ```

    Ends the MySQL session.


##  Summary

We built a **Client-Server MySQL architecture** on AWS using EC2 instances:

* Installed and configured **MySQL server**
* Created a **remote user** with privileges
* Installed **MySQL client** on another machine
* Connected remotely to the server
* Performed **basic database operations** (create, insert, query, drop)

This demonstrates how real-world **applications and databases** communicate over a network.

