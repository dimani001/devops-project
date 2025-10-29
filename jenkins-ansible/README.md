## Ansible Configuration Management (Automate Project 7–10)

# Project Overview

This project demonstrates server automation using Ansible. The aim is to manage multiple Linux servers (Web, DB, NFS, and Load Balancer) from a central Jenkins-Ansible server, automating repetitive tasks such as package installation, timezone configuration, and script execution.

- Key outcomes:

1. Learn Ansible playbooks and inventory setup.

2. Automate server configuration across multiple environments.

3. Integrate Git and Jenkins for CI/CD automation.

# Infrastructure Setup

  Server Role	         OS	         Private IP	       Public IP	          Purpose
- Jenkins-Ansible     Ubuntu	     10.0.1.214	        51.20.105.78	  CI/CD server, runs playbooks
- NFS Server	        RHEL 9	       10.0.1.178	      13.62.52.162	  Shared storage for apps/logs
- Web Server 1	      RHEL 9	     10.0.1.235	        51.20.96.151	  Web server, PHP/Apache, NFS client
- Web Server 2	      RHEL 9	     10.0.1.198	        13.48.24.0	    Web server, PHP/Apache, NFS client
- MySQL Database	    Ubuntu	       10.0.1.48	      13.62.19.177	  Database for applications
- Load Balancer	      Ubuntu	     10.0.1.127	        51.20.95.74	    Nginx Reverse Proxy

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
<img width="3840" height="2160" alt="Screenshot (27)" src="https://github.com/user-attachments/assets/55708d18-f2be-40c4-a78a-2765f951573c" />

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
<img width="3840" height="2160" alt="Screenshot (28)" src="https://github.com/user-attachments/assets/47e4d5e4-77df-477c-9489-148337599b55" />

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
<img width="3840" height="2160" alt="Screenshot (30)" src="https://github.com/user-attachments/assets/40fa3dd2-85fe-488c-ac06-01a89fe24d2e" />

# 2.3 Configure the GitHub Webhook
1. Go to your GitHub repository's Settings - Webhooks.
2. Add a new webhook with your Jenkins-Ansible Public IP (or EIP) and port 8080:
```bash
http://<Jenkins-Ansible_Public_IP>:8080/github-webhook/.
```
<img width="3840" height="2160" alt="Screenshot (31)" src="https://github.com/user-attachments/assets/c7f1bb00-84cc-4e4b-a4ea-7d4b2c60908b" />

3. Set Content Type to application/json.
4. Select Just the push event to trigger the build.
- Reason: This link ensures that Jenkins automatically starts a new job whenever code is pushed to the repository.
<img width="3840" height="2160" alt="Screenshot (32)" src="https://github.com/user-attachments/assets/3b47e831-7f8b-4f7b-925a-b1454bd734e4" />

# Test the setup:
- Make changes to the README.md file in the main branch.

```bash
ls /var/lib/jenkins/jobs/ansible/builds/1/archive/
```
- Ensure builds start automatically and Jenkins saves the files in: 

<img width="3840" height="2160" alt="Screenshot (34)" src="https://github.com/user-attachments/assets/f1fafdcc-5504-4a26-af84-0da59d9fff28" />


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
<img width="3840" height="2160" alt="Screenshot (35)" src="https://github.com/user-attachments/assets/9f9554a5-93b3-441f-bc37-b5ac02a9f3ae" />

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
<img width="3840" height="2160" alt="Screenshot (40)" src="https://github.com/user-attachments/assets/7c1680f7-ad2d-4996-8c3e-8bf529de1438" />

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
<img width="3840" height="2160" alt="Screenshot (36)" src="https://github.com/user-attachments/assets/87f7868b-192a-4365-9a15-c5d82ea81cd9" />

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
<img width="3840" height="2160" alt="Screenshot (37)" src="https://github.com/user-attachments/assets/ea89db9b-0ff9-4934-a8cf-76d3137306ab" />

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
<img width="3840" height="2160" alt="Screenshot (39)" src="https://github.com/user-attachments/assets/91b94f3e-a675-4b21-b7bf-de037dd62097" />

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
<img width="3840" height="2160" alt="Screenshot (44)" src="https://github.com/user-attachments/assets/7441be14-0509-4a2f-bb3c-f7bd58fdcda6" />

