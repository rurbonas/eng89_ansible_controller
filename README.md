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

Check all the servers:
- `ansible all -a "uname -a"` # Check names
- `ansible all -a "date"`     # Check dates
- `ansible all -a "free -m"`  # Check memory

Make a YAML Playbook file (TAB doesn't work - yet indentation matters):
- `sudo nano nginx_playbook.yml`
- Add:
```
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

