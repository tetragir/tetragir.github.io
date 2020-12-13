---
layout:	post
title:	How I update FreeBSD and packages on a Raspberry PI with Rundeck and Ansible
date: 2020-12-13 16:00
category: Raspberry
tags: freebsd raspberry rundeck ansible automation
type: it
platform: FreeBSD 12.2
---

This is my first post in a VERY long time. This is the result of a couple of things. 2020 turned out to be a year with a lot of change to put it mildly. I also changed jobs, which turned out to be a very good decision even if I took a risk, doing it in the middle of a pandemic. But to my defense, there were only some rumors when I started the whole process.

Thanks to the lockdowns and quarantine, I had significantly more time to work on my home infrastructure and I made a lot of progress. One leap forward is that I started automating processes which allowed me to focus on other topic. A result of this automation is the fact that the Raspberries in my infrastructure are running the newest release of FreeBSD and packages.

* TOC
{:toc}

## Introduction
Rundeck and Ansible allowed me to automate the building of the OS and packages which made it possible for me to just lean back and deploy an updated FreeBSD and packages with press of a button. One of a painpoints for me when I started using a Raspberri Pi with FreeBSD, that freebsd-update wouldn't work. This is due to the fact that ARM (aarch64 in particular in my case) is still a Tier 2 platform. Once a new release is published, it is available as an SD Card image, but there is no way to upgrade from an installed OS using freebsd-update.

Another aspect is packages. I not only specify custom options for compiling a couple of packages, but only the quarterly branch is available for aarch64. For some time I used my Raspberry Pi as is, with no OS upgrades (as it's in a different country and traveling is not really possible nowadays) and using quarterly packages.

## Automation software
### Rundeck
Rundeck is an automation software that allows administrators to create pre-defined workflows and allow access to these workflows to different parts of the organization. A good example would be a process of adding a new user and their SSH key to all of the organization's servers. Traditionally an admin would either log in to all servers, create the user and paste SSH key or use some jenky script that does this. Rundeck enables the admin to create a workflow for this process to speed up this deployment or even give access to this workflow to HR.

Access to different jobs and processes can be fine-tuned which makes it possible to create different workflows and different tasks and give access to different parts of the company to perform these workflows safely.
