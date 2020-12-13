---
layout:	post
title:	Using bhyve on FreeBSD
date: 2015-07-07 20:00:00
category: freebsd/bhyve
tags: freebsd bhyve libvirt virt-manager virtualization centos vm tech
platform: FreeBSD 10.1
type: it
---

I wanted to write an article about bhyve for a long time now, and fortunately I recently had the time to do just that. Bhyve has the potential to became a very sophisticated, and advanced hypervisor.

* TOC
{:toc}

## Introduction

I'm not sure since when, but bhyve now supports libvirt. That's great news, given that libvirt is a universal tool which helps in managing VMs. There are quite a few hypervisors compatible with libvirt, for example KVM, Xen, Linux containers (LXC) and a [bunch of others](https://libvirt.org/ "Libvirt.org website"). On the other hand, there are only a few graphical fronteds compatible with it, for example [Virtual Machine Manager](https://virt-manager.org/ "Virtual Machine Manager website") (virt-manager), which is a general purpose VM management tool. This can be installed to a computer and the hypervisors can be managed from there. Virt-manager is generally good for a few hypervisors, with 10-20 VMs, but managing a cloud infrastructure is quite a challenge with it. There a few tools compatible with libvirt, which are targeted at clouds or greater deployment.

In this article, I will cover the Virtual Machine Manager (virt-manager). This tool allows for the basic management of VMs, such as installing, starting, deleting etc. The goal of this document is to introduce the installation and configuration of bhyve to a FreeBSD host, libvirt, and the virt-manager to a remote computer, then to explain how you can connect from the virt-manager to the remote host. This setup allows the management of VMs remotely. This article is customized for CentOS guest installation as a guest, however other Linux guests could be installed and used in a similar way.

As its current state, libvirt is not fully compatible with bhyve and therefore interacting with VMs with libvirt is not possbile. However, this article provides instructions for installing and managing VMs with bhyve itself.

## Additional informations for this article

I use the following setup:

- **FreeBSD 10.1-RELEASE-p13** - as the host computer
- **Arch Linux current** - as the remote computer
- These two computers are on one network

## bhyve

So what is exactly bhyve? From the offical website:
>bhyve, the "BSD hypervisor" is a hypervisor/virtual machine manager developed on FreeBSD and relies on modern CPU features such as Extended Page Tables (EPT) and VirtIO network and storage drivers.

It has been released on 20th January, 2014, so it's a quite new, "shiny" technology. Therefore it is possible that a few bugs remain, or some feature do not work yet. If you encounter some, please report it. bhyve supports FreeBSD, OpenBSD, NetBSD and Linux distributions as a guest. Bhyve gets loaded and acts like KVM. This means, that bhyve itself is loaded into the kernel and that allows it to provide good performance to the virtual machines.

In order to boot Linux as a guest, it is important to have the *sysutils/grub2-bhyve* installed.

## Installing and using bhyve

bhyve is by default installed on FreeBSD, it just needs to be loaded.
Load the bhyve kernel module so:

```console
# kldload vmm
```

In order to load automatically while booting up the system. It appends a new line to the */boot/loader.conf* file:

```console
# echo 'vmm_load="YES"' >> /boot/loader.conf
```

Then, build *sysutils/grub2-bhyve* from ports:

```console
# portmaster sysutils/grub2-bhyve
```

Or install the package:

```console
# pkg install sysutils/grub2-bhyve
```

To get further information about bhyve:

```console
# man bhyve
```

## Installing and starting a CentOS guest

If the guest needs to be be connected to the network, a tap interface and a bridge between the physical interface and the tap has to be created. My physical interface is *re0*, it is important to check and correct the commands accordingly.

```console
# ifconfig tap0 create
# sysctl net.link.tap.up_on_open=1
  net.link.tap.up_on_open: 0 -> 1
# ifconfig bridge0 create
# ifconfig bridge0 addm re0 addm tap0
# ifconfig bridge0 up
```

Create the file, which will be used as the disk for the guest:

```console
# truncate -s 8G centos.img
```

A Device map has to be created in order to clarify, which file should be mapped to which virtual device. In the example, both the *centos.img*, the *device.map* file and the *CentOS-7-x86_64-DVD-1503-01.iso* (which is the CentOS installer) was placed in one folder. The content of the device.map file here is:

```console
(hd0) ./centos.img
(cd0) ./CentOS-7-x86_64-DVD-1503-01.iso
```

To start up a Linux guest, two steps must be taken. First, point GRUB to the kernel and initrd, and then the guest can be started. The following line starts the GRUB, points to the previously created device.map file, selects the root device, allocates memory and assigns a name.

```console
# grub-bhyve -m device.map -r cd0 -M 2048 centos
```

Now the GRUB needs to be pointed to the vmlinuz and initrd files. First, check which devices are present:

```console
grub> ls
(hd0) (cd0) (cd0,msdos2) (host)
```

Check, which files are present in the CD:

```console
grub> ls (cd0)/
CentOS_BuildTag EFI/ EULA GPL images/ isolinux/ LiveOS/ Packages/ repodata/ RPM
-GPG-KEY-CentOS-Testing-7 RPM-GPG-KEY-CentOS-7 TRANS.TBL
```

In the *isolinux* folder:

```console
grub> ls (cd0)/isolinux
boot.cat boot.msg grub.conf initrd.img isolinux.bin isolinux.cfg memtest splash
.png TRANS.TBL upgrade.img vesamenu.c32 vmlinuz
```

Then locate vmlinuz and initrd:

```console
grub> linux (cd0)/isolinux/vmlinuz
grub> initrd (cd0)/isolinux/initrd.img
```

Boot the installer:

```console
grub> boot
```

The Linux kernel is now loaded, so it is possible to start the guest.

```console
# bhyve -AI -H -P -s 0:0,hostbridge -s 1:0,lpc -s 2:0,virtio-net,tap0 -s 3:0,virtio-blk,./centos.img -s 4:0,ahci-cd,./CentOS-7-x86_64-DVD-1503-01.iso -l com1,stdio -c 4 -m 2048M centos
```


![CentOS CLI installer configured](/images/bhyve_libvirt/centos_set.png)


If everything went right, the CentOS CLI installer has been started. After being installed, the instance has to be destroyed!

```console
# bhyvectl --destroy --vm=centos
```

Then the installed guest can be started:

```console
# grub-bhyve -m device.map -r hd0,msdos1 -M 2048M centos
```

Also the Linux guest is installed, and a correct grub.cfg is present.

```console
grub> ls (hd0,msdos1)/grub2/
themes/ device.map i386-pc/ locale/ fonts/ grubenv grub.cfg
grub> configfile (hd0,msdos1)/grub2/grub.cfg
```

![Grub loaded with configuration](/images/bhyve_libvirt/grub_configured.png)


Start the VM (without the installer ISO):

```console
# bhyve -AI -H -P -s 0:0,hostbridge -s 1:0,lpc -s 2:0,virtio-net,tap0 -s 3:0,virtio-blk,./centos.img -l com1,stdio -c 4 -m 2048M centos
```

The config file, installed by the CentOS installer, applies the settings to GRUB and the choose option appears. When the OS is loaded:

```console
# uname -a
Linux centos.local 3.10.0-229.el7.x86_64 #1 SMP Fri Mar 6 11:36:42 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```

The previosly created tap0 interface is present in the guest, and it is connected to the network. After the network is configured, the VM is reachable over SSH. After shutting down the guest, it is important to destroy the instance!

```console
# bhyvectl --destroy --vm=centos
```

In order to find out, which VMs are running, the */dev/vmm/* shoud be checked.

```console
# ls /dev/vmm/
```

## Installing and running FreeBSD as a guest

Naturally, it is possible to run FreeBSD as a guest. That's a bit easier task than running Linux. A disk image file is needed to be created:

```console
# truncate -s 16G freebsd.img
```

Then running the installer from the downloaded FreeBSD installer ISO with a built in script:

```console
# sh /usr/share/examples/bhyve/vmrun.sh -c 4 -m 1024M -t tap0 -d guest.img -i -I FreeBSD-10.1-RELEASE-amd64-dvd1.iso freebsd
```

This will start up the guest, booting from the ISO and showing the guest on the terminal. From here, the installation could be performed.

## libvirt

As I have mentioned before, libvirt is a very powerful tool that allows us to interact with hypervisors and management consoles. It is compatible with a bunch of hypervisors, and also with a lot of management tools. It not just makes the local management easier, but with a few hypervisors, it can be used remotely over SSH.

Installing libvirt on FreeBSD is pretty straightforward. *devel/libvirt* has to be either compiled from ports, or installed from packages.

Compiling from source:

```console
# portmaster devel/libvirt
```

It is important to configure libvirt with bhyve support!
Installing from packages:

```console
# pkg install devel/libvirt
```

libvirt provides a command line tool, called "virsh" to interact with hypervisors.
In order to start libvirt, the daemon has to be started:

```console
# service libvirtd onestart
```

To start the *libvirtd* service automatically, when the host boots up, the following line needs to be added to the */etc/rc.conf* file:

```console
libvirtd_enable="YES"
```

>Unfortunately starting a VM (or domain, with virsh syntax) with virsh, freezes the libvirt daemon and the VM actually does not start. In other words, at its current state, it is not possible to use bhyve with libvirt. I hope that this bug will be solved soon. Please also note, that in the example, FreeBSD 10.1-RELEASE-p13 is used, and a possible fix could have been submitted and could be present in 11-CURRENT!

To start the instance with virsh, a centos.xml file has been created in the folder */usr/local/etc/libcirt/qemu* with contents [stated on the website of libvirt](https://libvirt.org/drvbhyve.html "libvirt: Bhyve driver"). Then the file has to be initialized:

```console
# virsh define centos.xml
```

When it is initialized, the VM could be started:

```console
# virsh start centos
```

This is where it freezes. Starting a VM remotely with virt-manager will eventually work, when this issue has been fixed.

## virt-manager

The Virtual Machine Manager (virt-manager) is an easy-to-use interface for managing VMs. It is compatible with libvirt and enables the user to manage bhyve (and a bunch of other hypervisors) remotely, or locally. The developer of virt-manager is Red Hat, and the first stable version has been released on 7th of September, 2014. I have been using this application for a long time to interact with KVM, and I find it a very handy tool.

Despite the fact that its not working as it's current state with bhyve, here is the guide on how to continue. Because virt-manager interacts with libvirt and libvirt is unable to start a VM on bhyve, starting up a VM from virt-manager is not working. The first step is to install the virt-manager with the package manager of the distribution. On Arch Linux:

```console
# pacman -S virt-manager
```

Once installed, it is possible to connect to the bhyve running on the FreeBSD.


![Adding bhyve+SSH connection to virt-manager](/images/bhyve_libvirt/connection_window.png)

The connection has been added to the virt-manager

![Connection details of bhyve on remote host](/images/bhyve_libvirt/connection_details.png)

## Summary

Despite the fact, that bhyve and libvirt have a few bugs, and do not work right now, my hopes are high conserning this combo. I find FreeBSD a very good operating system, which has just needed a fine hypervisor. I hope that the developers of bhyve and libvirt will find the problem soon and will fix it. Until that, it is possible to use bhyve from command line and access the guests from there until the SSH has been configured.

##. Comments

It is important for me to stress, that this article is written to inform, and to be a guide to accomplish something. I am not responsible for any data losses, or for anything that goes wrong. It is your responsibility to understand the meaning of the commands and to determine the outcome.

It is also very important to have a backup of everything.

## Thanks, sources I used

I would like to thank everybody, who contributed to bhyve, libvirt and virt-manager, also the FreeBSD community, RedHat and everyone else who made this possible.
I would like also to thank to [artsies](http://artsiesforever.tumblr.com/ "artsies Tumblr") for proofreading this document.
For this article, I used the following sources:

- [FreeBSD Handbook](https://www.freebsd.org/doc/en/books/handbook/ "FreeBSD Handbook") - FreeBSD as a Host with bhyve
- [libvirt.org](https://libvirt.org/drvbhyve.html "libvirt: Bhyve driver") - Bhyve driver
- [FreeBSD Wiki](https://wiki.freebsd.org/bhyve "bhyve - FreeBSD Wiki") - bhyve
- [Wikipedia](https://en.wikipedia.org/wiki/Virtual_Machine_Manager "Virtual Machine Manager") - Virtial Machine Manager

Thank you for reading.
