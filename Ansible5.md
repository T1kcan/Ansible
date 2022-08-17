Launch 3 Amazon Linux 2 and 1 Ubuntu instance and name them as:
    1. Control_Node ----> (SSH PORT 22, HTTP PORT 80)
    2. Worker_Node1 ----> (SSH PORT 22, HTTP PORT 80)
    3. Worker_Node2 ----> (SSH PORT 22, HTTP PORT 80)
    4. Ubuntu ----------> (SSH PORT 22, HTTP PORT 80)
```bash
# Tag CentOS nodes them with 
environment=web
# Tag Ubuntu node them with 
environment=db
#
cd /home/ec2-user/ansible/
mkdir ../ansible2
cp ansible.cfg ../ansible2
cd ../ansible2
# If you work in a different directory other than your ansible configuration file\
# keep in minde that Ansible has Presedence Order for configuration file:
# ansible config presedence:
## $ANSIBLE_CONFIG
## ./ansbile.cfg
## ~/.ansible.cfg
## /etc/ansible/ansible.cfg)

# change inventory line 
vi ansible.cfg
# do not forget to change your key
```
```ini
[defaults]
inventory = /home/ec2-user/ansible2/inventory_aws_ec2.yml
interpreter_python = auto_silent
private_key_file = /home/ec2-user/YOUR_KEY_FILE.pem
host_key_checking = False
command_warnings=False
```
```bash
# let's export ANSIBLE_CONFIG=~/YOUR_FOLDER_NAME/ansible.cfg
export ANSIBLE_CONFIG=~/ansible2/ansible.cfg
vi inventory_aws_ec2.yml
```
```yml
---
plugin: aws_ec2
regions:
  - "us-east-1"
keyed_groups:
  - key: tags.Name
  - key: tags.environment
compose:
  ansible_host: public_ip_address
```
```bash
# Dynamic Inventory
# Public IP, (Elastic IP) vs Private IP
ansible-inventory -i inventory_aws_ec2.yml --graph
ansible-inventory --graph
ansible all -m ping
# working with variables
# create playbook20.yml
```bash
vi playbook20.yml
```
```yml
---
- name: Working With Variables
  hosts: _web
  become: true
  vars:
    package: 
    state: 
  tasks:
  - name: Install Package
    yum:
      name: "{{ package }}"
      state: "{{ state }}"
```
```yml
ansible-playbook playbook20.yml
# observe the problem
# provide var definitions from commandline:
ansible-playbook playbook20.yml --extra-vars "package=httpd state=present"
ansible web -m shell -a "rpm -qa | grep httpd"
# group names with dynamic inventory seperator "_"
ansible-inventory --graph 
ansible _web -m shell -a "rpm -qa | grep httpd" 
ansible-playbook playbook20.yml --extra-vars "package=tree state=latest"
ansible _web -m shell -a "rpm -qa | grep tree" 
vi playbook21.yml
```yml
---
- name: Dynamic Inventory With Variables
  hosts: _web
  become: true
  vars:
    package: httpd
    state: present
    service: httpd
    service_state: started
  tasks:
  - name: Install Package
    yum:
      name: "{{ package }}"
      state: "{{ state }}"
  - name: Start Service
    service:
      name: "{{ service }}"
      state: "{{ service_state }}"
```
```bash
ansible-playbook playbook21.yml
# cmd line parameters have presedence over the variables provided inside of playbooks
ansible _web -m shell -a "systemctl status httpd"
# observe the web page
```
# Ansible Roles:

```bash
# make sure that you have updated your ansible configuration file:
vi ansible.cfg
# add following lines:
# **** don't forget to change pem file ***
```
```ini
[defaults]
inventory = /home/ec2-user/ansible2/inventory_aws_ec2.yml
interpreter_python = auto_silent
private_key_file = /home/ec2-user/YOUR_KEY_FILE.pem
host_key_checking = False
command_warnings = False
roles_path = /home/ec2-user/ansible2/roles/
```
```bash
ansible-galaxy init /home/ec2-user/ansible2/roles/apache
# create README.md file and give a description to role
vi README.md 
cd /home/ec2-user/ansible2/roles/apache
ls -la
sudo yum install tree -y
tree
# Create Apache Installation Role
cd tasks
vi main.yml
```
```yml
- name: Install Apache by roles
  yum:
    name: httpd
    state: latest

- name: custom web page
  copy:
    content: "<h1>Ansible Roles</h1>"
    dest: /var/www/html/index.html

- name: restart apache2
  service:
    name: httpd
    state: restarted
    enabled: yes
```
```bash
# change directory to /home/ec2-user/ansible2
cd /home/ec2-user/ansible2
# create a sample yml file for roles
vi role1.yml
```yml
---
- name: Install and Start apache
  hosts: _web
  become: yes
  roles:
    - apache
```
```bash
ansible-playbook role1.yml
# observe the web pages
# create role2.yml
vi role2.yml
```yml
- hosts: _db
  become: true
  roles:
    - dbservers
```
```bash
# let's create another role for dbservers
ansible-galaxy init /home/ec2-user/ansible2/roles/dbservers
# change directory to /home/ec2-user/ansible2/roles/dbservers/tasks
cd /home/ec2-user/ansible2/roles/dbservers/tasks
# edit main.yml
vi main.yml
```
```yml
- name: Update apt
  apt:
    upgrade: dist
    update_cache: yes
  when: ansible_distribution == "Ubuntu"
- name: install mariadb package
  apt:
    name:
      - mariadb-server
      - php
    state: latest
  when: ansible_distribution == "Ubuntu"
- name: start and enable httpd
  service:
    name: mariadb
    state: started
    enabled: yes
  when: ansible_distribution == "Ubuntu"
```
```bash
cd /home/ec2-user/ansible2
ansible-playbook --user ubuntu role2.yml
ansible --user ubuntu _db -b -m shell -a "apt list --installed | grep mysql"
ansible --user ubuntu _db -b -m shell -a "apt list --installed | grep php"
# Ansible Galaxy
# Basically, Ansible Collections are like Ansible Roles but much more than that. 
# In Ansible Role you have items like variables, handlers, tasks, templates, files etc. 
# But in Ansible Collection you have more items like modules, plugins, filters and Ansible Roles.
ansible-galaxy --help
# check out Ansible Galaxy web site `www.galaxy.ansible.com`
# under seach section type `nginx`
ansible-galaxy search nginx
ansible-galaxy search nginx | grep geerl
ansible-galaxy info geerlingguy.php
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install geerlingguy.nginx -p DIRECTORY
ansible-galaxy remove geerlingguy.nginx
# check out roles directory
tree
cd roles
ls
cd geerlingguy.nginx/
ls
vi README.md
cd tasks
ls
vi main.yml
cd ../vars
# go to your ansible2 ansible2 folder 
cd /home/ec2-user/ansible2
# create a new playbook for nginx installation through Ansible Collection:
vi install_nginx.yml
```
```yml
---
- name: nginx installation by galaxy
  hosts: _db
  become: true
  roles:
    - geerlingguy.nginx
```
 ansible-playbook --user ubuntu install_nginx.yml
 # check out web page
 # to list all roles under ansible:
 ansible-galaxy list

