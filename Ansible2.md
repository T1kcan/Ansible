# On the control node do following steps:
```bash
sudo yum update -y
sudo amazon-linux-extras install ansible2 -y
ansible --version
#- Optional: Run the commands below to install Python3 and Ansible. 
#sudo yum install -y python3 
#$ pip3 install --user ansible
ansible --help
# on your personal computer secure copy your private key file to your control node: 
scp -i <yourkeyfile.pem> <yourkeyfile.pem> ec2-user@<dns_or_ip>:/home/ec2-user
# on control node:
chmod 400 <yourkeyfile.pem>
cd /etc/ansible
sudo vi hosts
# Add following lines into hosts file
# Don't forget to remove ## before [webservers], [dbservers]
```
```ini
[webservers]
node1 ansible_host=<node1_ip> ansible_user=ec2-user
node2 ansible_host=<node2_ip> ansible_user=ec2-user

[dbservers]
node3 ansible_host=<node3_ip> ansible_user=ubuntu
[dbservers:vars]
ansible_ssh_private_key_file=/home/ec2-user/<pem_file>

[all:vars]
ansible_ssh_private_key_file=/home/ec2-user/<pem_file>
```
```bash
# do not forget to remove ## before group lines.
ansible localhost -m ping
# type yes for host checking
ansiblel all -m ping
ansible webservers -m ping
ansible dbservers -m ping
ansible node2 -m ping -o
ansible node1:node2 -m ping -vv
# look at Ansible official page for modules
ansible localhost -m command -a "hostname"
ansible all -m command -a hostname
ansible all -a hostname
# Command module is the default module;
# If you use command module -m can be omitted, just put -a 
# If you give more than one argument put into quotes
# No idempotent
ansible webservers -m shell -a "uptime"
ansible all -a uptime
ansible dbservers -a date
ansible all -a "df -h"
```bash
# ansible config file
cd /etc/ansible
sudo vi ansible.cfg
```
```ini
[defaults]
inventory = inventory.txt
interpreter_python = auto_silent
private_key_file = /home/ec2-user/firstkey.pem
host_key_checking = False
```
```bash
ansible all -m ping
cd ~
mkdir ansible
cd ansible
vi ansible.cfg
# add following lines:
# don't forget to change pem file
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
ansible all -m ping
ansible localhost -b -a "useradd user1"
ansible web -b -a "useradd user1"

ansible localhost -m shell -a "echo TuranCyberHub > test1"
ls
cat test1
ansible web -m copy -a "src=/home/ec2-user/ansible/test1 dest=/home/ec2-user/"
ansible web -m copy -a "src=/home/ec2-user/ansible/test1 dest=/home/ec2-user/"
# explain colors in /etc/ansible/ansible.cfg
# explain idempotence
# explain path difference
ansible web -m shell -a "ls -la && cat test1"
ansible db -m copy -a "src=/home/ec2-user/ansible/test1 dest=/home/ubuntu/"
```bash
# change directory to ~
cd ~
ansible db -m ping
# explain the problem
export ANSIBLE_CONFIG=~/ansible/ansible.cfg
ansible db -m ping
```
```bash
# let's install nginx to our servers
# shell module
ansible web -b -m shell -a "amazon-linux-extras install -y nginx1 ; systemctl start nginx ; systemctl enable nginx"
ansible db -b -m shell -a "apt update -y ; apt install -y nginx ; systemctl start nginx ; systemctl enable nginx"
# checkout public ips of worker nodes, observe web pages
ansible-doc yum
ansible web -b -m shell -a "yum remove nginx -y"
# observe web pages 
ansible web -b -m yum -a "name=nginx state=present"
ansible all -b -m package -a "name=nginx state=present"
ansible all -b -m systemd -a "name=nginx state=started"
# observe web pages again
ansible web -b -m shell -a "yum update -y"
ansible db -b -m shell -a "apt update -y; apt upgrade -y" -v