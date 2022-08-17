Launch 3 Amazon Linux 2 and 1 Ubuntu instance and name them as:
    1. Control_Node ----> (SSH PORT 22, HTTP PORT 80)
    2. Worker_Node1 ----> (SSH PORT 22, HTTP PORT 80)
    3. Worker_Node2 ----> (SSH PORT 22, HTTP PORT 80)
    4. Ubuntu ----------> (SSH PORT 22, HTTP PORT 80)

# On the control node do following steps:
```bash
sudo yum update -y
sudo amazon-linux-extras install ansible2 -y
ansible --version
# on your personal computer secure copy your private key file to your control node: 
scp -i <yourkeyfile.pem> <yourkeyfile.pem> ec2-user@<dns_or_ip>:/home/ec2-user
# on control node:
chmod 400 <yourkeyfile.pem>
cd ~
mkdir ansible && cd ansible
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
# create new hosts(inventory) file
vi inventory.txt
```
```ini
[web]
<yournode1_ip>
<yournode2_ip>

[db]
<yournode3_ip> ansible_user=ubuntu
```
```bash
# in case of change directory to ~ or to other directories export location of your ansible configuration file:
export ANSIBLE_CONFIG=~/ansible/ansible.cfg
ansible all -m ping
ansible all --list-hosts
ansible web --list-hosts
# Playbooks (Declerative Language)
# Create "playbook1.yml"
cd ansible
vi playbook1.yml
```
```yml
---
- name: Connection Test
  hosts: all
  tasks:
    - name: ping
      ping:
```
```bash
ansible-playbook playbook1.yml
ansible-playbook playbook1.yml -vvv
# explain idempotence
# create "install_httpd.yml"
ansible-doc -l | grep yum
ansible-doc yum
ansible-doc package
vi install_httpd.yml
```
```yml
---
- name: install apache
  hosts: web
  # become: true
  tasks:
    - name: install apache latest version
      yum:
        name: httpd
        state: latest
```
```bash
ansible-playbook install_httpd.yml
ansible-playbook -b install_httpd.yml
ansible web -b -m shell -a "systemctl status httpd"
ansible web -b -m shell -a "systemctl enable httpd; systemctl start httpd"
# absent

# explain become
# explain idempotence
edit install_httpd.yml
```
```yml
---
- name: install apache
  hosts: web
  become: true
  tasks:
    - name: install apache latest version
      yum:
        name: httpd
        state: present
    - systemd:
        state: started
        name: httpd
```
```bash
# create "install_apache.yml"
vi install_apache.yml
```
```yml
---
- name: install apache
  hosts: db
  become: true
  tasks:
    - name: install apache latest version
      apt:
        name: apache2
        state: latest
```
```bash
# examine official web-page ansible modules apt
# multi task in a play: 
vi install_apache.yml
```
```yml
---
- name: install apache
  hosts: db
  become: true
  tasks:
    - name: update repository
      apt:
        update_cache: yes

    - name: install apache latest version
      apt:
        name: apache2
        state: latest
```
```bash
ansible-playbook --syntax-check install_apache.yml
ansible-playbook -C install_apache.yml
# Playbooks with multiple plays:
# remove apache from servers; create "remove_apache.yml"
vi remove_apache.yml
```
```yml
---
- name: remove apache
  hosts: db
  become: true
  tasks:
    - name: remove apache2
      package:
        name: apache2
        state: absent

- name: remove httpd
  hosts: web
  become: true
  tasks:
    - name: remove httpd
      package:
        name: httpd
        state: absent
```
```bash
ansible-playbook remove_apache.yml
# create "playbook2-yml"
vi playbook2.yml
```
```yml
---
- name: Install httpd with Page Content
  hosts: web
  become: true
  tasks:
  - name: Install httpd
    package:
      name: httpd
      state: latest

  - name: Create Web Page Content
    copy:
      content: "<h1>TuranCyberHub</h1>"
      dest: /var/www/html/index.html

  - name: Start httpd
    service:
      name: httpd
      state: started
      enabled: yes
```
```bash
ansible-playbook playbook2.yml 
echo "line 1" > test.txt
# Playbooks with multiple plays & file content:
vi playbook3.yml
```
```yml
---
- name: Copying files for Centos
  hosts: web
  tasks:
   - name: Copy test file to web servers
     copy:
       src: /home/ec2-user/ansible/test.txt
       dest: /home/ec2-user/test.txt

- name: Copying files for Ubuntu
  hosts: db
  tasks:
   - name: Copy file to ubuntu server
     copy:
       src: /home/ec2-user/ansible/test.txt
       dest: /home/ubuntu/test.txt
       mode: u+rw,g-wx,o-rwx

- name: Copying files with content
  hosts: db
  tasks:
   - name: Copy using inline content
     copy:
       content: '# This file and content was created by copy module and content argument'
       dest: /home/ubuntu/content.txt

```
```bash
ansible-playbook --syntax-check playbook3.yml
ansible-playbook -C playbook3.yml
ansible-playbook playbook3.yml
# use ansible ad-hoc command to see content
ansible web -m shell -a "ls -la; cat *.txt"
ansible db -m shell -a "ls -la; cat *.txt"
# create playbook with multiple plays & item:
vi playbook4.yml
```
```yml
---
- name: Multiplay 1
  hosts: web
  become: true
  tasks:
    - name: install git, wget and tree
      yum:
        pkg: "{{ item }}"
        state: present
      loop:
        - git
        - wget
        - tree

- name: Multiplay 2
  hosts: db
  become: true
  tasks:
    - name: install mariadb & git
      apt:
        pkg: "{{ item }}"
        state: latest
      loop:
        - mariadb-server
        - git
```
```bash
ansible-playbook --syntax-check playbook4.yml
ansible-playbook -C playbook4.yml
ansible-playbook playbook4.yml
# Gather_facts
ansible all -m gather_facts
ansible all -m setup
