## Ansible Dynamic Assignments (Include) and Community Roles

Intro: The goal of this task is to build on our existing ansible projects, and include dynamic roles to better understand the diffrence between a dynamic playbook and a static playbook.


# Step 1 : Updating Github Repo
In your ansible-config-mgt directory github repo , you'd have to create another branch, name it dynamic-assignments

i,.  Create a branch for dynamic assignments

```bash
cd ~/ansible-config-mgt       # Go to your local repository
git checkout -b dynamic-assignments  # Create a new branch
```
- This keeps your dynamic playbook changes separate from the main branch.

ii, Folder structure for dynamic variables

You should have:

```bash
ansible-config-mgt/
├─ dynamic-assignments/
│  ├─ env-vars.yml          # main dynamic vars file
│  └─ env-vars/             # folder for environment-specific vars
│      ├─ dev.yml
│      ├─ staging.yml
│      ├─ prod.yml
│      └─ uat.yml
```

```bash
mkdir -p dynamic-assignments/env-vars
cd dynamic-assignments
nano env-vars.yml
``

Add:

```yaml
---
vars_files:
  - "{{ playbook_dir }}/../../env-vars/dev.yml"
  - "{{ playbook_dir }}/../../env-vars/staging.yml"
  - "{{ playbook_dir }}/../../env-vars/prod.yml"
  - "{{ playbook_dir }}/../../env-vars/uat.yml"
```
- This tells Ansible where to load your environment-specific variables from.

# Create environment-specific files

```bash
cd env-vars
touch dev.yml staging.yml prod.yml uat.yml
```

Now you can edit each environment file:

```bash
nano uat.yml
# add environment-specific variables here, e.g.
```

```bash
---
load_balancer_is_required: true
enable_nginx_lb: true
enable_apache_lb: false
```

Repeat for dev.yml, prod.yml, staging.yml with their own settings.

4️.  Update site.yml
---
- name: Include dynamic variables
  hosts: all
  become: yes
  tasks:
    - include_vars: ../dynamic-assignments/env-vars.yml
      tags:
        - always

- import_playbook: ../static-assignments/uat-webservers.yml
- This makes your main playbook dynamic and able to import other playbooks.
```

## Step 2: Creating Mysql Database using roles 

