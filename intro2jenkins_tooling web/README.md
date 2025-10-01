## Tooling Website deployment automation with Continuous Integration (Jenkins)

## Project Overview

This project enhances my existing load-balanced infrastructure by implementing Continuous Integration using Jenkins. I added a Jenkins CI server to my Tooling Website architecture, configuring automated deployments that fetch code changes from my GitHub repository, created build artifacts, and copied them to my NFS server (/mnt/apps) where web servers mount content. When code is pushed to the repository, Jenkins will automatically deploy the changes, ensuring all web servers serve the latest version instantly. This automates repeatable deployments, demonstrates CI fundamentals, and streamlines our development workflow.

## Prerequisites
Completed Projects
-  Project 7: Tooling Website with NFS and MySQL: A functional Tooling Website with NFS and MySQL setup.
-  Project 8: Apache Load Balancer implementation: An Apache Load Balancer implementation to distribute traffic across web servers.

Infrastructure Requirements
- AWS EC2 instances: Running instances for NFS, Web Servers, and Load Balancer.
- Jenkins Server: An AWS EC2 instance (Ubuntu 20.04) for Jenkins with:
    - Public IP or DNS reachable from GitHub (or use ngrok/tunnel if needed).
    - Security group: TCP 8080 open from your IP (or GitHub).

