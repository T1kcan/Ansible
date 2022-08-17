
Launch 3 Amazon Linux 2 and 1 Ubuntu instance and name them as:
    1. Control_Node ----> (SSH PORT 22, HTTP PORT 80)
    2. Worker_Node1 ----> (SSH PORT 22, HTTP PORT 80)
    3. Worker_Node2 ----> (SSH PORT 22, HTTP PORT 80)
    4. Ubuntu ----------> (SSH PORT 22, HTTP PORT 80)

```bash
# make sure that you have provided worker node's ip addresses in inventory file:
# give a variable name for each of the node
cd ansible
vi inventory.txt
```
```ini
[web]
node1 ansible_host=<yournode1_ip>
node2 ansible_host=<yournode2_ip>

[db]
node3 ansible_host=<yournode3_ip> ansible_user=ubuntu
```
```bash
# make sure that you have updated your ansible configuration file:
vi ansible.cfg
# add following lines:
# **** don't forget to change pem file ***
```
```ini
[defaults]
inventory = inventory.txt
interpreter_python = auto_silent
private_key_file = /home/ec2-user/<yourkeyfile.pem>
host_key_checking = False
```
```bash
# view your "ansible.cfg" file
ansible-config view
# list inventory
ansible-inventory --list
# observer inventory in tree structure
ansible-inventory --graph
# first thing to do: connection check through ping module
vi playbook10.yml
```
```yml
---
- name: Connection Test
  hosts:
  - web
  - db
  become: true
  tasks:
    - name: ping
      ping:
```
```bash
ansible-playbook playbook10.yml
# run the playbook limited to one node:
ansible-playbook playbook10.yml -l node1
# create a new playbook file as playbook11.yml
vi playbook11.yml
```
```yml
---
- name: Install Apache for CentOS Nodes
  hosts:
  - node1
  - node2
  become: true
  tasks:
  - name: install apache latest version
    yum:
      name: httpd
      state: present
  - service: name=httpd state=started enabled=true
```
```bash
ansible-playbook --syntax-check playbook11.yml
ansible-playbook -C playbook11.yml
ansible-playbook playbook11.yml
# explain list mode one the hosts and shorthand form on tasks
# create playbook12.yml
```
```yml
---
- name: Install Apache for CentOS Nodes
  hosts: [node1, node2]
  become: true
  tasks:
  - name: Upgrade all packages
    yum:
      name: '*'
      state: latest
```
```bash
# explain [] array format in hosts and '*' in packages
ansible-playbook playbook12.yml
```
# Register & Debug
```bash
# create playbook13.yml
vi playbook13.yml
```
```yml
---
- name: debug dump
  hosts: db
  become: true
  tasks:
  - name: install mariadb & wget & git
    apt:
      pkg: "{{ item }}"
      state: latest
    loop:
      - mariadb-server
      - python3-PyMySQL
      - wget
      - git
    register: db_result

  - debug: var=db_result
  ```

# Ansible Handlers
```bash
# create handlers1.yml
vi handlers1.yml
```
```yml
---
- name: handlers sample
  hosts: web
  become: true
  tasks:
  - name: Create Web Page Content
    copy:
      content: "<h1>TuranCyberHub</h1>"
      dest: /var/www/html/index.html
    notify:
    - restart apache

  handlers:
  - name: restart apache
    service:
      name: httpd
      state: restarted
```
```bash
ansible-playbook handlers1.yml
# create handlers1.yml
vi handlers2.yml
```
```yml
---
- name: handlers sample 2
  hosts: web
  become: true
  tasks:
  - name: Replace Web Page Content
    lineinfile:
      path: /var/www/html/index.html
      regexp: " "<h1>TuranCyberHub</h1>"
      line: "<h1>TuranCyberHub <br>
        Handler Sample</h1>"
    notify:
    - restart apache
  
- name: db server configuration
  hosts: db
  become: true
  tasks:
  - name: Install mariadb and PyMySQL
    package:
      name:
      - mariadb-server
      - python3-PyMySQL
      state: latest
    notify: restart mariadb

  handlers:
  - name: restart apache
    service:
      name: httpd
      state: restarted
  handlers:
  - name: restart mariadb
    systemd:
      name: mariadb
      state: restarted
```

