## Ansible Configuration Management (Automate Project 7–10)

# Project Overview

This project demonstrates server automation using Ansible. The aim is to manage multiple Linux servers (Web, DB, NFS, and Load Balancer) from a central Jenkins-Ansible server, automating repetitive tasks such as package installation, timezone configuration, and script execution.

- Key outcomes:

1. Learn Ansible playbooks and inventory setup.

2. Automate server configuration across multiple environments.

3. Integrate Git and Jenkins for CI/CD automation.

# Infrastructure Setup

  Server Role	         OS	         Private IP	       Public IP	          Purpose
- Jenkins-Ansible     Ubuntu	     10.0.1.214	      51.20.105.78	  CI/CD server, runs playbooks
- NFS Server	      RHEL 9	     10.0.1.178	      13.62.52.162	  Shared storage for apps/logs
- Web Server 1	      RHEL 9	     10.0.1.235	      51.20.96.151	  Web server, PHP/Apache, NFS client
- Web Server 2	      RHEL 9	     10.0.1.198	      13.48.24.0	  Web server, PHP/Apache, NFS client
- MySQL Database	  Ubuntu	     10.0.1.48	      13.62.19.177	  Database for applications
- Load Balancer	      Ubuntu	     10.0.1.127	      51.20.95.74	  Nginx Reverse Proxy

* Tip: Ensure your Jenkins-Ansible server can SSH into all remote servers on port 22 using the proper SSH keys.

# Project Structure

ansible-config-mgt/
│
├── inventory/
│   ├── dev.ini        # Development environment hosts
│   ├── staging.ini    # Staging environment hosts
│   ├── uat.ini        # UAT hosts
│   └── prod.ini       # Production hosts
│
├── playbooks/
│   └── common.yml     # Playbook for common tasks
│
└── README.md          # Project documentation


# Step 1: Install and Configure Ansible on Jenkins-Ansible Node

Rename Server: Update the Name tag in the AWS console to Jenkins-Ansible. To clearly identify the server's role in the infrastructure.The Jenkins server will double as our Ansible Control Node—the machine from which all commands and playbooks are executed.
Connect to your Jenkins-Ansible EC2 instance. 

```bash
ssh -i <your-private-key.pem> ubuntu@51.20.105.78
```

Update packages and install Ansible:
```bash
sudo apt update
# Install Ansible using the apt package manager

# Check the version to confirm successful installation
sudo apt install ansible -y
```

Verify Ansible installation:
```bash
ansible --version
```

# Step 2: Configure the GitHub Repository and Jenkins Job 

We need a central repository for our Ansible code and a Jenkins job to automatically pull and archive that code.

# 2.1 - Create the Repository

1. Create a new repository on your GitHub account named ansible-config-mgt.

2. Clone this repository to your Jenkins-Ansible server:
```Bash
git clone <YOUR_REPO_LINK>
```

# 2.2 Configure the Jenkins Freestyle Project
1. In Jenkins, create a new Freestyle project named ansible.
2. In the General section, point it to your ansible-config-mgt GitHub repository.
3. In the Build Triggers section, select GitHub hook trigger for GITScm polling.
4. In the Post-build Actions section, select Archive the artifacts and use ** (two asterisks) as the files to archive.
- Reason: This ensures every file in the repository is saved by Jenkins every time a build is triggered.

# 2.3 Configure the GitHub Webhook
1. Go to your GitHub repository's Settings - Webhooks.
2. Add a new webhook with your Jenkins-Ansible Public IP (or EIP) and port 8080:
```bash
http://<Jenkins-Ansible_Public_IP>:8080/github-webhook/.
```
3. Set Content Type to application/json.
4. Select Just the push event to trigger the build.
- Reason: This link ensures that Jenkins automatically starts a new job whenever code is pushed to the repository.

# Test the setup:
- Make changes to the README.md file in the main branch.

```bash
ls /var/lib/jenkins/jobs/ansible/builds/1/archive/
```
- Ensure builds start automatically and Jenkins saves the files in: 



# Step 3: Prepare Your Development Environment (VS Code)

Install Visual Studio Code: VS Code Download if not already set up

- Clone your repository on Jenkins-Ansible node:

```bash
git clone https://github.com/dimani001/ansible-config-mgt.git
cd ansible-config-mgt
```

- Confirm it’s a git repo
```bash
git status
```
- You should see something like: On branch main

# Create and Checkout a Feature Branch (Create a new development branch)
- Create a new branch for your Ansible work:
On your local machine (using Git Bash/VSC):

```Bash
# Create and switch to a new branch
git checkout -b feature/ansible-setup
```
Reason: Development should occur on a feature branch to isolate changes and allow for code review (Pull Request).

# Create project directories

- Inside this repository, create the main folders Ansible will use:

```bash
mkdir playbooks
mkdir inventory
```
- You can confirm they exist:
```bash
ls
```
- You should see:
```bash
.git README.md inventory/ playbooks/
```
# Create your first playbook file
- Inside the playbooks folder:

