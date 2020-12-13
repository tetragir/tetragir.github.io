---
layout:	post
title:	Cisco Virtual Wireless LAN Controller on FreeBSD bhyve
date: 2016-05-27 18:00:00
category: freebsd/bhyve
tags: freebsd bhyve  virtualization vm cisco vwlc wireless_controller wifi wireless
type: it
platform: FreeBSD 11, Cisco vWCL 8.1.100.0
---

In this guide a Cisco vWLC will be installed on FreeBSD bhyve. Altough the [Cisco Virtual Wireless LAN Controller](https://www.cisco.com/c/en/us/products/wireless/virtual-wireless-controller/index.html) (vWLC) is officially supported only on VMWare ESXi, Linux KVM and Microsoft Hyper-V, it is relatively easy to install and operate on FreeBSD bhyve. The installation is actually pretty similar to KVM. 

* TOC
{:toc}

## Setup

I use the following setup in this guide:

- FreeBSD 11-CURRENT r297234
- Cisco Virtual Wireless LAN Controller image 8.2.100.0 (Small Scale Image)

## Preparation, requirements

### bhyve and GRUB

In order to run any VMs with bhyve, the vmm module has to be loaded in (or compiled in the kernel). To load it in:

```console
# kldload vmm
```

The following is needed for the vmm module to load automatically at startup. It appends a new line to the */boot/loader.conf* file.

```console
# echo 'vmm_load="YES"' >> /boot/loader.conf
```

The [*sysutils/grub2-bhyve*](https://www.freshports.org/sysutils/grub2-bhyve/) also has to be installed either from ports with portmaster,

```console
# portmaster sysutils/grub2-bhyve
```

or from packages.

```console
# pkg install grub2-bhyve
```

### The Cisco Virtual Wireless LAN Controller

The requirements of the Cisco vWLC are pretty straightforward. With the Small Scale Image, the restrictions are 200 Access Points and 6000 clients maximum, while with the Large Scale Image it is possible to register up to 3000 access points and have up to 32000 clients. In this scenario I use the Small Scale Image. 

The requirements are (Small Scale / Large Scale):

- **CPU:** 1 vCPU / 2 vCPU
- **RAM:** 2GB / 8GB
- **Disk space:** 8GB / 8GB
- **Network Interfaces:** 2 vNIC / 2vNIC

The only difference is with the VMs and install images; the licensing and the controllers are the same. More can be read about the Cisco Virtual Wireless LAN Controller on [Cisco's website](https://www.cisco.com/c/en/us/products/collateral/wireless/virtual-wireless-controller/data_sheet_c78-714543.html).

### Prepare the environment

In order to install the vWLC, the installer ISO is needed: this can be downloaded from [Cisco's website](https://software.cisco.com/portal/pub/download/portal/select.html?&mdfid=284464214&flowid=34567&softwareid=280926587).

Now create the file used as storage.

```console
# truncate -s 8G vwlc.img
```

Create a device.map file for GRUB identifying which file is which device. The contents are:

```console
(hd0) ./vwlc.img
(cd0) ./MFG_CTVM_8_2_100_0.iso
```

(The following files are in one folder)

```console
MFG_CTVM_8_2_100_0.iso	device.map		vwlc.img
```

## Install

The two network interfaces are not needed here, and thus they will be created later on. 

The installation process of the Cisco vWLC is pretty easy, since the ISO file contains an automated installer. Once the installer ISO is booted up, it looks for the virtual hard drive at **/dev/sda**. If it finds it and the disk is empty, a script installs the vWLC software and then reboots. Everything is automatic, the only interaction actually needed is in the case of a previous installer present on the virtual hard drive.

Actually loading the installer ISO is not so straightforward however, as Cisco uses GRUB version 1. Fortunately grub2-bhyve can load the legacy config file.

Start GRUB and point it to the legacy config file:

```console
# grub-bhyve -m device.map -r cd0 -M 2048M vwlc
```

```console
grub> ls (cd0)/boot/grub/
menu.lst menu_vm.lst stage2_eltorito
grub> legacy_configfile (cd0)/boot/grub/menu.lst
```

![vWLC Installer Bootloader](/images/vwlc_bhyve/vwlc_installer_bootloader.png)

After that, the virtual machine can be started. Right now only the console, the hard disk (with ahci-hd emulation!) and the installer ISO is needed. Besides that, 1vCPU and 2048MB of RAM is defined.

```console
bhyve -A -H -P \
-s 0:0,hostbridge \
-s 1:0,lpc \
-s 2:0,ahci-hd,./vwlc.img \
-s 3:0,ahci-cd,./MFG_CTVM_8_2_100_0.iso \
-l com1,stdio \
-c 1 \
-m 2048M \
vwlc
```

![vWLC Installer Done](/images/vwlc_bhyve/vwlc_installer_done.png)

The installation is automatic and takes little time. After a successful install, the Virtual Machine powers down.

Note: It is important to "destroy" the VM! To do this, first check which VMs are running:

```console
# ls /dev/vmm 
vwlc
```

Then stop the vWLC.

```console
# bhyvectl --vm=vwlc --destroy
```

## First start, setup

From this point, the Cisco vWLC needs two vNIC in order to operate, else it will crash during startup. The next few lines define two tap interfaces and a bridge between the first one and the physical interface. The name of the physical interface can differ, check with *ifconfig*!

```console
# ifconfig tap0 create
# ifconfig tap1 create
# sysctl net.link.tap.up_on_open=1
  net.link.tap.up_on_open: 0 -> 1
# ifconfig bridge0 create
# ifconfig bridge0 addm *em0* addm tap0
# ifconfig bridge0 up
```

Load GRUB and point it to the legacy config file

```console
# grub-bhyve -m device.map -r hd0,msdos1 -M 2048 vwlc
```

```console
grub> legacy_configfile /grub/menu.lst
```

![vWLC Booting with GRUB](/images/vwlc_bhyve/vwlc_installed_grub.png)

After that the VM could be started similarly as before, but now without the installer ISO and with the two newly created vNICs.

```console
# bhyve -A -H -P \
-s 0:0,hostbridge \
-s 1:0,lpc \
-s 2:0,ahci-hd,./vwlc.img \
-s 3:0,virtio-net,tap0 \
-s 4:0,virtio-net,tap1 \
-l com1,stdio \
-c 1 \
-m 2048M \
vwlc
```

The setup of the vWLC is now loaded and it can be configured. After the setup process, it reboots.

Note: It is important to destroy the instance after every shutdown!

```console
# bhyvectl --vm=vwlc --destroy
```

## Operation

While starting the vWLC with the bhyve command above, the current console session is lost. Therefore it is advisable to use a nulldevice for that very reason. The GRUB process is the same as above, the difference is only defining the console device for the virtual machine.

```console
bhyve -A -H -P \
-s 0:0,hostbridge \
-s 1:0,lpc \
-s 2:0,ahci-hd,./vwlc.img \
-s 3:0,virtio-net,tap0 \
-s 4:0,virtio-net,tap1 \
-l com1,/dev/nmdm0A \
-c 1 \
-m 2048M \
vwlc &
```

Connect to the nulldevice's other end:

```console
cu -l /dev/nmdm0B
```

If the vWLC was properly set up, it is now available to log in from a web browser.

![vWLC Login screen](/images/vwlc_bhyve/vwlc_login_https.png)

## Summary

Unfortunately, running the vWLC on FreeBSD bhyve is not yet officially supported by Cisco, but it boots up and looks like it's working. By default the vWLC comes with a 90 days evaluation license for up to 200 AP. After the 90 days the vWLC will continue to work but will deny any APs trying to register.