note there are lots of community roles already built which we can use in our existing projects, one of these is mysql role by geerlingguy  , they can be found in this link : [Roles-Link](https://galaxy.ansible.com/home)
Inside your roles folder, create the mysql role by making use of ansible-galaxy command

1. Install MySQL role
```bash
cd ~/ansible-config-mgt/roles
ansible-galaxy install geerlingguy.mysql

#Now Rename the downloaded folder to mysql
mv geerlingguy.mysql/ mysql
```

2. Configure MySQL credentials

Edit:
```bash
nano ~/ansible-config-mgt/roles/mysql/vars/main.yml
```

Example:

```bash
mysql_root_password: admin
mysql_databases:
  - name: tooling_db
    encoding: latin1
    collation: latin1_general_ci
mysql_users:
  - name: myuser
    host: "10.0.1.0/24"
    password: password
    priv: "tooling_db.*:ALL"
```
- configure your db credentials. P.S: these credentials will be used to connect to our website later on.

3. # Create DB playbook
Create a new playbook inside static-assignments folder and call it db-servers.yml , update it with the created roles. use the code below
```bash
cd ~/ansible-config-mgt/static-assignments
nano db-servers.yml

---
- hosts: db-servers
  become: yes
  vars_files:
    - ../roles/mysql/vars/main.yml
  roles:
    - role: mysql
```

4.  Reference DB playbook in site.yml
Return to your general playbook which is the playbooks/site.yml and reference the newly created db-servers playbook, add the code below to import it into the main playbook
```bash
- import_playbook: ../static-assignments/db-servers.yml
```

5. # Commit and push changes
```bash
git add .
git commit -m "Add MySQL role and db-servers playbook"
git push origin dynamic-assignments
```
- Create a Pull Request on GitHub and merge to main.

## Step 3: Apache & Nginx Roles. Creating roles for load balancer, for this project , we will be making use of NGINX and APACHE as load balancers, so we need to create roles for them using same method as we did for mysql

1. Install Apache and Nginx roles

```bash
# Download and install roles for apache , we can get this role from same source as mysql
ansible-galaxy role install geerlingguy.apache -p ~/ansible-config-mgt/roles

# Rename the folder to apache
mv ~/ansible-config-mgt/roles/geerlingguy.apache ~/ansible-config-mgt/roles/apache

# Download and install roles for nginx
ansible-galaxy role install geerlingguy.nginx -p ~/ansible-config-mgt/roles

# Rename the folder to nginx
mv ~/ansible-config-mgt/roles/geerlingguy.nginx ~/ansible-config-mgt/roles/nginx
```

2. Add LB variables

- we cannot use both apache and nginx load balancer at the same time, it is advisable to create a condition that enables eithr one of the two, to do this ,

i. Declare a variable in roles/apache/defaults/main.yml file inside the apache role , name the variable enable_apache_lb
Apache:
```bash
nano ~/ansible-config-mgt/roles/apache/defaults/main.yml
```
```yaml
enable_apache_lb: false
```

ii. Declare a variable in roles/nginx/defaults/main.yml file inside the Nginx role , name the variable enable_nginx_lb
Nginx:
```bash
nano ~/ansible-config-mgt/roles/nginx/defaults/main.yml
```
```yaml
enable_nginx_lb: false
```

iii. declare another variable that ensures either one of the load balancer is required and set it to false.
Global LB toggle (uat.yml):
```yaml
load_balancer_is_required: false
```

3.  Load balancer playbook: Create a new playbook in static-assignments and call it loadbalancers.yml, update it with code below:

```bash
cd ~/ansible-config-mgt/static-assignments
nano loadbalancers.yml

---
- hosts: lb
  become: yes
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
```

- Now , inside your general playbook (site.yml) file, dynamically import the load balancer playbook so it can use the roles weve created
```yaml
  - import_playbook: ../static-assignments/loadbalancers.yml
    when: load_balancer_is_required
```

Like we stated earlier, to activate load balancer, and enable either of Apache or Nginx load balancer, we can achieve this by setting these in the respective environment's env-vars file.

Open the env-vars/uat.yml file and set it . here is how is how the code should be
```yaml
  ---
  load_balancer_is_required: true
  enable_nginx_lb: true
  enable_apache_lb: false
```               
To use apache, we can set the enable_apache_lb variable to true, and enable_nginx_lb to false. do the same thing for nginx if you want to enable nginx load balancer

4.  Configuring the apache and Nginx roles to work as load balancer- Configure Apache to stop Nginx.

# For Apache
in the roles/apache/tasks/main.yml file, we need to include a task that tells ansible to first check if nginx is currently running and enabled, if it is, ansible should first stop and disable nginx before proceeding to install and enable apache. this is to avoid confliction and should always free up the port 80 for the required load balancer. use the code below to achieve this :

```yaml
- name: Check if nginx is running
  ansible.builtin.service_facts:

- name: Stop and disable nginx if running
  ansible.builtin.service:
    name: nginx
    state: stopped
    enabled: no
  when: "'nginx' in services and services['nginx'].state == 'running'"
  become: yes
```

To use apache as a load balancer, we will need to allow certain apache modules that will enable the load balancer. this is the APACHE A2ENMOD
in the roles/apache/tasks/configure-debian.yml file, Create a task to install and enable the required apache a2enmod modules, use the code below :
```yaml
- name: Enable Apache modules
  ansible.builtin.shell:
    cmd: "a2enmod {{ item }}"
  loop:
    - rewrite
    - proxy
    - proxy_balancer
    - proxy_http
    - headers
    - lbmethod_bytraffic
    - lbmethod_byrequests
  notify: restart apache
  become: yes
```

- Create another task to update the apache configurations with required code block needed for the load balancer to function. use the code below :

Still in roles/apache/tasks/configure-debian.yml, add this task:
```yaml
- name: Insert load balancer configuration into Apache virtual host
  ansible.builtin.blockinfile:
    path: /etc/apache2/sites-available/000-default.conf
    block: |
      <Proxy "balancer://mycluster">
        BalancerMember http://<webserver1-ip-address>:80
        BalancerMember http://<webserver2-ip-address>:80
        ProxySet lbmethod=byrequests
      </Proxy>
      ProxyPass "/" "balancer://mycluster/"
      ProxyPassReverse "/" "balancer://mycluster/"
    marker: "# {mark} ANSIBLE MANAGED BLOCK"
    insertbefore: "</VirtualHost>"
  notify: restart apache
  become: yes
```
- Tip: Replace <webserver1-ip-address> and <webserver2-ip-address> with the private IPs of your UAT/webservers.

Once these are added:
- Save your files.
- Commit and push your changes.
- Create a pull request and merge to the main branch.

# For Nginx

In the roles/nginx/tasks/main.yml file, create a similar task like we did above to check if apache is active and enabled, if it is, it should disable and stop apache before proceeding with the tasks of installing nginx. use the code below :
roles/nginx/tasks/main.yml

```yaml
---
# Check if Apache is running and stop it before installing Nginx
- name: Check if Apache is running
  ansible.builtin.service_facts:

- name: Stop and disable Apache if it is running
  ansible.builtin.service:
    name: apache2
    state: stopped
    enabled: no
  when: "'apache2' in services and services['apache2'].state == 'running'"
  become: yes

# Include OS-specific setup tasks (if any)
- include_tasks: "setup-{{ ansible_os_family }}.yml"
```

-  roles/nginx/handlers/main.yml
In the roles/nginx/handlers/main.yml file, set nginx to always perform the tasks with sudo privileges, use the function : become: yes to achieve this

```yaml
---
# Make all handler tasks run with sudo
- name: restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
  become: yes
```
Apply become: yes to all other handler tasks in this file if they need sudo privileges.

- roles/nginx/defaults/main.yml
```yaml
---
# Enable Nginx vhosts and upstream
nginx_vhosts:
  - listen: "80"  # default: "80"
    server_name: "example.com"
    server_name_redirect: "example.com"
    root: "/var/www/html"
    index: "index.php index.html index.htm"  # default: "index.html index.htm"

    locations:
      - path: "/"
        proxy_pass: "http://myapp1"

    # Optional properties
    server_name_redirect: "www.example.com"
    error_page: ""
    access_log: ""
    error_log: ""
    extra_parameters: ""
    template: "{{ nginx_vhost_template }}"
    state: "present"

nginx_upstream:
  - name: myapp1
    servers:
      - "127.0.0.1:8080"
```

- Notes / Best Practices:
i. Do not delete existing content unless instructed; just add these sections to the correct files.
ii. The main.yml in tasks handles pre-checks and ensures no conflicts with Apache.
iii. The handlers/main.yml makes sure restart and other operations run with sudo.


The defaults/main.yml contains all the default Nginx configuration values; uncomment or modify only the sections mentioned.


1. Update roles/nginx/defaults/main.yml
Under nginx_upstream, replace the placeholder IPs with your UAT or webserver IP addresses:
```
nginx_upstreams: 
  - name: myapp1
    strategy: "ip_hash"  # alternatives: "least_conn", etc.
    keepalive: 16  # optional
    servers:
      - "<uat-server2-ip-address> weight=5"
      - "<uat-server1-ip-address> weight=5"
```
- Replace <uat-server2-ip-address> and <uat-server1-ip-address> with your real server private IPs.

2. Update inventory/uat.yml
update the inventory/uat.yml to include the neccesary details for ansible to connect to each of these servers to perform all the roles we have specified. use the code below :
Add all servers so Ansible can connect properly:
```yaml
[uat-webservers]
<server1-ipaddress> ansible_ssh_user=<ec2-username>
<server2-ipaddress> ansible_ssh_user=<ec2-username>

[lb]
<lb-instance-ip> ansible_ssh_user=<ec2-username>

[db-servers]
<db-instance-ip> ansible_ssh_user=<ec2-username>
```
- Replace placeholders with your actual EC2 IP addresses and usernames.

## Step 5 : Configure your webserver roles to install php and all its dependencies , as well as cloning your tooling website from your github repo
In the roles/webserver/tasks/main.yml , write the following tasks. use the code below :
```yaml
---
# Step 0: Create swap memory to avoid memory issues
- name: Create swap file
  ansible.builtin.command:
    cmd: fallocate -l 2G /swapfile
  args:
    creates: /swapfile
  become: yes

- name: Set swap file permissions
  ansible.builtin.file:
    path: /swapfile
    owner: root
    group: root
    mode: '0600'
  become: yes

- name: Make swap
  ansible.builtin.command:
    cmd: mkswap /swapfile
  args:
    creates: /swapfile
  become: yes

- name: Enable swap
  ansible.builtin.command:
    cmd: swapon /swapfile
  become: yes

- name: Ensure swap entry in fstab
  ansible.builtin.lineinfile:
    path: /etc/fstab
    line: '/swapfile swap swap defaults 0 0'
    state: present
  become: yes

# Step 1: Install Apache
- name: Install Apache
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "httpd"
    state: present

# Step 2: Install Git
- name: Install Git
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "git"
    state: present

# Step 3: Install EPEL release
- name: Install EPEL release
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y

# Step 4: Install dnf-utils and Remi repo
- name: Install dnf-utils and Remi repository
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm -y

# Step 5: Reset PHP module
- name: Reset PHP module
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf module reset php -y

# Step 6: Enable PHP 7.4 module
- name: Enable PHP 7.4 module
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo dnf module enable php:remi-7.4 -y

# Step 7: Install PHP and extensions
- name: Install PHP and extensions
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name:
      - php
      - php-opcache
      - php-gd
      - php-curl
      - php-mysqlnd
    state: present

# Step 8: Install MySQL client
- name: Install MySQL client
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.yum:
    name: "mysql"
    state: present

# Step 9: Start PHP-FPM
- name: Start PHP-FPM service
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.service:
    name: php-fpm
    state: started

- name: Enable PHP-FPM service
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.service:
    name: php-fpm
    enabled: true

- name: Set SELinux boolean for httpd_execmem
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.command:
    cmd: sudo setsebool -P httpd_execmem 1

# Step 10: Clone the tooling website
- name: Clone a repo
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.git:
    repo: https://github.com/dimani001/tooling.git
    dest: /var/www/html
    force: yes

- name: Copy HTML content to one level up
  remote_user: ec2-user
  become: true
  become_user: root
  command: cp -r /var/www/html/html/ /var/www/

- name: Start httpd service, if not started
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.service:
    name: httpd
    state: started

- name: Recursively remove /var/www/html/html directory
  remote_user: ec2-user
  become: true
  become_user: root
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
```
The code block above tells ansible to create swap memory to prevent OOM and UAT server crashes, install apache on the webservers , install git, install php and all its dependencies, clone the website from out github repo,as well ascopy the website files into the /var/www/html directory.

- Key thing i added, Using group_vars for LB hosts

Create a group_vars folder and a file for the lb group:
```bash
mkdir -p ~/ansible-config-mgt/group_vars
nano ~/ansible-config-mgt/group_vars/lb.yml

Add the following content:
# group_vars/lb.yml
load_balancer_is_required: true
enable_apache_lb: false
enable_nginx_lb: true
```
- This ensures that all hosts in the lb group automatically have these variables, without needing to include them manually in the playbook.

- Update your site.yml
You no longer need to include_vars just for the load balancer; it’s handled by group_vars. A clean version of site.yml:

```yaml
---
- name: Include dynamic variables for all hosts
  hosts: all
  become: yes
  tasks:
    - include_vars: ../dynamic-assignments/env-vars/uat.yml
      tags:
        - always

- import_playbook: ../static-assignments/uat-webservers.yml
- import_playbook: ../static-assignments/db-servers.yml
- import_playbook: ../static-assignments/loadbalancers.yml
  when: load_balancer_is_required
```
- load_balancer_is_required is now defined via group_vars/lb.yml, so the conditional will work correctly.



# Step 3: Clean up loadbalancers.yml
Make sure the playbook for load balancers looks like this:

```yaml
---
- hosts: lb
  become: yes
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
```
Do not redefine the variable here. It will pick up automatically from group_vars/lb.yml.



# Step 4: Check variables
Run this to confirm:
```bash
ansible -i inventory/uat.yml lb -m debug -a "var=load_balancer_is_required"
ansible -i inventory/uat.yml lb -m debug -a "var=enable_apache_lb"
ansible -i inventory/uat.yml lb -m debug -a "var=enable_nginx_lb"
```
You should see them correctly defined:
```yaml
10.0.1.127 | SUCCESS => {
    "load_balancer_is_required": true
}
```

- Why this works
- group_vars is automatically loaded for all hosts in that group.
- No need for include_vars in the playbook for LB hosts.
- The when condition in loadbalancers.yml can safely reference these variables.


# Push your branch to GitHub (if not already done):
```bash
git add .
git commit -m "Final updates for load balancer and webserver roles"
git push origin <your-branch-name>
```
Go to GitHub and create a Pull Request (PR) from your branch into main. Merge it once approved.


On your Ansible server, pull the latest changes from main to make sure your local environment is up-to-date:

```bash
cd ~/ansible-config-mgt
git pull origin main
```
After pulling, you can re-run the playbook to ensure the merged changes are applied:

```bash
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```
- This ensures your Ansible server has the latest merged version and all updates are applied cleanly.
<img width="3840" height="2160" alt="Screenshot (123)" src="https://github.com/user-attachments/assets/e6333a63-8322-4592-b6b9-1232cee13ec7" />
<img width="3840" height="2160" alt="Screenshot (124)" src="https://github.com/user-attachments/assets/22f7b90a-c3f9-451b-855f-462c7cc9c051" />
<img width="3840" height="2160" alt="Screenshot (125)" src="https://github.com/user-attachments/assets/0e00f7b6-f438-44a0-95b4-c41dbecc6a21" />

# Additional step : To be able to login to your database from the tooling website uploaded,
- Login to the uat-webservers , update the functions.php in the /var/www/html directory, input the credentials we specified in the mysql roles, in our case, admin, admin, use the mysql db server private ip as host name. save and exit.

- Ensure you can login remotely to the mysql server from the uat-server.
import the tooling-db.sql file into the database. ( check previous projects for how to do this ).

- Create a user in the tooling database, use myuser for username and password for password. exit .

- visit your uat servers ip address and attempt to login. use same password and username we inputed into the db table.
As well visit yor load balncer and ensure you can login as well
Output:
<img width="3840" height="2160" alt="Screenshot (126)" src="https://github.com/user-attachments/assets/6807c087-1979-4086-8b30-5b1ecf9e204a" />
<img width="3840" height="2160" alt="Screenshot (127)" src="https://github.com/user-attachments/assets/d098c14e-5852-4707-861a-f27f9a5472cb" />

<img width="3840" height="2160" alt="Screenshot (128)" src="https://github.com/user-attachments/assets/c6b9222e-6d47-4760-8229-666245598e1a" />
<img width="3840" height="2160" alt="Screenshot (129)" src="https://github.com/user-attachments/assets/4f9ffd93-cb0c-4180-be34-0d7c05527492" />