```bash
cd playbooks
touch common.yml
```

- Now confirm:
```bash
ls
```

- Output should show:
```bash
common.yml
```

# Create inventory files for different environments
- Go back one level and create these inside the inventory folder:
```bash
cd ../inventory
touch dev.ini staging.ini uat.ini prod.ini
```
Now confirm:
```bash
ls
```

- You should see:
```bash
dev.ini  staging.ini  uat.ini  prod.ini
```
- Check your work:
```bash
tree
```
- (If tree isn’t installed: sudo yum install tree -y or sudo apt install tree -y)
-At this point, your folder tree should look like this:

ansible-config-mgt/
│
├── README.md
├── inventory/
│   ├── dev.ini
│   ├── staging.ini
│   ├── uat.ini
│   └── prod.ini
└── playbooks/
    └── common.yml

- This structure organizes hosts (inventory) and tasks (playbooks), which is essential for scalable Ansible management.    

# Step 4: Configure the Inventory File (Host List) 
The inventory file tells Ansible which servers to manage and how to connect to them.

4.1 Update inventory/dev.ini
Open the inventory/dev.ini file and add the private IPs of your five servers, ensuring you use the correct SSH username for each OS.
```bash
[nfs]
10.0.1.178 ansible_ssh_user=ec2-user

[webservers]
10.0.1.235 ansible_ssh_user=ec2-user
10.0.1.198 ansible_ssh_user=ec2-user

[db]
10.0.1.48 ansible_ssh_user=ubuntu

[lb]
10.0.1.127 ansible_ssh_user=ubuntu
```
Reason: This defines the development server group structure and ensures Ansible attempts to log in with the correct OS-specific username.

# Step 5: Create the Common Playbook 
The common.yml playbook contains the configuration tasks to be applied to the servers.

5.1  Update playbooks/common.yml
Update your playbooks/common.yml with the following tasks. Note how tasks are separated by OS (RHEL uses yum, Ubuntu uses apt).

```yaml
---
   - name: update web, nfs servers
 hosts: webservers, nfs, 
 become: yes
 tasks:
   - name: ensure wireshark is at the latest version
     yum:
       name: wireshark
       state: latest
  

   - name: update LB server and db server
     hosts: lb , db
     become: yes
     tasks:
       - name: Update apt repo
         apt: 
           update_cache: yes
   
       - name: ensure wireshark is at the latest version
         apt:
           name: wireshark
           state: latest
```

Feel free to add additional tasks such as creating a directory, changing the timezone, or running shell scripts.

Here is the full common.yml playbook with all the additional tasks included:

```yaml
---
- name: update web and nfs servers
  hosts: webservers, nfs
  become: yes
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest

    - name: create a directory
      file:
        path: /path/to/directory
        state: directory
        mode: '0755'

    - name: add a file into the directory
      copy:
        content: "This is a sample file content"
        dest: /path/to/directory/samplefile.txt
        mode: '0644'

    - name: set timezone to UTC
      timezone:
        name: UTC

    - name: run a shell script
      shell: |
        #!/bin/bash
        echo "This is a shell script"
        echo "Executed on $(date)" >> /path/to/directory/script_output.txt

- name: update LB and db servers
  hosts: lb, db
  become: yes
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest

    - name: create a directory
      file:
        path: /path/to/directory
        state: directory
        mode: '0755'

    - name: add a file into the directory
      copy:
        content: "This is a sample file content"
        dest: /path/to/directory/samplefile.txt
        mode: '0644'

    - name: set timezone to UTC
      timezone:
        name: UTC

    - name: run a shell script
      shell: |
        #!/bin/bash
        echo "This is a shell script"
        echo "Executed on $(date)" >> /path/to/directory/script_output.txt
```

Reason: This playbook defines a set of repeatable, idempotent tasks (install wireshark, create directory, set timezone) that are common across your infrastructure.

# Step 6: Git Workflow and Jenkins Build 
This step ensures your code moves from your feature branch to the main branch, triggering a Jenkins build.

Add and Commit (Local Machine):

```Bash
git add .
git commit -m "feat: Add initial ansible inventory and common playbook"
# Push the Feature Branch (Local Machine)
git push origin feature/ansible-setup
```
# Create and Merge Pull Request (GitHub):
- On GitHub, create a Pull Request (PR) from feature/ansible-setup to main.

- Merge the PR (acting as the reviewer).

- pull the Latest Changes On Local Repository:
```Bash
cd ~/ansible-config-mgt
git checkout main
git pull origin main
```

- Pull Latest Changes (Jenkins-Ansible Control Node):
Jenkins Build: The merge to main should automatically trigger a Jenkins build, archiving your new files to 
```bash
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/.
```
Reason: This process enforces code review and ensures the Control Node has the latest, verified configuration files.

# Set up SSH Agent Forwarding (Crucial Step)
Ansible needs your private key to SSH into the managed nodes. SSH Agent Forwarding (ssh -A) securely provides this key from your local machine to the Control Node.