Repository and Access Requirements
- GitHub Repository: A working Tooling Website repository (e.g., https://github.com/<yourname>/tooling).
- NFS Server: Reachable from Jenkins over SSH (port 22) and mounted by web servers on /mnt/apps.
- SSH Keypair: Jenkins needs a private key, and the NFS server must have the matching public key in ~/.ssh/authorized_keys for the remote user.


User and Permission Requirements
- NFS Server Account: An account usable by Jenkins (e.g., ec2-user) with appropriate permissions on the /mnt/apps directory.

## Step 1 — Create Jenkins Server EC2 (Ubuntu 20.04)
(Do this in AWS Console or using your existing EC2 setup.) 
- Launch Ubuntu 20.04 LTS EC2 instance
- Name: "Jenkins"
- Instance type: t3.medium (recommended for Jenkins)
- Security Group: Open ports 22 (SSH) and 8080 (Jenkins)
. TCP 8080 from your IP (Jenkins UI).
. TCP 22 from your IP (if you need SSH access to the Jenkins server).

2. ssh into your instance using terminal.
```bash
ssh -i "your-key-pair.pem" ubuntu@your-ec2jenkinsserver-public-ip
```

3. Update your ec2 instance
```bash
sudo apt update && sudo apt upgrade -y
```
# Update package repository

4. Install Java Development Kit (JDK)
Purpose: Jenkins LTS needs Java 17+.Jenkins is built on Java, so we need JDK to run it.
```bash
sudo apt install openjdk-17-jdk -y
```

- Verify Java installation
```bash
java -version
```

5. Install Jenkins
Purpose: Set up Jenkins automation server.
```bash
# Add Jenkins repository key for package authentication
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null


# Add Jenkins repository to package sources
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null


# Update package list with new repository
sudo apt update

# Install Jenkins
sudo apt-get install jenkins -y
```

6. Start and Enable Jenkins
Purpose: Ensure Jenkins runs automatically on system boot.
```bash
# Start Jenkins service
sudo systemctl start jenkins

# Enable Jenkins to start automatically on boot
sudo systemctl enable jenkins

# Check Jenkins status
sudo systemctl status jenkins
```

7. Configure Security Group
- The defaul port for jenkins is 8080. Open it in AWS Security Group for Jenkins server
Source: 0.0.0.0/0 (or your IP for security)

## Step 2 - Initial Jenkins Setup
1. Access Jenkins Web Interface
```bash
# Access via web browser
http://<Jenkins-Server-Public-IP>:8080
```

2. Retrieve Admin Password
Purpose: Jenkins generates a secure initial admin password.
```bash
# Get the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
3. Installation Steps in Web Interface
Copy the password that displays as out put and paste it into your jenkins page opened in the browser, you should get the same output below

- Install suggested plugins - provides essential functionality

- Create admin user - set up credentials for future access

- Instance configuration - use default Jenkins URL

## Step 3 - GitHub Webhook Configuration
1. Configure GitHub Repository (Tooling-website Repo)
Purpose: Allow GitHub to notify Jenkins of code changes automatically.

In your GitHub repository:

Go to Settings → Webhooks → Add webhook

Payload URL: http://<Jenkins-IP>:8080/github-webhook/

Content type: application/json

Which events: Just the push event

Active: Checked

2. Create Jenkins Freestyle Project
Purpose: Define a job that GitHub can trigger automatically.

In Jenkins Web Interface:

Click "New Item"

Item name: tooling-website-ci

Type: Freestyle project

Click OK

3. Configure Source Code Management
Purpose: Tell Jenkins where to get the source code.

## In job configuration: 

Source Code Management → Git

Repository URL: https://github.com/dimani001/tooling-website-jenkins.git

Credentials: Add your GitHub username/password or personal access token
I created a personal access token for this project

Branches to build: */main 

# The whole idea behind using jenkins is to make Continious integration very seamless , to achieve this, we have to configure jenkins to build everytime we push a new code to our repository or update an existing code.
4. Configure Build Triggers
Purpose: Automate job execution when code changes.
- # In "Build Triggers" section:
☑ GitHub hook trigger for GITScm polling

-  Configure Post-build Actions
# In "Post-build Actions" section:
Add post-build action → Archive the artifacts
Files to archive: **/*

- Save the configuration

# To test this, we made slight changes to the README.md file of our github repo. a build was launched automatically and the artifacts saved
- Click "Build Now"
- Check Console Output for success
- Verify artifacts are archived
```bash
ls /var/lib/jenkins/jobs/tooling-website-ci/builds/<build_number>/archive/
```

## Step 4 - SSH Deployment to NFS Server
- Critical Pre-requisite: Cross-AZ Configuration,a blocker that i had...my jenkins server was in eu-north1b while nfs was in eu-north 1a
Before proceeding, ensure NFS server allows Jenkins access:

1. Update NFS Exports (If servers in different AZs)
# On NFS server, edit exports to include Jenkins subnet
sudo vi /etc/exports

# Add both subnets:
/mnt/apps 172.31.16.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/apps 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)

# Apply changes
sudo exportfs -arv

2. Fixed File Permissions
```bash
# On NFS server, ensure ec2-user owns the directories
sudo chown -R ec2-user:ec2-user /mnt/apps
sudo chmod -R 775 /mnt/apps
```

3. Update Security Groups
Ensure NFS security group allows SSH from Jenkins server

Allow traffic between subnets if using different AZs

# Now back to the project, 

4.1 Install Publish Over SSH Plugin

- On the left sidebar, click on Manage Jenkins.
- In the Manage Jenkins page, click on Manage Plugins.
- In the Plugin Manager, go to the Available tab.
- Use the search box to find `Publish Over SSH Plugin
- Check the box next to Publish Over SSH Plugin.
- Click on install and wait for it to download and install
- Install without restart

4.2 Configure SSH Connection
Purpose: Set up secure connection to NFS server.
- Go to the Jenkins dashboard.
- Click on Manage Jenkins and select 'configure system'
- Scroll down to the Publish over SSH section.
- Under key, paste the content of your key pair (same keypair to access nfs server) eg steghub.key
- Add SSH Server:
1. Name: nfs-server
2. Hostname: 172.31.19.59 (NFS server private IP)
3. Username: ec2-user
4. Remote Directory: /mnt/apps

4.3 Test SSH Connection
Purpose: Verify Jenkins can connect to NFS server.

- Click Test Configuration
. Expect: Success message

- Ensure NFS server allows SSH (port 22) from Jenkins server

4.4 Configure Job for SSH Deployment
Purpose: Automate file transfer after successful build.

In your Jenkins job configuration:
- Post-build Actions → Send build artifacts over SSH
- SSH Server: Select nfs-server
- Transfers → Add Transfer:
- Source files: **
Remove prefix: 
Remote directory:
Exec command: 
# make changes again to README.md , the build should trigger immediately and deploy to the nfs server.

```bash
cat /mnt/apps/README.md
 ```

