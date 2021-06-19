---
layout:	post
title: NASBuild - The Host
date: 2021-06-19 12:40
category: nasbuild
tags: freebsd ansible concept docker
type: it
platform: FreeBSD 13.0 and Ansible 2.11.1
---

The first step on my NASBuild is to install the host OS and set it up. For that, I will create 2 Ansible roles: *base* and *phy*. Base will be assign to all instances, let it be a physical machine, Jail or VM. Phy will be assigned to physical host machines and will contain tasks that will only be applicable to those. I will create the base at another time, as I want to focus only on the settings that are needed to be done on a host.

All of the tasks and roles here are published to [GitHub](https://github.com/tetragir/nasbuild) so it might be useful for someone else too. Please note that as of writing this post, the inventory contains IP addresses as my DNS is not set up yet. Once I have DNS, I will replace the IP addresses with hostnames.

* TOC
{:toc}

# Manual Changes

## Installing Packages

Before I can use Ansible, I have to install python and doas on the the host.

```
root@nas:/boot # pkg install python sudo
```

This will install the default version of Python on the system which is python 3.7 at the time of writing. Once the default gets numped to 3.8, it will be also updated after a pkg upgrade. I install doas instead of sudo. It's much smaller (21kb vs. 6MB) and has a much easier configuration format.

## doas.conf
As I will use Ansible as my user, I have to be able to become root on the host. There are many ways to do that, I will add my user to the doas.conf file to be able to become root without a password. Another way would be to tell Ansible to use a password with doas and define it.

```
root@nas:/boot # echo "permit nopass :wheel" > /usr/local/etc/doas.conf
```

I'm logging in with my user *tetragir* to test if this worked (the user is part of the group *wheel*):

```
tetragir@nas:~ % doas whoami
root
```

# Automated setup

From this point on, the host will be configured with Ansible. For the settings below, I created a role called "phy". The role on GitHub: [https://github.com/tetragir/nasbuild/tree/main/roles/phy](https://github.com/tetragir/nasbuild/tree/main/roles/phy)

## ansible.conf
Before starting executing Ansible playbooks, it needs to be configured. Ansible needs to be told to use doas instead of sudo (the default). These settings can be assigned to hosts as variables, but it will be the same for all of my hosts so I add them to ansible.conf. I also add a couple more settings to the configuration file. The most important settings (for me) are the following:

**[defaults] section**

* ```forks = 40``` => Connect up to 40 hosts at the same time. I don't expect I'll have more than 40 hosts.
* ```ansible_managed = "#### Managed by Ansible ####"``` => So that I can use *{{ ansible_managed }}* in templates

**[privilege_escalation] section**

* ```become=True``` => Use privilege escalation
* ```become_method=doas``` => To use doas instead of sudo
* ```become_user=root``` => Become root user

**[ssh_connection] section**

* ```pipelining = True``` => Speed up connections by not copying all files to the host

**[diff] section**

* ```always = yes``` => All changes to a file will be shown in the output

If this setup is correct, test if Ansible can use the host from the machine where Ansible is used.

```
tetragir@cerebro:~/git/home % ansible -i inventory all -m ping
172.16.10.2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/local/bin/python3.7"
    },
    "changed": false,
    "ping": "pong"
}
```

## Packages
I usually start a role with a list of packages to install. In this case I'll try to do as less as possible as I want all services to be in a Jail. That means that I only install a handful of packages.

## Base settings
By default the loading screen is shown for 10 seconds waiting for an input before booting the OS. This is too much, I usually reduce it to only a couple of seconds. I won't modify the default ```/boot/loader.conf```, but I will create a new file in ```/boot/loader.conf.d/```. As that file is newly created, it's easy to template it. Even though there would be no "dynamic" part in the configuration, I usually put all files in the templates folder and use the Ansible template module instead of just copying them over from the roles' files folder. This gives allows me to dynamically add a comment in the first line that says "#### Ansible Managed ####" so I know not to modify the file on the machine because it will be overwritten.

# Apply the changes
Before executing the playbook, run it with ```--check``` to see what would've happen. If the results are fine, execute it without check.

```
ansible-playbook -i inventory site.yml --check
```
