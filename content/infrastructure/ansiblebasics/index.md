---
title: "Intro To Ansible"
summary:  "Creating some basic ansible scripts as an intro to help our deloyment"
date: 2025-07-20
tags: ["deployment", "ansible", "scripting"]
draft: false
---


##An Intro To Ansible
This is going to be a crash course in Ansible and we are going to create 2 ansible roles and combine them into a machine prep script.

First ansible is an automation scripting tool for configuration and management of machines which is agentless.  It mostly depends upon ssh which also means ssh keys. 

Roles in ansible I like to think of as lego blocks you can create to build make scripts more readable, resusable and module.  These scripts will be placed in our infrastructure git once we stand up Gitea.  


Since Ansible is already installed on our machine lets prep a directory for this repo.  First we are going to make a directory and then use git to initialize it and then 

```css
mkdir -p infrascripts/ansible/roles
cd infrascripts/roles
git init --initial-branch=main
```

In the role directory we are going to start making our own custom role

```css
ansible-galaxy role init osupdate

```

We're are also going to install an already existing role by Jeff Geerling for installing docker.  We are going to retrieve from the Ansible Galaxy store which is a large repo of preconfigured roles and collections.  It's a good place to check before writing your own.  

```css
ansible-galaxy role install geerlingguy.docker
```

Now lets make our custom role.  This will be for just updating the OS in our environment.  

So we want to cd into the tasks directory of this role and make a main.yaml file for this role and then edit it.   main.yaml in the directories is the default file Ansible is looking for.  
```css
vi roles/osupdate/tasks/main.yaml

```

Should would be like this

```css

    - name: Update apt repo and cache on all Debian/Ubuntu boxes
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

    - name: Upgrade all packages on servers
      apt: upgrade=dist force_apt_get=yes

    - name: Check if a reboot is needed on all servers
      stat:
        path: /var/run/reboot-required
      notify: Reboot Box

    
     
```css
- To break this down, we are basically going to do an apt-get update

- Then an apt-get dist-upgrade -y

- If a reboot is needed we will send the handler to reboot



In the handlers directory we are going to edit the main.yaml file.   In Ansible you can send tasks to handlers.  That's what the notify file call says.  

```css
vi roles/osupdate/handlers/main.yaml
```

```css
- name: Reboot Box
  ansible.builtin.reboot:
    msg: "Reboot initiated by Ansible for kernel updates"
    connect_timeout: 5
    reboot_timeout: 300
    pre_reboot_delay: 0
    post_reboot_delay: 30
    test_command: uptime
```

Handlers are special tasks that only run when notified, and they run at the end of the playbook.  Like in this example if the system needs to be rebooted, complete all other tasks then reboot.  

We’ll create a general OS update script to run periodically, and a Docker install script that first updates the OS, then installs Docker

The first script we are going to make is the docker install script so lets make an install_docker.yaml

```css
---
- hosts: all
  become: true

  roles:
    - osupdate
    - geerlingguy.docker

  tasks:
    - name: adding current user to docker group
      ansible.builtin.user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes
      when: ansible_user is defined
```

So lets break this down
  -  hosts all is any host we define in our inventory
  - become is saying sudo up before doing anything

-  Performs an OS update and reboots if needed
- Install docker using the docker role
- Finally what ever user we use in the -u attribute when running the playbook will be added to the docker group 

The last script we are going to do is just the OS update script and its much simpler.  So in an osupdate.yaml file put the following:

```css
- hosts: all
  become: true

  roles:
    - osupdate
```

That's enough scripts to starting setting up the rest of environment.  To get Gitea up and running we are going need a bit more backend infrastructure.  The following is the infrastructure we should build first

- Internal DNS server
- Reverse Proxy handling SSL certs
- Authentication
- Finally Gitea itself
So the next blog will focus on setting up a basic internal DNS server.
