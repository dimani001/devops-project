## Ansible Refactoring & Static Assignments (Imports and Roles) - 102

This document serves as the complete technical guide for setting up a robust Continuous Deployment (CD) pipeline. This pipeline uses Jenkins (running on an Ubuntu Control Node) to execute Ansible playbooks, deploying a sample web application to two UAT (User Acceptance Testing) Web Servers running RHEL/Amazon Linux.

# AWS Infrastructure and SSH Key Management

The foundation requires three EC2 instances and proper network configuration.

# 1.1 Server Roles and Operating Systems

- Control Node (jenkins-ansible Server): Ubuntu 20.04+

- UAT Web Server 1: RHEL / Amazon Linux 2 (Target for deployment)

- UAT Web Server 2: RHEL / Amazon Linux 2 (Target for deployment)

# Re-Establish SSH Agent Forwarding (On Local PC)

This step gives your Control Node the ability to SSH into the UAT servers without storing your private key.

# Start the Agent: Open your local terminal (Git Bash) 
```bash
eval `ssh-agent -s`
```

- Add Your Key: Add your private key to the agent (replace the path)
```bash
ssh-add <path-to-your-key.pem>
```

- SSH to Control Node: Connect to the Jenkins server, ensuring you use the -A flag
```bash
ssh -A ubuntu@13.53.83.71
```
Reason: The -A flag is essential; it forwards your key, which Ansible needs.

2. # Update Security Group (SG) Rules (AWS Console)

This step prevents the "Connection timed out" error for your new UAT servers.

Go to the AWS Console - Security Groups, find the Security Group attached to Web1-UAT and Web2-UAT. In the Inbound Rules, add a rule:
Type: SSHSource: 10.0.1.214/32 (The private IP of your Jenkins-Ansible server).
- Reason: The Control Node (10.0.1.214) must be able to initiate SSH (Port 22) connections to the new UAT servers.

3. # Pull Latest Code (On Control Node)

Inside your Jenkins-Ansible server, ensure your local repository copy is up-to-date.
- Navigate and Pull
```bash
cd /home/ubuntu/ansible-config-mgt/
git pull origin main
```
Reason: We need the final, working code from Project 11 before refactoring.

## Step 1: Jenkins Job Enhancement
_ This is the setup according to the study material

# Initial Setup
Before we begin, we need to optimize our Jenkins setup to handle artifacts more efficiently:

Create a directory on your Jenkins-Ansible server (ec2-instance) to store all artifacts after each build:
```bash
  sudo mkdir /home/ubuntu/ansible-config-artifact
  ```
*Change Permission*  to this new directory so that jenkins can save the archives to it

```bash
  sudo chmod -R 0777 /home/ubuntu/ansible-config-artifact
  ```
#  Configure save_artifacts Jenkins Job

- Go to Jenkins web console -> `Manage Jenkins` -> `Manage Plugins`- search for `copy Artifacts` and then install it
* Plugin: Install the `Copy Artifact` Plugin.

- Job Creation: Create a new Freestyle project and call it `save_artifacts`

- Configure this project to be triggered by the completion of the previous ansible project(ansible), to do this ,
* go to `save_artifacts` project, > `configure` > `general tab` , Click to check the `discard old builds` option, on `strategy` ,set it to *log rotation*, also set the `max number of builds` to ""2"" . this is done in order to keep space on the server

- Set `SCM` to *none*,

- Under `build triggers`, select `build after other projects are built` , and under `projects to watch`, input the name of your previous ansible project, in this case : "ansible"

- Set up `build steps` select "copy artifacts from another project" , in the drop down , input the name of the project. as is this case : "ansible". under `which build`, set it to "latest successful build" , and under artifacts to copy , use `**` to select all


- Save the configurations and make changes to your `README.md` file in your github `ansible-config-mgt` repo and ensure the build triggers upon changes made on the file, also confirm the new save artifact job build also triggers from the completion of the ansible build


However, while executing this project, The primary ansible job failed because the jenkins user lacked read/write access to the Git repository checked out by the ubuntu user (/home/ubuntu/ansible-config-mgt).

Rationale: The jenkins user must own the workspace to pull updates and execute commands, resolving "Permission Denied" errors.

Action (On Control Node):

