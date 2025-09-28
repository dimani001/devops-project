## Apache Load Balancer Implementation for DevOps Tooling Website

## Project Overview

This project enhances the previously implemented 2-tier web application architecture by adding a Load Balancer to distribute traffic across multiple web servers, providing high availability and scalability.

## Why Load Balancing is Necessary?

- The Problem with Direct Server Access:

In our previous setup (Project 7), we had:
1. 2 Web Servers with different public IP addresses
2. Users needed to remember multiple URLs to access the application
3. No automatic failover if one server goes down
4. Manual traffic distribution

- Load Balancing Benefits

1. Single Point of Access: Users access one URL instead of multiple server IPs
2. High Availability: If one web server fails, the LB routes traffic to healthy servers
3. Scalability: Easily add more web servers behind the LB
4. Performance: Distributes load evenly across servers
5. Simplified Maintenance: Take servers offline for maintenance without downtime

## Architecture Components
- Updated Infrastructure

Internet Users → Load Balancer (Apache) → [Web Server 1, Web Server 2] → MySQL Database
                                      ↘ NFS Shared Storage

## Prequisties

- Load Balancer: Ubuntu 20.04 (New Instance)
- Web Server 1: RHEL 8 (172.31.20.187)
- Web Server 2: RHEL 8 (172.31.28.1)
- MySQL Server: Ubuntu (172.31.41.146)
- NFS Server: RHEL 8 (172.31.19.59)
<img width="1920" height="1080" alt="Screenshot (712)" src="https://github.com/user-attachments/assets/65eae57b-c8eb-4d5d-9d7e-1869c4907c48" />

## Step-by-Step Implementation

# Step 1: Create Load Balancer Instance

```bash
# Launch Ubuntu 20.04 instance named "Project-8-apache-lb"
# Open TCP port 80 in Security Group for HTTP traffic

ssh -i <key.pem> ubuntu@<LB IP address>
```
- Ubuntu is chosen for the LB because it has excellent Apache support and stable LTS releases.
<img width="1920" height="1080" alt="ssh'd into LB server" src="https://github.com/user-attachments/assets/31a334c2-65e9-4511-b1bc-fa16bdb69f13" />

## Step 2: Install Apache and Required Modules
```bash
# Update package repository
sudo apt update
# Purpose: Ensures we have the latest package information
<img width="1920" height="1080" alt="server updated" src="https://github.com/user-attachments/assets/7c7e7d79-e0e6-4cc8-bc28-56187e6c2202" />

# Install Apache2
sudo apt install apache2 -y
# Purpose: Installs the Apache web server that will function as our load balancer

# Install XML library (required for some modules)
sudo apt-get install libxml2-dev
# Purpose: Provides XML processing capabilities needed by Apache modules
```
<img width="1920" height="1080" alt="downloaded packages" src="https://github.com/user-attachments/assets/a38071dd-30ac-4bb0-bee3-1f98152fd7bc" />

## Step 3: Enable Apache Modules
```bash
# Enable URL rewriting module
sudo a2enmod rewrite
# Purpose: Allows URL manipulation and redirection

# Enable proxy module
sudo a2enmod proxy
# Purpose: Core module for forwarding requests to backend servers

# Enable balancer module
sudo a2enmod proxy_balancer
# Purpose: Provides load balancing functionality across multiple backend servers

# Enable HTTP protocol support for proxy
sudo a2enmod proxy_http
# Purpose: Allows proxying of HTTP requests

# Enable header manipulation
sudo a2enmod headers
# Purpose: Allows modification of HTTP request and response headers

# Enable traffic-based load balancing method
sudo a2enmod lbmethod_bytraffic
# Purpose: Distributes load based on current traffic volume

# Restart Apache to apply module changes
sudo systemctl restart apache2
# Purpose: Activates all enabled modules
```

# Why these specific modules?:
- proxy and proxy_balancer are core to load balancing functionality
- lbmethod_bytraffic ensures even distribution based on actual traffic
- headers allows proper request/response handling between LB and web servers

## Step 4: Verify Apache Service
```bash
# Check Apache service status
sudo systemctl status apache2
```
- Purpose: Confirms Apache is running correctly after configuration changes
<img width="1920" height="1080" alt="apache server running" src="https://github.com/user-attachments/assets/a3c8988c-55df-420c-9ffc-64d6b545e441" />

## Step 5: Configure Load Balancing
```bash
# Edit Apache default site configuration
sudo vi /etc/apache2/sites-available/000-default.conf
# Purpose: Modify the main configuration file to implement load balancing
```
- Add this configuration inside <VirtualHost *:80> section:
```bash
<Proxy "balancer://mycluster">
    BalancerMember http://172.31.20.187:80 loadfactor=5 timeout=1
    BalancerMember http://172.31.28.1:80 loadfactor=5 timeout=1
    ProxySet lbmethod=bytraffic
</Proxy>

ProxyPreserveHost On
ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```
<img width="1920" height="1080" alt="config test" src="https://github.com/user-attachments/assets/7c442ef2-77c8-4a13-97cc-ad32fb5b29ea" />

