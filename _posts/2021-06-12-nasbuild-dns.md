---
layout:	post
title: NASBuild - DNS
date: 2021-07-01 19:00
category: life
tags: freebsd ansible dns unbound nasbuild
type: it
platform: FreeBSD 13.0 and Ansible Core 2.11.2
---

The most important service in an infrastructure, at least in my opinion, is reliable DNS. Mostly everything relies on DNS, using IP addresses just like that is rare. Also it's quite easy to have DNS set up. There are many solutions that offer out-of-the box good experience, even including adblocking, like Pi-Hole. While I think it's a good project, I want to just use unbound, a very lightweight and reliable DNS server. Also Pi-Hole is not available for FreeBSD.

I separated out the various roles into different repositories on GitHub to make it easier to just cheery-pick a role and not have to download the whole setup. The repository for this setup can be found on [GitHub tetragir/unbound-freebsd](https://github.com/tetragir/unbound-freebsd). Please note that this role will work on FreeBSD, but not immediately on Linux. In order to make it work, the package name, paths and "fetch" in the cronjob needs to be replaced.

* TOC
{:toc}

## Installing the application
Unbound is available from packages, so I'm installing it from there.

```jinja
- name: Install unbound
  ansible.builtin.package:
    name: dns/unbound
    state: present
```

## Configuration files
I use 4 different files for configuration:
* localdns.conf - In this file, I list all internal hostnames. Items are list items in the vars/main.yml file
* block.conf - If there are any websites that should be blocked on top of the adblock list, they can be listed here
* adblock.conf - This file contains all blocked domains. The file is only created with Ansible but not filled. There is a cronjob that will update this file
* unbound.conf - The main configuration file for Unbound

In the configuration file, I mostly left the default values. Queries from private IP addresses are allowed, everything else is blocked. In addition for templating the configuration files, the sample configuration is removed.

## Start and enable the service
Once all files are in place, the service can be started and enabled.

```jinja
- name: Make sure unbound is running and enabled
  ansible.builtin.service:
    name: unbound
    state: started
    enabled: yes
```

## Add cronjob to update adblock list weekly
I update the adblock list once a week using cron. This cronjob will download a current list and fill the previously created adblock.conf file so that the DNS Server will reply with an NXDOMAIN for the domains listed. It will also reload Unbound.

```jinja
- name: Update adblock list
  ansible.builtin.cron:
    name: Update Adblock list
    weekday: '6'
    hour: '4'
    minute: '20'
    user: unbound
    job: >
      fetch https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts -o - |
      grep -v "#" | grep "0.0.0.0" |
      awk '{print "local-zone: \""$2"\" always_nxdomain"}' >
      /usr/local/etc/unbound/adblock.conf &&
      kill -HUP `cat /usr/local/etc/unbound/unbound.pid`
```
