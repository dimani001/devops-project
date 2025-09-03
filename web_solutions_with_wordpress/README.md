#  Web Solution with WordPress (Three-Tier Architecture on AWS EC2)

This project implements a **three-tier architecture** using AWS EC2 instances:

1. **Presentation Layer** → Browser / Apache Web Server (WordPress)
2. **Application Layer** → WordPress + PHP (on Red Hat Server)
3. **Database Layer** → MySQL DB Server (on separate Red Hat Server)

Both the Web Server and DB Server use attached **EBS volumes** with **LVM partitioning** for storage management.


##  Step 1 — Provision EC2 Instances

* **WordPress Server (Web Server)**: Red Hat EC2
* **DB Server**: Red Hat EC2
* Attach **3 EBS Volumes (10GB each)** to each server (in the same AZ: `eu-north1c`).


##  Step 2 — Configure Storage (Web Server)

### Connect to Web Server

```bash
ssh -i steghub.pem ec2-user@13.53.138.152
```
- Connect to the EC2 instance using your private key.


### Inspect Disks

```bash
lsblk
```
- List all block devices to confirm the 3 new EBS volumes are attached.

```bash
df -h
```
- Show current disk usage and mounted partitions. Helps verify what is mounted before changes.


### Partition Disks with `gdisk`

For each new disk:

```bash
sudo gdisk /dev/nvme1n1
```
- Open gdisk (GPT partitioning tool) to create a partition on the first volume.

Inside `gdisk`:

```
n     # new partition
<Enter>  # default partition number
<Enter>  # default first sector
<Enter>  # default last sector
<Enter>  # partition type (8300 Linux filesystem)
w     # write changes
y     # confirm
```

Repeat for:

```bash
sudo gdisk /dev/nvme2n1
sudo gdisk /dev/nvme3n1
```

### Verify New Partitions

```bash
lsblk
```
- Confirm partitions now exist (nvme1n1p1, nvme2n1p1, nvme3n1p1).

Expected output:

```
nvme1n1    10G
└─nvme1n1p1
nvme2n1    10G
└─nvme2n1p1
nvme3n1    10G
└─nvme3n1p1
```


### Install LVM & Scan

```bash
sudo yum install lvm2
```
- Install LVM2 (Logical Volume Manager) to manage the 3 volumes as one pool.

```bash
sudo lvmdiskscan
```
- Scan system for disks/partitions available for LVM.


### Create Physical Volumes

```bash
sudo pvcreate /dev/nvme1n1p1
sudo pvcreate /dev/nvme2n1p1
sudo pvcreate /dev/nvme3n1p1
```
- Mark the 3 partitions as physical volumes (LVM building blocks).

```bash
sudo pvs
```
- Verify the physical volumes were created successfully.

### Create Volume Group

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
- Groups the 3 PVs into a Volume Group (VG) named webdata-vg.

```bash
sudo vgs
```
- Displays details of the volume group.


### Create Logical Volumes

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
- Creates two logical volumes:

1. apps-lv → 14G (for website files under /var/www/html).

2. logs-lv → 14G (for logs under /var/log)

```bash
sudo lvs
```
- Shows logical volume details.


### Verify Setup

```bash
sudo vgdisplay -v
sudo lsblk
```
- Verifies that the VG, PVs, and LVs are correctly set up.


### Format Volumes

```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
- Formats both logical volumes with ext4 filesystem.

### Mount Volumes

```bash
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```
- Creates directories for website files and temporary logs.

```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html
```
- Mounts apps-lv to Apache’s web root.

```bash
sudo rsync -av /var/log/ /home/recovery/logs/
```
- Copies existing logs to backup before remount.

```bash
sudo mount /dev/webdata-vg/logs-lv /var/log
```
- Mounts logs-lv to the logs directory.

```bash
sudo rsync -av /home/recovery/logs/ /var/log
```
- Restores old logs back after remount.


### Persist Mounts

```bash
sudo blkid
```
- Gets UUIDs of mounted partitions for permanent mount setup.

```bash
sudo vi /etc/fstab
```
- Opens fstab to configure auto-mount at reboot. Add entries like:

Add UUIDs:

```
UUID=5c2fab08-f4a8-4641-92f8-30411bdd7d09   /var/www/html       ext4    defaults  0 0
UUID=5f1c4a3b-a8f6-447a-82c6-587f4d1eb132   /home/recovery/logs ext4    defaults  0 0
```
i.e : UUID=<UUID-for-apps-lv>   /var/www/html       ext4    defaults  0 0
      UUID=<UUID-for-logs-lv>   /home/recovery/logs ext4    defaults  0 0

Apply changes:

```bash
sudo mount -a
sudo systemctl daemon-reload
```
- Reloads fstab and mounts partitions automatically.

```bash
df -h
```
- Confirms mounted volumes.

##  Step 3 — Configure Storage (DB Server)

### Connect to DB Server

```bash
ssh -i steghub.pem ec2-user@16.171.45.85
```

Repeat the same steps as above on a second RedHat EC2 instance (DB Server) but:,

* Create `db-lv` instead of `apps-lv`.
* Mount it to `/db` instead of `/var/www/html`.

`logs-lv` setup is identical (mounted to `/var/log`).


##  Step 4 — Install WordPress on Web Server

### Update Repository

```bash
sudo yum -y update
```
- Updates package repository to latest versions.


### Install Apache + PHP

```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
- Installs Apache, PHP, and required modules for WordPress.


