---
layout:	post
title:Why I switched back from Linux to FreeBSD on my NAS
date: 2021-06-12 10:15
category: IT
tags: freebsd rundeck ansible docker linux
type: it
platform: FreeBSD 13.0
---

So in my previous post from approx. a month ago, I explained my decision to switch from Linux to FreeBSD on my NAS. A month has passed since but now I decided that I'm switching back.

* TOC
{:toc}

## ZFS
My most important requirement on my NAS is to have ZFS. I trust ZFS, I use it since many years and it has never failed me. I used to back up the NAS using zrepl (which uses ZFS send/recv), I take regular snapshots, etc. When I was deciding what distribution to chose, I decided to use CentOS Stream. The reason behind this decision is that it's a fairly stable system, I have a lot fo experience with it, it's flexible and even though now it's not a carbon-copy of RedHat, for my needs it's perferctly fine. I'm not running any critical application on my NAS. Anyway so I started using ZFS on Linux using CentOS Stream. This was fine at the start, all Docker containers were working, I guess speed was also fine, I never did any benchmarks. I also turned on auto-update to make sure I have the latest and most secure software (and also because I'm lazy and I don't want to manually update systems).

The above combination, CentOS Stream, ZFS On Linux, auto-updates resulted in a non-working system after a reboot. I tried to reinstall ZFS, so a kernel module is compiled to the newest kernel, but that failed. I also tried to boot an older kernel, but the system didn't even start. That's when I decided that, while it was fun, I'm going back to FreeBSD. I use FreeBSD since 6 years and so far it never failed me.

## The new concept
There were a couple of things I liked in Docker, one being the decoupling of the data and the "system" using volumes. I want to have this concept on FreeBSD as well, albeit it will have to set it up manually (or rather with Ansible) and not with any tool.

In any case, I will try to document as much as I can here so it might be useful for someone else as well.