```bash
cd /home/ubuntu/
# Recursively change ownership of the project directory to the jenkins user
sudo chown -R jenkins:jenkins /home/ubuntu/ansible-config-mgt
```
2. Secure Artifact Storage Setup
A dedicated directory (/home/ubuntu/ansible-config-artifact) was created to store build artifacts, secured using group permissions instead of insecure 0777 permissions.

Rationale for Deviation: Using 0777 creates a major security vulnerability. The group-based approach (770 permissions) adheres to the principle of least privilege, granting access only to the authorized jenkins service via a shared devops group.

Implementation Steps (On Control Node):

```bash
# 1. Create directory
sudo mkdir /home/ubuntu/ansible-config-artifact

# 2. Add users to 'devops' group
sudo usermod -aG devops jenkins
sudo usermod -aG devops ubuntu

# 3. Set ownership to ubuntu and 'devops' group
sudo chown -R ubuntu:devops /home/ubuntu/ansible-config-artifact

# 4. Apply secure 770 permissions and SGID bit (g+s)
sudo chmod -R 770 /home/ubuntu/ansible-config-artifact
sudo chmod g+s /home/ubuntu/ansible-config-artifact

# 5. Restart Jenkins
sudo systemctl restart jenkins
```

Once you're done with the first step,
# Step 2 . Refactor Ansible Code by Importing Other Playbooks
- Preparation and Branching

Before initiating any refactoring, it is standard practice to ensure the local repository is up-to-date and to perform all changes on a dedicated branch.

Rationale: Working on a feature branch (refactor) isolates changes from the main production branch (master/main). This prevents accidental breakage and facilitates peer review via a Pull Request before merging the validated code.

# Action (On Local PC/VSC): Pull the latest code and create the new branch.

```bash
cd ~/ansible-config-mgt
git checkout main
git pull origin main
git checkout -b refactor
```

- Ceate a site.yml file in the playbooks folder. This will serve as the entry point to all configurations.
Inside the playbooks directory:
```bash
cd playbooks
touch site.yml
```
In site.yml, import common.yml:

```yml
---
- import_playbook: ../static-assignments/common.yml
```
-  site.yml now acts as the parent playbook.

. Organize Folder Structure
- Create a static-assignments folder at the root of the repository for organizing child playbooks and move common.yml into it:

```bash
cd ~/ansible-config-mgt
mkdir static-assignments
mv playbooks/common.yml static-assignments/common.yml
```

- Still under (On Local PC/VSC), Push to your github repo and create a pull request, carefully check your codes and merge into your main branch

- Access your jenkins-ansible server via ssh agent and navigate to ansible-config-artifact directory and run the playbook command against the dev environment

```bash
cd /home/ubuntu/ansible-config-mgt/
ansible-playbook -i inventory/dev.ini playbooks/site.yml
```

- Create another playbook `common-del.yml` under `static-assignments` for deleting Wireshark. 

```bash
cd static-assignments
touch common-del.yml
```

inside it, place the following code in it and save
```yaml
              ---
              - name: update web, nfs servers
                hosts: webservers, nfs
                remote_user: ec2-user
                become: yes
                become_user: root
                tasks:
                - name: delete wireshark
                  yum:
                    name: wireshark
                    state: removed
              
              - name: update LB and db servers
                hosts: lb, db
                remote_user: ubuntu
                become: yes
                become_user: root
                tasks:
                - name: delete wireshark
                  apt:
                    name: wireshark
                    state: absent
                    autoremove: yes
                    purge: yes
                    autoclean: yes
                  - 
```

- Update and Run site.yml

* Replace common.yml with common-del.yml in playbooks/site.yml (On Local PC/VSC)::

```bash
---
- import_playbook: ../static-assignments/common-del.yml
```

- Next, Push to git

Run Action (On Control Node)::

```bash
cd /home/ubuntu/ansible-config-mgt/
git pull origin main
ansible-playbook -i inventory/dev.ini playbooks/site.yml
```

- Confirm Wireshark was deleted:
```bash
wireshark --version
# Output: bash: wireshark: command not found
```

# Step 3. Configure UAT Webservers with a Role 'Webserver'
Overview
After setting up a clean development environment, we will now configure the two RHEL (9) new Web Servers as UAT environments. This step involves using Ansible roles to ensure our configurations are reusable and maintainable.