Start Apache:

```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```
- Enables Apache to start on boot and starts the service immediately.

```bash
sudo systemctl status httpd
```
- Confirms Apache is running.


### Install PHP Modules

```bash
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
```
- Adds extra repositories for PHP.

```bash
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
```
- Resets PHP modules and enables PHP 7.4 (needed by WordPress).

```bash
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
```
- Installs additional PHP extensions required by WordPress.

Enable PHP-FPM:

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```
- Starts and enables PHP FastCGI process manager.


### Allow Apache Execution

```bash
sudo setsebool -P httpd_execmem 1
sudo systemctl restart httpd
```
- Updates SELinux policy to allow Apache to execute memory functions and Restarts Apache to apply changes.

### Test PHP

```bash
sudo vi /var/www/html/info.php
```
- Create a test PHP file.

Add:

```php
<?php
phpinfo();
?>
```
- Verifies PHP is working when accessed in the browser.

Visit:

```
http://<Web-Server-Public-IP>/info.php
```


### Install WordPress

```bash
mkdir wordpress
cd wordpress
```
- Create a working directory for WordPress.

```bash
sudo wget http://wordpress.org/latest.tar.gz
```
- Downloads the latest WordPress release.

```bash
sudo tar -xzvf latest.tar.gz
```
- Extracts the archive.

```bash
sudo rm -rf latest.tar.gz
```
- Removes the archive file to save space.
```bash
cp wordpress/wp-config-sample.php wordpress/wp-config.php
```
- Creates a WordPress config file from the sample.
```bash
sudo cp -R wordpress /var/www/html/
```
- Copies WordPress files into Apache’s web root.


Set SELinux policies:

```bash
sudo chown -R apache:apache /var/www/html/wordpress
```
- Sets ownership to Apache user for WordPress files.
```bash
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
```
- Updates SELinux policies for Apache to write inside WordPress directory
```bash
sudo setsebool -P httpd_can_network_connect=1
```
- Allows Apache to make outbound connections (needed for DB).


##  Step 5 — Install MySQL on DB Server

```bash
sudo yum update
```
- Update DB server packages.

```bash
sudo yum install mysql-server
```
- Install MySQL server.

```bash
sudo systemctl status mysqld
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```
- Verify, restart, and enable MySQL service.


##  Step 6 — Configure MySQL for WordPress

```bash
sudo mysql
```
- Open MySQL shell.

Inside MySQL:

```sql
CREATE DATABASE wordpress;
CREATE USER 'dimma'@'172.31.5.87' IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'dimma'@'172.31.5.87';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit;
```
- Creates a database, a user, and grants permissions for the Web Server Private IP.

##  Step 7 — Configure WordPress DB Connection

- Open MySQL Port

On DB Server’s Security Group, allow port 3306 inbound only from Web Server’s private IP.

Install MySQL client on Web Server:

```bash
sudo yum install mysql
```
- Installs MySQL client to test DB connection.

```bash
sudo mysql -h 172.31.7.221 -u dimma -p
```
- Connects to DB Server from Web Server using private IP.

Check DB access:

```sql
SHOW DATABASES;
```
- Confirms connection is successful.


Update WordPress config:

```bash
cd /var/www/html/wordpress
sudo vi wp-config.php
```

Set:

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'dimma');
define('DB_PASSWORD', 'mypass');
define('DB_HOST', '172.31.7.221');  // DB Server Private IP
```

##  Step 8 — Finalize Network & Access

* Open **MySQL port 3306** on DB Server, restricted to Web Server private IP.
* Open **HTTP port 80** on Web Server for public access.


##  Step 9 — Access WordPress

Visit in browser:

```
http://<Web-Server-Public-IP>/wordpress/
```

Fill in DB credentials to complete WordPress setup.

