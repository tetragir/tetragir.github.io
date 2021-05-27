---
layout:	post
title:	Why I switched from FreeBSD to Linux on my NAS
date: 2021-05-15 08:00
category: life
tags: life blog
type: life
---


This is my first post in a really long time... There were a lot of changes since the last post. For a long time my blog was hosted on my very own VPS on DigitalOcean, then I moved it to Hetzner, then to another provider that does webhosting as a sidebusiness, but I think now I found a long-term solution. It's hosted now on GitHub Pages. The change was fairly easy, as the blog was anyway written for Jekyll and GitHub Pages have native support for it, so no changes were needed to be done, other then changing the origin URL in git.

With that said, this shows a change on what I spend my time with.

* TOC
{:toc}

## How it started

For a long time I was using FreeBSD as my OS of choice on my NAS. I started out with FreeBSD when 10.0 was released and learned a lot about how Unix works. I used this knowledge later in my carreer as most of it applied to Linux as well.

So anyway my NAS was always running FreeBSD. At the beginning it was FreeNAS, but when my old NAS died, I opted to use vanilla FreeBSD on the new hardware. I installed every service in a separate Jail. Then I started to use Rundeck which was outdated in the ports tree, so I decided to take ownership and update it. It shouldn't be that difficult, it's a Java app. The rundeck port was split to rundeck2 and rundeck3, the former was kept only to cater to users of this old version, even thought it's not supported anymore. Similar thing happened with Kanboard, it was outdated and I successfully managed to update it. For this, however, I needed to have some kind of a staging area, also some kind of a tool to be able to build packagaes. This is how my own build "infrastructure" was born with the help of Poudriere.

## Complications

This solution worked fine, until I realized that I needed some custom options for some ports so I started to build all packages myself. After configuring the update method, the process was mostly automated through Rundeck and Ansible, however even though it was running fine, there was always something to do, something to fix, update, to fiddle with.

Lately I focus a bit less on my own home-IT, as I have other priorities too. After some discussions with the other sysadmin in-house (my lovely wife), we first decided to opt for FreeNAS (again) and use services in Jails. However after some research, it turned out that there are only a very limited number of pre-built Jails for FreeNAS and for a lot of services we need, we again need to build them manually and then maintain it.

# The Switch

Then came Linux and Docker. Or better said, I tried them out. They were around for sime time now... So after doing some experiments, we decided to just use Linux (in this case CentOS) and Docker on the NAS. The setup was fairly easy, may be because I use CentOS at works as well. I opted for CentOS Stream, I know the controversy behind it, but it fits my needs so why not. I was thinking about Fedora as well, but the fact that ZFS support is limited (as new kernels are released farily often) made me sure to stick with CentOS.

Even though I switched to Linux, there are a couple things I wanted to keep using. Most important is ZFS. Fortunately ZFS On Linux picked up the pace in the last couple of years, installing ZFS on CentOS was as easy as adding a new repository and installing the package. There was some complication installing the kABI package, maybe because I'm running Stream, but the DKMS package seems to be working now.

## Summary

All in all, I think this was change necessary. We use quite a few applications and I like the idea how these applications are spearated from each other and the OS. It's really usefuly how it's possible to Docker volumes to other hosts and just start the container there like nothing happened.

A couple applications we use:
- Nextcloud
- Bitwarden
- Calibre
- Pi-Hole
- Grocy