# Step 6: Git Workflow and Jenkins Build 
This step ensures your code moves from your feature branch to the main branch, triggering a Jenkins build.

Add and Commit (Local Machine):

```Bash
git add .
git commit -m "feat: Add initial ansible inventory and common playbook"
# Push the Feature Branch (Local Machine)
git push origin feature/ansible-setup
```
<img width="3840" height="2160" alt="Screenshot (45)" src="https://github.com/user-attachments/assets/bbfbd35c-a30a-4056-bb81-66e5f25abb5e" />

# Create and Merge Pull Request (GitHub):
- On GitHub, create a Pull Request (PR) from feature/ansible-setup to main.
<img width="3840" height="6918" alt="Comparing-main-feature-ansible-setup-·-dimani001-ansible-config-mgt-10-29-2025_07_30_AM" src="https://github.com/user-attachments/assets/52eabb0e-d8d9-4adc-929d-7026b57aa153" />

- Merge the PR (acting as the reviewer).
<img width="3840" height="5686" alt="Comparing-main-feature-ansible-setup-·-dimani001-ansible-config-mgt-10-29-2025_07_28_AM" src="https://github.com/user-attachments/assets/ab042ed3-4c60-45e1-baa6-c7bcd877fe6f" />

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
<img width="3840" height="2160" alt="Screenshot (38)" src="https://github.com/user-attachments/assets/25b70baf-430f-4c12-af4a-ef91b816d15f" />

# Run playbook on development inventory:
- Run the First Ansible Test
Finally, run the playbook from the Jenkins-Ansible Control Node (ssh -A session).

1. Run the Playbook:
```bash
cd ~/ansible-config-mgt
ansible-playbook -i inventory/dev.ini playbooks/common.yml
```
(Note: Ensure your inventory file is named .ini, not .yml, or Ansible will fail to parse it.)

<img width="3840" height="2160" alt="Screenshot (52)" src="https://github.com/user-attachments/assets/b16d91f5-3bd0-4964-8e5b-da6552b65f1f" />
<img width="3840" height="2160" alt="Screenshot (55)" src="https://github.com/user-attachments/assets/3cef0b4e-a57e-4447-9b3e-de33cc91c824" />
<img width="3840" height="2160" alt="Screenshot (56)" src="https://github.com/user-attachments/assets/6df7bd5b-3a7c-4f12-acfa-b50deaf16f6e" />

# Verify tasks on servers:

Verify that wire shark is running in each of the servers by running
```bash
      wireshark --version
```
- # NFS SERVER
<img width="3840" height="2160" alt="Screenshot (57)" src="https://github.com/user-attachments/assets/ea51641a-6559-49de-aa75-cc2086548ac9" />

- # WEB_SERVER 1
<img width="38<img width="3840" height="2160" alt="Screenshot (59)" src="https://github.com/user-attachments/assets/2fb553c9-180d-4102-9500-88d9618123b6" />

- # DB SERVER
<img width="3840" height="2160" alt="Screenshot (58)" src="https://github.com/user-attachments/assets/668364d4-45c7-49e8-8ca1-501faa38569b" />

- # WEB_SERVER 2
<img width="3840" height="2160" alt="Screenshot (60)" src="https://github.com/user-attachments/assets/707916c7-5846-426d-9826-db546da8fa3e" />

- # LB_SERVER
<img width="3840" height="2160" alt="Screenshot (61)" src="https://github.com/user-attachments/assets/34ac3154-055f-4383-923e-4346a43c68d4" />



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
