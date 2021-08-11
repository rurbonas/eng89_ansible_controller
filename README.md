![](ANSVagrant-1.png)

# Ansible controller and agent nodes set up guide
- Clone this repo and run `vagrant up`
- `(double check syntax/intendation)`

## We will use 18.04 ubuntu for ansible controller and agent nodes set up 
### Please ensure to refer back to your vagrant documentation

- **You may need to reinstall plugins or dependencies required depending on the OS you are using.**

```vagrant 
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what

# MULTI SERVER/VMs environment 
#
Vagrant.configure("2") do |config|

# creating first VM called web  
  config.vm.define "web" do |web|
    
    web.vm.box = "bento/ubuntu-18.04"
   # downloading ubuntu 18.04 image

    web.vm.hostname = 'web'
    # assigning host name to the VM
    
    web.vm.network :private_network, ip: "192.168.33.10"
    #   assigning private IP
    
    config.hostsupdater.aliases = ["development.web"]
    # creating a link called development.web so we can access web page with this link instread of an IP   
        
  end
  
# creating second VM called db
  config.vm.define "db" do |db|
    
    db.vm.box = "bento/ubuntu-18.04"
    
    db.vm.hostname = 'db'
    
    db.vm.network :private_network, ip: "192.168.33.11"
    
    config.hostsupdater.aliases = ["development.db"]     
  end

 # creating are Ansible controller
  config.vm.define "controller" do |controller|
    
    controller.vm.box = "bento/ubuntu-18.04"
    
    controller.vm.hostname = 'controller'
    
    controller.vm.network :private_network, ip: "192.168.33.12"
    
    config.hostsupdater.aliases = ["development.controller"] 
    
  end

end
```

# Why Ansible?

    It is the fastest growing python-based open source IT automation tool that can be used to configure/manage systems, deploy applications and provision infrastructure on numerous cloud platforms.

Benefits :

    Save time
    Open source
    Makes configuration management predictable
    Cost effective

Why use it:

    Ansible is angentless because we only need to have ansible installed on the controller.
    We connect using SSH - this also adds to its simplicity


![](ansdiagram.JPG)

# Working with Ansible
SSH into the controller with `vagrant ssh controller`
We'll update/install dependecies:
- `sudo apt-get update -y`
- `sudo apt-get install software-properties-common`
- `sudo apt-add-repository ppa:ansible/ansible`
- `sudo apt-get update`
- `sudo apt-get install ansible`

Now check Hosts:
- `cd /etc/ansible/`
- `sudo apt-get install tree`
- `tree`
- `ansible all -m ping`
Add hosts:
- `sudo nano hosts`
- Add 
```
[web]
192.168.33.10 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant

[db]
192.168.33.11 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant
 ```
To acces`ssh vagrant@192.168.33.10`
Check all the servers:
- `ansible all -a "uname -a"` # Check names
- `ansible all -a "date"`     # Check dates
- `ansible all -a "free -m"`  # Check memory

## Make a YAML Playbook file (TAB doesn't work - yet indentation matters):
### Install nginx in web
- `sudo nano nginx_playbook.yml`
- Add:
```YAML
# A playbook to install and set up Nginx
# A YAML file need to start with 3 dashes

---
# Name of the hosts - needs to be defined in hosts file
- hosts: web

# Find the facts about the hosts
  gather_facts: yes

# We need admin access
  become: true

# Add instructions using tasks module in ansible
  tasks:
  - name: install nginx

# Install nginx
    apt: pkg=nginx state=present update_cache=yes

# Ensure its running/active
    notify:
    - restart nginx
  - name: Allow all access to tcp port 80
    ufw:
      rule: allow
      port: '80'
      proto: tcp

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted

# Update cache
# Restart nginx if needed like reverse proxy
```
- `ansible-playbook nginx_playbook.yml`
- `ansible web -m shell -a "systemctl status nginx"`

Copy app from local to VM:
- `vagrant plugin install vagrant-scp`
- `vagrant scp C:/Users/Urbon/VagrantProjects/eng89_01/app app`

### Installing mongodb
- `sudo nano mongodb_playbook.yml`
- Add:
```YAML
# This is a YAML file to install nginx onto oue web VM using YAML
---

- hosts: db

  gather_facts: yes

  become: true

  tasks:
  - name: install mongodb
    apt: pkg=mongodb state=present

  - name: Remove mongodb file (delete file)
    file:
      path: /etc/mongodb.conf
      state: absent

  - name: Touch a file, using symbolic modes to set the permissions (equivalent to 0644)
    file:
      path: /etc/mongodb.conf
      state: touch
      mode: u=rw,g=r,o=r


  - name: Insert multiple lines and Backup
    blockinfile:
      path: /etc/mongodb.conf
      backup: yes
      block: |
        "storage:
          dbPath: /var/lib/mongodb
          journal:
            enabled: true
        systemLog:
          destination: file
          logAppend: true
          path: /var/log/mongodb/mongod.log
        net:
          port: 27017
          bindIp: 0.0.0.0"

```