# Configuration Explanation:

- <Proxy "balancer://mycluster">: Defines a load balancer cluster named "mycluster"
- BalancerMember: Each line represents one web server in the pool
- loadfactor=5: Equal weight for both servers (5:5 ratio)
- timeout=1: 1-second timeout for server responses
- lbmethod=bytraffic: Distributes requests based on current traffic load
- ProxyPreserveHost On: Preserves original host header from client
- ProxyPass: Routes incoming requests to the balancer cluster
- ProxyPassReverse: Ensures responses are properly rewritten
<img width="1920" height="1080" alt="editing config file" src="https://github.com/user-attachments/assets/a3ac0647-3f36-4862-966c-450cc101760c" />

## Step 6: Apply Configuration

```bash
# Restart Apache to apply load balancing configuration
sudo systemctl restart apache2
# Purpose: Activates the new load balancing setup
```
# Load Balancing Methods Explained

- bytraffic (Our Choice)
. How it works: Distributes load based on the I/O traffic volume
. Best for: General web applications with mixed content types
. Advantage: Automatically adapts to different request sizes

- Alternative Methods:
- byrequests
```bash
ProxySet lbmethod=byrequests
```
. How it works: Distributes requests equally regardless of size
- Best for: Applications with consistent request sizes
- Example: 1000 requests = 500 to each server

- bybusyness
```bash
ProxySet lbmethod=bybusyness
```
. How it works: Sends requests to the least busy server
- Best for: Applications with varying processing times
- Advantage: Considers current server load

- heartbeat
```bash
ProxySet lbmethod=heartbeat
```
- How it works: Monitors server health and avoids failed servers
- Best for: High-availability requirements
- Advantage: Automatic failover detection

## Step 7: Testing and Verification

# Test Load Balancer Access
```bash
# Access via web browser
http://<Load-Balancer-Public-IP>/index.php
# Purpose: Verify the load balancer serves the application correctly
```
<img width="1920" height="1080" alt="LB IP test 1" src="https://github.com/user-attachments/assets/6abd58e1-3bdc-4893-9fe4-8c512cee9875" />
<img width="1920" height="1080" alt="LB IP test 2" src="https://github.com/user-attachments/assets/ca7b3111-a269-4ff4-9bb9-1670b796f715" />

# Monitor Traffic Distribution
```bash
# On each web server, monitor access logs
sudo tail -f /var/log/httpd/access_log
# Purpose: Observe how requests are distributed between servers

# Refresh browser multiple times and observe logs
# Expected: Requests should appear in both servers' logs approximately equally
```
<img width="1920" height="1080" alt="servers logtail" src="https://github.com/user-attachments/assets/c26b3603-c37e-4179-8e5a-1acaf2647ee5" />

# Important Note: NFS Log Configuration

- If you mounted /var/log/httpd/ to NFS in Project 7:

```bash
# Unmount NFS log directory on each web server
sudo umount /var/log/httpd
# Purpose: Each web server needs its own log files for proper monitoring

# Ensure local log directory exists
sudo mkdir -p /var/log/httpd
# Restart Apache to use local logs
sudo systemctl restart httpd
```
<img width="1920" height="1080" alt="Screenshot (711)" src="https://github.com/user-attachments/assets/865d5e1f-b533-464f-97dd-1150d631824a" />

## Step 8: Optional - Local DNS Resolution

# Configure Hosts File on LB Server
```bash
# Edit hosts file for local name resolution
sudo vi /etc/hosts
# Purpose: Simplify configuration with server names instead of IP addresses

# Add these lines:
172.31.20.187 Web1
172.31.28.1 Web2
```
<img width="1920" height="1080" alt="renaming servers" src="https://github.com/user-attachments/assets/b2ed56fb-4402-482e-9238-430905dd6f63" />

# Update LB Configuration with Names

```bash
<Proxy "balancer://mycluster">
    BalancerMember http://Web1:80 loadfactor=5 timeout=1
    BalancerMember http://Web2:80 loadfactor=5 timeout=1
    ProxySet lbmethod=bytraffic
</Proxy>
```
<img width="1920" height="1080" alt="add named servers to config file" src="https://github.com/user-attachments/assets/48d71fa6-43bd-49f5-a20c-b8b14166c0f9" />

# Test Local Resolution

```bash
# Test name resolution from LB server
curl http://Web1
curl http://Web2
# Purpose: Verify local DNS names work correctly
```
<img width="1920" height="1080" alt="curl web 1" src="https://github.com/user-attachments/assets/cc516187-6e6e-46d4-83c7-585e0e8fc0c9" />
<img width="1920" height="1080" alt="curl web 2" src="https://github.com/user-attachments/assets/676a0b25-e5bc-4540-8c25-d2673a34580c" />

Users can access the website through a single URL, traffic is evenly distributed between web servers, and the system can withstand the failure of any single web server without service interruption.
