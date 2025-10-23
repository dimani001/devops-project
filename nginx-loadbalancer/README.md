## Project 10 – Load Balancer Solution With Nginx and SSL/TLS

## Setting Up Load Balancer with NGINX, Domain Name, and SSL with Certbot

This project demonstrates how to deploy a secure and highly available web solution using Nginx as a Load Balancer to distribute traffic across multiple web servers, protected with SSL/TLS encryption using Certbot (Let's Encrypt).

Literally, I am implementing a Load balancer for our webservers, and connect the load balancer to a domain name so our website can be accessed via a domain name, install and configure ssl/tls for security.

## Prerequisites

- Create and configure two web servers (Web1 & Web2) hosting a sample "Tooling Website".
- Set up a dedicated Nginx Load Balancer (Nginx LB) to distribute traffic between them - A Linux server (e.g., Ubuntu) for NGINX
- Secure the setup with an SSL/TLS certificate issued for your real domain: hairbydimani.store. (Domain name registered) 
- Ensure both ssh, http and https ports are opened and are set to allow access from anywhere on the steghub-SG
- Sudo privileges on the server
- Configure automatic certificate renewal with cron.

## PART 1 – CONFIGURE NGINX AS A LOAD BALANCER

### Step 1: Launch a New EC2 Instance for Nginx
```bash
# Launch Ubuntu 20.04 instance on AWS
Reason:
Ubuntu is lightweight, stable, and optimized for web server and load balancer roles.

Required ports:

TCP 80 (HTTP) – for standard web traffic

TCP 443 (HTTPS) – for encrypted web traffic

Assign an Elastic IP to ensure your DNS record (domain) always points to the same public IP, even after instance restarts.

First Install and Configure Nginx

bash
sudo apt update
sudo apt install nginx -y
Reason:
Updates package lists and installs the latest stable Nginx version.

Step 2: Update Local DNS Records
bash
sudo vi /etc/hosts
Add your web servers' private IPs:

text
172.31.xx.xx  Web1
172.31.yy.yy  Web2
Reason:
This creates local name resolution so that Nginx can refer to your backend servers (Web1, Web2) without needing public DNS lookups.

Now edit the Nginx configuration:

bash
sudo vi /etc/nginx/nginx.conf
Add this under the http block and comment out include /etc/nginx/sites-enabled/*;:

nginx
http {
  upstream myproject {
      server Web1 weight=5;
      server Web2 weight=5;
  }

  server {
      listen 80;
      server_name hairbydimani.store www.hairbydimani.store;

      location / {
          proxy_pass http://myproject;
      }
  }

  # include /etc/nginx/sites-enabled/*;
}
Replace your already purchased domain name in the section for domain name, save and exit

Reason:

upstream myproject: Defines a group of backend servers (Web1, Web2).

weight=5: Ensures even load distribution between servers.

proxy_pass: Forwards all requests from Nginx to backend servers.

Commenting the default include avoids duplicate or conflicting server blocks.

Restart Nginx:

bash
sudo systemctl restart nginx
sudo systemctl status nginx
Reason:
Restarts Nginx to apply changes and verifies the configuration is active.

Step 3: Configure Elastic IP address for the load balancer
Every time an EC2 instance is stopped and started again, its public IP address changes. If that IP is already linked to a DNS record, this means you would need to manually update the DNS each time the instance restarts — a tedious and error-prone process.

To prevent this, AWS provides a service called Elastic IP. An Elastic IP is a static public IP address that remains permanently associated with your account and can be attached to an EC2 instance. Unlike a regular public IP, it does not change when the instance is stopped or restarted.

To allocate and associate an Elastic IP:

In the AWS Management Console, search for Elastic IP in the search bar.

Click Allocate Elastic IP address and ensure it's in the same region as your instance.

After allocation, select the Elastic IP, then choose Actions → Associate Elastic IP address.

Associate it with your Load Balancer instance.

Step 4: Setting Up DNS Records for Your Domain to Point to the Server Using the Elastic IP Address
After obtaining and associating your Elastic IP with your Load Balancer instance, the next step is to link your domain name to this static IP so that anyone who types your domain in a browser is directed to your web application.

This is done through DNS configuration, where you create or modify A Records — the DNS entries that map a domain name (like www.hairbydimani.store) to an IP address (your Elastic IP).

Steps:

Log in to your domain registrar's website (the platform where you purchased your domain, e.g., GoDaddy, Namecheap, or Domain.com).

Navigate to the DNS Management or Zone Editor section for your domain.

Locate the A Record section.

For the Host field, enter @ (this represents your root domain, e.g., hairbydimani.store).

In the Points to field, enter your Elastic IP address.

Save the changes.

(Optional but recommended) — Create another A Record for the www subdomain so that www.hairbydimani.store also resolves correctly.

Host: www

Points to: the same Elastic IP address.

Why This Is Important:
This step ensures that your domain name is properly linked to your server. Without it, browsers would not know where to direct traffic when someone enters your domain name, resulting in "site not found" errors.

Output:
Once DNS records are updated, you can verify the propagation (i.e., that the changes have taken effect globally):

Visit https://dnschecker.org/

Enter your domain name (e.g., hairbydimani.store) and select the record type A.

Click Search.

You should see your Elastic IP address appear across multiple DNS servers — this confirms your domain is correctly pointing to your load balancer.

Output Example:

hairbydimani.store → 13.51.76.136

www.hairbydimani.store → 13.51.76.136

Output: Your website is accessible using your domain name.

PART 2 – SECURE THE LOAD BALANCER WITH SSL/TLS
Step 1: Install Certbot (Let's Encrypt)
After linking your domain to your Nginx Load Balancer, the next step is to secure your website using SSL/TLS encryption. This ensures all communication between your users and the website is encrypted and safe.

Ensure the snapd service is active and running:

bash
sudo systemctl status snapd
Output: snapd active (running)

Install Certbot using snapd:

bash
sudo snap install --classic certbot
Create a symlink for Certbot:

bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot
Output: certbot installed

Reason:

Certbot automates the SSL certificate request and configuration.

Symbolic linking (ln -s) allows running certbot without full path.

Request your SSL/TLS certificate:

bash
sudo certbot --nginx
Follow the prompts to select your domain name and confirm the HTTPS setup.

Output: SSL installed successfully

Verify SSL installation:
Once completed, visit your website in the browser using https://your-domain-name.com.

If the setup is correct, you should see a padlock icon in the browser's address bar — confirming that SSL is active and the connection is secure.

Output: Website secured with SSL
Output: Website loads successfully

Step 2: Automate SSL Renewal with a Cron Job
Let's Encrypt SSL certificates are valid for 90 days. To avoid manual renewal, configure an automated renewal process using Cron Jobs.

A Cron Job is a scheduled task on Unix/Linux systems that automatically runs commands at specific intervals — useful for repetitive administrative tasks like certificate renewal.

Test the renewal command:

bash
sudo certbot renew --dry-run
Output: Renewal test successful

Set up an automated renewal task:

bash
crontab -e
Add the following line to schedule automatic SSL renewal twice daily:

text
* */12 * * *   root /usr/bin/certbot renew > /dev/null 2>&1
Save and exit.

Result:
Your SSL certificate will now be automatically renewed without manual intervention, ensuring your website remains secure at all times.

Final Output
You have now successfully:

Configured Nginx as a Load Balancer for your web servers

Linked it to your domain name

Installed and automated renewal of SSL/TLS certificates

Your website (https://hairbydimani.store) is now secure, functional, and production-ready.

By Chinecherem Chidimma Ani