### Installing node and running app
- `sudo nano node_run.yml`
- Add:
```YAML

# This is a playbook to install and set up Nginx in our web server (192.168.33.10)
# This playbook is written in YAML and YAML starts with three dashes (front matter)

---
# name of the hosts - hosts is to define the name of your host of all
- hosts: web

# find the facts about the host
  gather_facts: yes

# admin access
  become: true

# instructions using task module in ansible
  tasks:
  - name: Install Nginx

# install nginx
    apt: pkg=nginx state=present update_cache=yes

-
  name: "installing nodejs"
  hosts: web
  become: true
  tasks:
    - name: "add nodejs"
      apt_key:
        url: https://deb.nodesource.com/gpgkey/nodesource.gpg.key
        state: present
    - name: add repo
      apt_repository:
        repo: deb https://deb.nodesource.com/node_13.x bionic main
        update_cache: yes
    - name: "installing nodejs"
      apt:
        update_cache: yes
        name: nodejs
        state: present
-
  name: "reverse proxy"
  hosts: web
  become: true
  tasks:
    - name: "delete current default"
      file:
        path: /etc/nginx/sites-available/default
        state: absent
    - name: "create file"
      file:
        path: /etc/nginx/sites-available/default
        state: touch
        mode: 0644
    - name: "change default file"
      blockinfile:
        path: /etc/nginx/sites-available/default
        block: |
          server{
            listen 80;
            server_name _;
            location / {
            proxy_pass http://192.168.33.10:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            }
          }
      notify:
        - Restart nginx
    - name: update npm
      command: npm install pm2 -g -y
  handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
- name: "starting app"
  hosts: web
  become: true
  tasks:
   - name: "adding DB_HOST"
     blockinfile:
       path: .bashrc
       block: |
         DB_HOST=192.168.33.11:27017/posts
   - name: "starting node.js"
     shell: |
       source ~/.bashrc
       cd app/
       npm install
       npm install -g pm2
       node seeds/seed.js
       pm2 kill
       pm2 start app.js
```
![](ans_workflow.png)

# Moving to cloud
Install dependencies in controller
- `sudo apt-get install python3-pip`
- `sudo pip3 install awscli`
- `sudo pip3 install boto boto3`
- `cd etc/ansible`
- `sudo mkdir group_vars`
- `cd group_vars`
- `sudo mkdir all`
- `cd all`
- `sudo ansible-vault create pass.yml`
```
Choose a password and add:

aws_access_key: ************
aws_secret_key: ************
```
- `ansible-vault edit pass.yml` if we would like to edit the file

## Create new Playbook to run EC2 instances in AWS for us
- Copy the `eng89_devops.pem` file to our controller so it can communicate with AWS
- `sudo scp -i eng89_devops.pem eng89_devops.pem vagrant@192.168.33.12:~/.ssh/`
- SSH into our controller
- in `.ssh` directory run `ssh-keygen -t rsa -C "your_email@example.com"` to generate a public key. It will prompt you to name it, we can use `eng89_devops` to match the pem file
- In `/etc/Ansible` directory add the following to `hosts`:
```YAML
[awsdb]
ubuntu ansible_host=54.194.90.31 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/eng89_devops.pem

[awsweb]
ubuntu ansible_host=34.243.71.83 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/eng89_devops.pem
```
App
- `sudo nano create_ec2_app.yml` and paste the following:
```YAML
---
- hosts: localhost
  connection: local
  gather_facts: True
  become: True
  vars:
    key_name: eng89_devops
    region: eu-west-1
    image: ami-03360601231433d7a
    id: "eng89 launch an aws ec2 instance app"
    sec_group: "sg-098f261f407f87b69"
    subnet_id: "subnet-0429d69d55dfad9d2"
    ansible_python_interpreter: /usr/bin/python3
  tasks:

    - name: facts
      block:

      - name: get instance gather_facts
        ec2_instance_facts:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          region: "{{ region }}"
        register: result

    - name: provisioning ec2 instances
      block:

      - name: upload public key to aws_access_key
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '~/.ssh/{{ key_name }}.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"

      - name: provision instance
        ec2:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          assign_public_ip: True
          key_name: "{{ key_name }}"
          id: "{{ id }}"
          vpc_subnet_id: "{{ subnet_id }}"
          group_id: "{{ sec_group }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: True
          count: 1
          instance_tags:
            Name: eng89_ron_ansible_playbook_app

      tags: ['never', 'create_ec2_app']
```
- To run the playbook with our AWS keys `sudo ansible-playbook create_ec2_app.yml --ask-vault-pass --tags create_ec2_app`

DB
- `sudo nano create_ec2_db.yml` and paste the following:
```YAML
---
- hosts: localhost
  connection: local
  gather_facts: True
  become: True
  vars:
    key_name: eng89_devops
    region: eu-west-1
    image: ami-0c0b30eded2ee2098
    id: "eng89 ansible playbook to launch an aws ec2 instance"
    sec_group: "sg-098f261f407f87b69"
    subnet_id: "subnet-0429d69d55dfad9d2"
    ansible_python_interpreter: /usr/bin/python3
  tasks:

    - name: facts
      block:

      - name: get instance gather_facts
        ec2_instance_facts:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          region: "{{ region }}"
        register: result

    - name: provisioning ec2 instances
      block:

      - name: upload public key to aws_access_key
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '~/.ssh/{{ key_name }}.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"

      - name: provision instance
        ec2:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          assign_public_ip: True
          key_name: "{{ key_name }}"
          id: "{{ id }}"
          vpc_subnet_id: "{{ subnet_id }}"
          group_id: "{{ sec_group }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: True
          count: 1
          instance_tags:
            Name: eng89_ron_ansible_playbook_db
            
      tags: ['never', 'create_ec2']
```
- To run the playbook with our AWS keys `sudo ansible-playbook create_ec2_db.yml --ask-vault-pass --tags create_ec2_db`

# Troubleshooting
If app is still running in the background and port 3000 is in use:
- ssh to awsweb
- `ps aux`
- `sudo kill [number]` find the number that corresponds to pm2