# facts and magic variables
```bash
ansible all -m gather_facts
https://docs.ansible.com/ansible/latest/collections/ansible/builtin/gather_facts_module.html
ansible all -m gather_facts | grep ansible
ansible all -m gather_facts | grep os_family
ansible all -m gather_facts | grep ansible_distribution
ansible node1 -m gather_facts | grep ansible_
ansible all -m setup
https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html
ansible node1 -m setup
ansible all -m gather_facts | grep ansible
ansible all -m setup | grep ansible_os
ansible node1:node2 -m setup | grep ansible_os_family
ansible node3 -m setup | grep ansible_distribution_version
# create facts.yml
vi facts.yml
```
```yml
---
- name: Gather Facts
  hosts: all
  tasks:
    - name: print out all available facts
      debug:
        var: ansible_facts
```
```bash
# using facts in variables
vi nodes_info.yml
```
```yml
---
- hosts: all
  tasks:
  - name: show IP address
    debug:
      msg: >
       This {{ inventory_hostname }} uses IP address of {{ ansible_default_ipv4.address }} and has {{ ansible_nodename }} name

```
```bash
ansible-playbook nodes_info.yml
# create user.yml
vi user.yml
```
```yml
---
- name: Create users
  hosts: "*"
  tasks:
    - user:
        name: "{{ item }}"
        state: present
      loop:
        - David
        - John
        - Walker
      when: ansible_os_family == "RedHat"

    - user:
        name: "{{ item }}"
        state: present
      loop:
        - Sam
        - Zac
      when: ansible_distribution == "Ubuntu"

    - user:
        name: "{{ item }}"
        state: present
      loop:
        - Mariam
        - Jane
      when: ansible_os_family == "Debian" or ansible_distribution_version == "2"
```
```bash
# creating user requires previliges **become**
ansible-playbook -b user.yml
ansible all -m shell -a "tail -7 /etc/passwd"
```
# Secret & Ansible Vault
```bash
ansible-vault create secret.yml
```
```yml
username: Tim
password: UxUx11
```
```bash
ansible-vault edit secret.yml
cat secret.yml
ansible-vault view secret.yml
# create a playbook called create_user.yml
vi create_user.yml
```
```yml
---
- name: New User
  hosts: all
  become: true
  vars_files:
    - secret.yml
  tasks:
    - name:  creating the new user
      user:
        name: "{{ username }}"
        password: "{{ password }}"
```
```bash
ansible-playbook create_user.yml
# ERROR! Attempting to decrypt but no vault secrets found
ansible-playbook --ask-vault-pass create_user.yml
ansible all -b -m command -a "tail /etc/shadow"
```
# Dynamic Inventory
```
AWS EC2 Plugin
https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html
```bash
- go to your AWS Management Consol and select Identity and Access Management (IAM);select the "Roles"

- click on the  "create role" button then select "AWS service" as Trusted entity type; "EC2" as Use case and clik next,

- Seach "AmazonEC2FullAccess" on the filter section (paste AmazonEC2FullAccess and hit enter)

- Select "AmazonEC2FullAccess" and click next,

- give a name for the role with "AmazonEC2FullAccess" and create

- go to EC2 instance Dashboard, and select the Control_Node instance for Ansible

- select actions -> security -> modify IAM role

- select the role thay you have jsut created for EC2 full access and save it.

- install "boto3 and botocore"

$ sudo yum install pip -y
$ pip install --user boto3 botocore

# Create another file named ```inventory_aws_ec2.yml``` in the project directory.
vi inventory_aws_ec2.yml
```
```yml
---
plugin: aws_ec2
regions:
  - "us-east-1"
keyed_groups:
  - key: tags.Name
compose:
  ansible_host: public_ip_address
```
```bash
# observe the dynamic inventory with ad-hoc command
ansible-inventory -i inventory_aws_ec2.yml --graph
# output:
@all:
  |--@_Control_Node:
  |  |--ec2-3-91-84-216.compute-1.amazonaws.com
  |--@_Ubuntu:
  |  |--ec2-54-163-34-203.compute-1.amazonaws.com
  |--@aws_ec2:
  |  |--ec2-3-91-84-216.compute-1.amazonaws.com
  |  |--ec2-52-204-56-96.compute-1.amazonaws.com
  |  |--ec2-54-163-34-203.compute-1.amazonaws.com
  |  |--ec2-54-210-225-117.compute-1.amazonaws.com
  |  |--ip-172-31-22-174.ec2.internal
  |--@ungrouped:

# edit ansible.cfg and change the static inventory to dynamic one with placing 'inventory=/home/ec2-user/ansible/inventory_aws_ec2.yml' into configuration file:
vi ansible.cfg
```
```ini
[defaults]
inventory = /home/ec2-user/ansible/inventory_aws_ec2.yml
interpreter_python = auto_silent
private_key_file = /home/ec2-user/<yourkeyfile.pem>
host_key_checking = False
```
```bash
# running with dynamic inventory
# create user.yml
vi user_delete.yml
```
```yml
---
- name: create a user using a variable
  hosts: all
  become: true
  vars:
    user: Zac

  tasks:
    - name: removing the user {{ user }}
      user:
        name: "{{ user }}"
        state: absent
        remove: yes
```
``bash
ansible-playbook user_delete.yml
$ ansible all -a "tail -2 /etc/passwd"
