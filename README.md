# Ansible Guide
![img.png](img.png)
## Ansible controller and agent nodes set up guide
- The following will set up three VMs
- Run `vagrant up` to start all three

```
# -*- mode: ruby -*-
# vi: set ft=ruby :
# All Vagrant configuration is done below. The "2" in Vagrant.configure configures the configuration version (we support older styles for backwards compatibility). Please don't change it.
# MULTI SERVER/VMs environment 
Vagrant.configure("2") do |config|

# creating first VM called web  
  config.vm.define "web" do |web|
    web.vm.box = "bento/ubuntu-18.04" # downloading ubuntu 18.04 image
    web.vm.hostname = 'web' # assigning host name to the VM
    web.vm.network :private_network, ip: "192.168.33.10" # assigning private IP
    config.hostsupdater.aliases = ["development.web"] # creating a link called development.web so we can access web page with this link instread of an IP   
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
## Initial set-up
- SSH into controller VM
- Install dependencies 
````
sudo apt-get update
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
sudo apt-get install tree
````
- Navigate to `cd /etc/ansible/hosts` to add web and db IP
````
[web]
192.168.33.10 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant
[db]
192.168.33.11 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant
````
- Check connection `ansible all -m ping`. To check one host replace all with the host name.
## Installing nginx and nodejs using ansible and YAML
- Navigate to `cd /etc/ansible` 
- Create a playbook in this directory `sudo nano PLAYBOOK_NAME.yml`
- Write the code in the file, keeping an eye on indentation 
- Run the playbook `ansible-playbook PLAYBOOK_NAME.yml`
- This is a playbook to install and set up Nginx in our web server (192.168.33.10)
- This playbook is written in YAML and YAML starts with three ---
```
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
# Set up the reverse proxy the same way we have done previously
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
            proxy_set_header Upgrade \$http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host \$host;
            proxy_cache_bypass \$http_upgrade;
            }
          }
      notify:
        - Restart nginx
    - name: update npm
      command: npm install pm2 -g -y
```
## Setting up Ansible-Vault and adding keys
- SSH into the controller
- Run the following commands
```
  sudo apt install python3-pip
  sudo pip3 install awscli
  sudo pip3 install boto boto3
  sudo apt-get update -y
  sudo apt-get upgrade -y
```
- Navigate to `cd /etc/ansible`
- Create a group variables directory `mkdir group_vars`
- cd into this directory and create a new directory `mkdir all`
- Run the following command `sudo ansible-vault create pass.yml` and create a password. This puts you into an editor
- Press `i` to insert text and add the corresponding keys
```
aws_access_key:
aws_secret_key:
```
- Press `esc` and type `:wq!` to save and exit
- `sudo cat pass.yml` this should return encrypted password

## EC2 instance provision using Ansible
```
---

- hosts: localhost
  connection: local
  gather_facts: True
  become: True
  vars:
    key_name: eng89_devops
    region: eu-west-1
    image: 
    id: "Ansible playbook to launch an aws ec2 instance"
    sec_group: 
    subnet_id: 
    ansible_python_interpreter: /usr/bin/python3
```
## Packer Task 
- Create a new directory in the ansible directory `mkdir packer`
- Navigate into the directory and create a file `sudo nano packer_task.pkr.hcl`
- The code added into this file can be broken down into three sections
- **Packer Block**
```
packer {
  required_plugins {
    amazon = {
      version = ">= 0.0.02"
      source  = "github.com/hashicorp/amazon" 
    }
  }
}
```
- **Source Block**
```
source "amazon-ebs" "ubuntu" {
  ami_name      = "eng89_saim_playbook_app_AMI"
  instance_type = "t2.micro"
  region        = "eu-west-1"
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/*ubuntu-xenial-16.04-amd64-server-*"
      root-device-type    = "ebs" # Default
      virtualization-type = "hvm" # Fully virtualized set of hardware and boot by executing the master boot record of the root block device of your image.
    }
    most_recent = true
    owners      = ["1"]
  }
  ssh_username = "ubuntu"
}
```
- **Build Block**
```
build {
  sources = [
    "source.amazon-ebs.ubuntu"
  ]
}
```
- Authenticate to AWS. Create environmental variables for access keys and secret keys.
```
export AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY
export AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY
```
- Run the following
```
packer init . # Initialize your Packer configuration
packer fmt . # Format your template. Packer will print out the names of the files it modified
packer validate . # Make sure your configuration is syntactically valid and internally consistent
packer build filename 
```