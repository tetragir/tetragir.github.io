---
layout:	post
title: NASBuild - The concept
date: 2021-06-20 14:15
category: nasbuild
tags: freebsd ansible concept docker
type: it
---

So now that I decided to build my Home NAS on FreeBSD, I got the define exactly what is it I want. It felt weird letting out the control from my hands using Docker, so that is one thing I would like to get back. To that end, I will configure all services and applications myself. Even though Ansible Galaxy exists, I will write all of my playbooks myself. This way I will have full control over what's happening and of course I will (be forced to) really get to know how to configure the given software.

Before just getting to it and start installing software, I will define some concepts which will guide on how I will set up the applications. Below is a list of my requirements, definitions and other concepts. For all of the software I will install, I will fill out this list to make it better documented and to decide some aspects of the configuration. For example some pretty important service will have to built redundantly and some might be fine if that wouldn't work for some days.

* TOC
{:toc}

# Service definitions

## Redundacy
Some software is important to be available at all times, some are not. I define 2 different redundancy levels for my infrastructure:
* Essential
* Nonessential
* Reproducable

### Essential
These are important software, like DNS. All software defined here will have to run on 2 different machines at the same time. If one server fails or goes down, the other still needs to be able to operate. For my needs it's enough if these software are reachable on different IP addresses and it also might be fine if the software is not actually running in HA mode, but configured the same way and provides the same functionality.

### Nonessential
Not so important software for which it's fine for it to go down and stay like that. As my home network is pretty small, almost all all software fall into this category. Still, there services will be replicated and they will be able to be brought up on my backup server.

### Reproducable
These services are either fully set up by automation or are there for experimentation. Because of this, no redundancy will be set up.

## Monitoring
At some point, I will install some monitoring solution. Under this section, I will define what aspects of the given service will be monitored. For example for a DNS Service, the following items come to my mind:
* The DNS Jail is up (prove by ping)
* If the DNS Service is running (prove by checking if the service is running inside the jail)
* If the DNS Service resolves internal hostnames (prove by resolving a hostname and compare with a pre-defined answer)
* If the DNS Serviuce resolves external hostnames (proving is the same as the previous point)

Defining monitoring goals has the advantage that it's service-independent. It doesn't matter if the monitoring software will be Icinga or Prometheus, the previous test will have to be checked by them.

# Basic concepts

## Decouple software, configuration and data
While I tried Docker, I really liked the concept to decouple the stored data, the configuration and the software itself. In Docker this is done using volumes. All data that is subject to change by the user is stored on a Docker volume. This way, if the container gets updated, or it needs to be rebuilt, all data is still there.

I will use this concept a bit differently and divide differently. Everything that is set up automatically by Ansible will be stored in the Jail, while all data will live in it's own dataset. This way, I will only have to back up the datasets and not the Jails themselves. Thius gives me grater flexibility as I will be able to just start up a container on another server given that the data stored on datasets is there.

## IP Addressing
As I run a pretty small network, I will use one /24 subnet. However it will be divided into 2 parts:
- First half IP Addresses (.0-.128): These addresses will be manually assigned to service that need a fix IP address
- Second half of IP Addresses (.129-.253): These addresses will be assigned dynamically from the DHCP Server for clients like phones.

## Automation vs. Manual setup
I will automate as much as possible, but not everything. Some setup will be done manually, for example creating Jails. This could be also done by automation, but that would just complicate it even more. Besides, doing this part manually fits the redundancy concept as well, as services that are essential will be running even with a machine machine being down and nonessential services can survive the time I create a new jail.

## Software
As stated in my previous post, I will use FreeBSD as operating system. I will also use iocage as it was stable and reliable in the last couple years.

## No self-building packages
One of the downsides of my previous NAS setup was that I build my own packages using Poudriere. This was mostly fully automated, but made the setup more complicated - and was mostly unnecessary. I built my own packages, as for some, I needed some custom options that were different in the packages available for FreeBSD by default. Also, I maintain a couple ports myself and poudriere was set up anyway. This time however, I will do my port maintenance duties in a virtual machine set up by Vagrant. Because of this, I will use the packages available from the FreeBSD Package repository.

## Databases
I will use local, sqlite databases as much as possible. Running a DB cluster is not easy and over time, more and more service would use it and would make it an essential service. However maintaining a DB cluster is not easy that is replicating and failing over correctly.

## Automatic updates
As I would like to have as little maintenance as possible, I will set up the system so that it performs automatic updates on the services. The base OS, FreeBSD will be updated manually once there is a new release.

## Fresh packages
I will use the "latest" available packages as opposed to the quarterly branch. As packages will be updated anyway automatically, if a package breaks or stops working, using the quarterly branch would just delay that.