- Initial Setup

1. Launch two EC2 instances using the RHEL 9 image. Name them Web1-UAT and Web2-UAT.
* Remember to stop EC2 instances that you are not using to avoid unnecessary charges.

# Role Creation
You can create the Ansible role using either the ansible-galaxy command or manually: However , since we use github as version control, it is advisable to manually create the roles directory and the chikd fikes in it, so in your vs code terminal, run the following code to create the roles directory and child files

- From your project root:

```bash
cd ~/ansible-config-mgt
mkdir roles
mkdir webserver
cd webserver
touch README.md
mkdir defaults, handlers, meta, tasks, templates
```

- Navigate into each of the created directories and create a `main.yml` file in each directory

2. Update Inventory
Edit inventory/uat.yml with your UAT server IPs:

[uat-webservers]
<Web1-UAT-Private-IP> ansible_ssh_user='ec2-user'
<Web2-UAT-Private-IP> ansible_ssh_user='ec2-user'

- Inventory ready for UAT environment.

3. Configure Roles Path
Configure ansible.cfg : Ensure that your ansible.cfg file (usually located at /etc/ansible/ansible.cfg) has the roles_path uncommented and correctly set:
- In /etc/ansible/ansible.cfg, ensure this line is active:
```bash
roles_path = /home/ubuntu/ansible-config-mgt/roles
```
- Ansible now knows where to find custom roles.

4. Define Webserver Tasks
- Edit `roles/webserver/tasks/main.yml `:
Navigate to the tasks directory of your webserver role and add tasks to install Apache, clone the GitHub repository, and configure the server:

Open the tasks/main.yml file and update it with the code to perform the above tasks i mentioned,
```bash
                      ---
                - name: install apache
                  become: true
                  ansible.builtin.yum:
                    name: "httpd"
                    state: present
                
                - name: install git
                  become: true
                  ansible.builtin.yum:
                    name: "git"
                    state: present
                
                - name: clone a repo
                  become: true
                  ansible.builtin.git:
                    repo: https://github.com/dimani001/tooling2.git
                    dest: /var/www/html
                    force: yes
                
                - name: copy html content to one level up
                  become: true
                  command: cp -r /var/www/html/html/ /var/www/
                
                - name: Start service httpd, if not started
                  become: true
                  ansible.builtin.service:
                    name: httpd
                    state: started
                
                - name: recursively remove /var/www/html/html/ directory
                  become: true
                  ansible.builtin.file:
                    path: /var/www/html/html
                    state: absent
 ```
NB- These tasks will ensure that your UAT servers are configured with Apache serving content cloned from your specified GitHub repository.

6. Reference 'Webserver' Role in Playbook
Within the static-assignments folder, create a new playbook file named uat-webservers.yml. This playbook will specifically configure your UAT web servers by utilizing the 'Webserver' role. update the file with the code below to reference the webserver role
```bash
                    ---
                   - hosts: uat-webservers
                     roles:
                        - webserver
```

Make sure to add a reference to this new playbook in site.yml, alongside your existing playbook settings. By doing this, site.yml will continue to serve as the central point for all your Ansible configurations. therefore update the site.yml file with the code below :

```bash
                  ---
                 ##- hosts: all
                 ##- import_playbook: ../static-assignments/common.yml
                 - hosts: uat-webservers
                 - import_playbook: ../static-assignments/uat-webservers.yml
```


# Step 5: Commit & Test
- Commit your changes to your Git repository.
- Create a Pull Request and merge it into the main branch

- Access your ansible-jenkins server and navigate to the ansible-config-artifact directory

Run the playbook command
```bash
  cd /home/ubuntu/ansible-config-artifact
  ansible-playbook -i inventory/uat.yml playbooks/site.yml
  ```

Please check that your UAT Web servers are set up correctly. You should be able to visit your servers using any web browser


conclusion :
Throughout this project, we've tackled a variety of improvements and optimizations that really showcase the strength of incorporating advanced DevOps tools and practices. From tweaking our Jenkins configurations to taking advantage of Ansible for more powerful and scalable infrastructure management, every step has played a part in making our processes more streamlined and effective.