1. Start the SSH Agent (Local Machine):

```bash
eval `ssh-agent -s`
```

2. Add your Private Key (Local Machine):
```bash
ssh-add <path-to-your-private-key.pem>
```

3. SSH into the Control Node (Local Machine):
```bash
ssh -A ubuntu@<Jenkins-Ansible_Public_IP>
```
Reason: Forwarding the key (-A) allows Ansible on the Jenkins-Ansible server to use your local private key to connect to all other servers without storing the key on the Control Node.

# Run playbook on development inventory:
- Run the First Ansible Test
Finally, run the playbook from the Jenkins-Ansible Control Node (ssh -A session).

1. Run the Playbook:
```bash
cd ~/ansible-config-mgt
ansible-playbook -i inventory/dev.ini playbooks/common.yml
```
(Note: Ensure your inventory file is named .ini, not .yml, or Ansible will fail to parse it.)


# Verify tasks on servers:

Verify that wire shark is running in each of the servers by running
```bash
      wireshark --version
```



# Troubleshooting Section for Ansible Configuration Management

Even with the best setup, issues can arise when running Ansible playbooks for the first time. Below are the most common problems and how to resolve them.

1️. Server Unreachable (unreachable=1)

Symptom:
When you run the playbook:
```bash
ansible-playbook -i inventory/dev.ini playbooks/common.yml
```

You see output like:

10.0.1.178 | UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: Connection timed out"}


Cause:
The Security Group (SG) of the target server does not allow inbound SSH traffic (TCP port 22) from the Jenkins-Ansible Control Node.

Solution:

Go to the AWS EC2 console → Security Groups → select the target server’s SG.

Add an Inbound Rule:

Type: SSH

Protocol: TCP

Port Range: 22

Source: 10.0.1.214/32 (private IP of Jenkins-Ansible node)

✅ Tip: Use /32 for a single IP, ensuring only the Control Node can SSH in.

2.  Permission Denied (Permission denied (publickey))

Symptom:
You see this when attempting to run a playbook or SSH manually:

ubuntu@10.0.1.48: Permission denied (publickey)


Cause:
The SSH public key for the Jenkins-Ansible node is either missing, incorrect, or not authorized on the target server.

Solution:

Verify key exists on the target server:
```bash
cat ~/.ssh/authorized_keys
```

Add the correct key:

Copy the public key from Jenkins-Ansible node:
```bash
cat ~/.ssh/id_rsa.pub
```

Paste it into the target server’s ~/.ssh/authorized_keys for the correct user (ubuntu or ec2-user).

Check permissions:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

✅ Tip: Always use the correct OS-specific username:

RHEL servers: ec2-user

Ubuntu servers: ubuntu

3️. Inventory Parsing Errors

Symptom:
Ansible complains about the inventory file:

[ERROR]: Unable to parse /path/to/inventory/dev.ini as an inventory source


Cause:

Wrong file format: .yml vs .ini

Syntax errors in the .ini file

Solution:

Ensure your file uses INI format with correct grouping:

[nfs]
10.0.1.178 ansible_ssh_user=ec2-user

[webservers]
10.0.1.235 ansible_ssh_user=ec2-user
10.0.1.198 ansible_ssh_user=ec2-user

[db]
10.0.1.48 ansible_ssh_user=ubuntu

[lb]
10.0.1.127 ansible_ssh_user=ubuntu


Test inventory parsing:

ansible-inventory -i inventory/dev.ini --list


✅ Tip: This command outputs your inventory structure and highlights syntax errors.

4️⃣ Tasks Failing (failed=1)

Symptom:
Playbook runs but some tasks fail:

TASK [ensure wireshark is at the latest version] failed


Cause:

Package manager mismatch (yum vs apt)

Missing sudo privileges

Solution:

Check which OS is running on the server:

cat /etc/os-release


Confirm the playbook uses the correct package module:

yum for RHEL/CentOS

apt for Ubuntu/Debian

Ensure tasks use become: yes if elevated privileges are required.

5️⃣ SSH Agent Forwarding Not Working

Symptom:
Even with keys in place, Ansible cannot connect.

Cause:
The private key is not loaded into the SSH agent or forwarding is not enabled.

Solution:

Start the agent:

eval `ssh-agent -s`


Add your private key:

ssh-add <path-to-private-key.pem>


SSH with agent forwarding enabled:

ssh -A ubuntu@51.20.105.78


✅ Tip: Agent forwarding allows Jenkins-Ansible to use your local private key without copying it to the control node.

Step 7: Optional - Extend Playbook

You can repeat the workflow:

Add new tasks (install packages, run scripts, configure files).

Commit → PR → merge → build → run playbook.

Reason: This shows how one command can manage a fleet of servers efficiently.

Congratulations 🎉

You have successfully:

- Installed Ansible.

- Configured inventory and playbooks.

- Automated server tasks across multiple Linux servers.

- Integrated Git and Jenkins CI/CD for automation.