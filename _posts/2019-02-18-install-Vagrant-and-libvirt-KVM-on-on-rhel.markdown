---
layout: post
title:  "Install Vagrant and libvirt (KVM) on RHEL7"
date:   2019-02-18 16:30:00 +0200
tags: random
author: pgustafs
---


# How I installed Vagrant and libvirt (KVM) on RHEL7

Hi,

While working with [Molecule](https://github.com/ansible/molecule)-based tests on our Ansible roles, I had to install vagrant
and libvirt (KVM) on my RHEL7 machine to quickly spin up instances for test purposes. 
(will cover the Molecule stuff in a later post). The advantage of using Vagrant is that it
downloads the pre-built Vagrant boxes that are ready to be consumed with in no-time.
A **Box** is a package that contains the base image of the VM. (Image of the operating system (template))
Boxes are stored in repositories and by default vagrant uses the HashiCorp repository at [https://vagrantcloud.com/search](https://vagrantcloud.com/search)
but you can also create your own Box and store it in **~/.vagrant.d/boxes**.


**NOTE!** Vagrant is **Not** a virtualization software, it is a tool for building and managing
virtual machine environments in a single workflow.

## Installation
1. Enable needed repositories
```
sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-optional-rpms
```
In case you have installed "workstation" variant, replace "server" with "workstation" in above.

2. Install the 'Virtualization Host' package group
```
sudo yum groupinstall "@Virtualization Platform"
```
3. Start the libvirtd service and have it start at boot time
```
sudo systemctl start libvirtd.service
sudo systemctl enable libvirtd.service
```
4. Add your user to the libvirt group (make life easier for yourself) 
```
sudo usermod -aG libvirt USER
```
5. install the vagrant rpm from a remote repo.
```
sudo yum install https://releases.hashicorp.com/vagrant/2.2.3/vagrant_2.2.3_x86_64.rpm
```
6. Install build dependencies for vagrant-libvirt plugin installation
```
sudo yum install libvirt-devel ruby-devel gcc
```
7. Install the Libvirt vagrant plugin
```
vagrant plugin install vagrant-libvirt
vagrant plugin list
```
8. Setup environment variable to let vagrant know that we would like to use the Libvirt plugin (vagrant defaults to VirtualBox)
```
export VAGRANT_DEFAULT_PROVIDER=libvirt
export VAGRANT_PREFERRED_PROVIDERS=libvirt
```


## Did I really execute all those commands manually?

Nah, since I'm practicing an Automation first policy, I only executed one command;)
```
ansible-playbook -e "user=USER" vagrant-libvirt.yml
```
And the playbook look like this:
```
- hosts: localhost
  become: true
  connection: local
  vars:
  tasks:
    - name: install the 'Virtualization Host' package group
      yum:
        name: "@Virtualization Platform"
        state: present

    - name: Start the libvirtd service and have it start at boot time
      service:
        name: libvirtd.service
        state: started
        enabled: yes

    - name: Adding existing user {%raw%}{{ user }}{%endraw%} to group libvirt
      user:
        name: "{%raw%}{{ user }}{%endraw%}"
        groups: libvirt
        append: yes

    - name: install the vagrant rpm from a remote repo
      yum:
        name: https://releases.hashicorp.com/vagrant/2.2.3/vagrant_2.2.3_x86_64.rpm
        state: present

    - name: Install build dependencies for vagrant-libvirt plugin installation.
      yum:
        name:
          - libvirt-devel
          - ruby-devel
          - gcc
        state: present

    - name: List installed Vagrant plugins
      shell: "vagrant plugin list"
      changed_when: False
      register: vagrant_initial_plugins_list

    - name: Install Vagrant plugins
      shell: "vagrant plugin install vagrant-libvirt"
      when:
        - '"vagrant-libvirt" not in vagrant_initial_plugins_list.stdout'
```

BR, pgustafs
