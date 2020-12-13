---
layout:	post
title:	Booting FreeBSD on a Raspberry Pi Zero
date: 2016-01-01 15:00:00
category: freebsd/raspberry
tags: [freebsd raspberry raspberrypi raspberrypizero zero]
type: it
platform: FreeBSD 10.2
---

Luckily, I was able to get a Raspberry Pi Zero not so long ago. Naturally, my first move was to try to boot FreeBSD on it. Although there is no official support for the Zero on FreeBSD, given that the hardware is very similar to the Raspberry Pi A model, I was able to boot it. The process is quite simple.

* TOC
{:toc}

## Update

The guide is no longer accurate mainly because FreeBSD 10.3 came out and also 11-CURRENT doesn't need to be modified in order to boot on the Zero. You can follow this guide with the 11-CURRENT image, it will work without modifying the files on the boot partition. The images could be downloaded from the [FreeBSD ARM images](ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/arm/armv6/ISO-IMAGES/11.0/) page. Look for the RPI-B image.
The downloaded image can easy copied to the microSD card and it will boot. There is no need to modify the files in the boot partition.

## Scope of this guide
This guide gives instructions on how can one boot FreeBSD 10.2-STABLE on a Raspberry Pi Zero. As a result, the operating system boots up, and is available over SSH. The instructions for the activities on the SD card are written and done on a Fedora 23, but they should apply to many other Linux/UNIX systems. I use the following additional components:

The SD card: **Samsung microSD EVO+ 32GB (MB-MC32DAAMZ)** 

The microUSB - Ethernet adapter: **Icy Box IB-AC510**

## Preparing the SD card
On my system, the SD card came up as */dev/sdb*. It is possible that on other systems it would be something else. Checking the *dmesg* or *lsblk* could be very helpful. The first step is to “format” the SD card. I zero out the card before doing anything on it.

```console
# dd if=/dev/zero of=/dev/sdb bs=4M
dd: error writing ‘/dev/sdb’: No space left on device
7633+0 records in
7632+0 records out
32010928128 bytes (32 GB) copied, 2205.6 s, 14.5 MB/s
```

This process usually takes a long time, but depends on the size and maximal speed of the SD card.

## Downloading and writing the FreeBSD image to the SD card
In the meantime, download the actual FreeBSD image and unxz it. I use the **10.2-STABLE** image, but it should work with the 11-CURRENT image too. The up-to-date images are available on the [FreeBSD ARM Download page](ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/arm/armv6/ISO-IMAGES/11.0/ "FreeBSD 10.2-STABLE ARM images").
After downloading, the file should be checked against the checksum and then it will be decompressed.

```console
$ sha512sum FreeBSD-10.2-STABLE-arm-armv6-RPI-B-20151229-r292855.img.xz 
19e051736e7112e7a0fd699e0d404cde35000b774b3c5fffbd03be35850b0efa4309d9269dd5666d326fd8c8a74ecea9d9b1cf2de28e44cc352e0e91f5e1544f  FreeBSD-10.2-STABLE-arm-armv6-RPI-B-20151229-r292855.img.xz
```

If the checksum is equal to the value shown on the Download page, then decompress it.

```console
$ unxz FreeBSD-10.2-STABLE-arm-armv6-RPI-B-20151229-r292855.img.xz
```

The result is an *IMG* file, that should be copied to the SD card.

```console
# dd if=FreeBSD-10.2-STABLE-arm-armv6-RPI-B-20151229-r292855.img of=/dev/sdb bs=4M
120+0 records in
120+0 records out
503316480 bytes (503 MB) copied, 34.9966 s, 14.4 MB/s
```

A few files need to be replaced on the card in order to make it boot on the Raspberry Pi Zero.

## Making the SD Card bootable on the Zero
The easiest way now is to remove and reattach the card. After reattaching, a partition is mounted (if not, it needs to be mounted manually).

```console
# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
…
sdb      8:16   1 29.8G  0 disk 
├─sdb1   8:17   1   17M  0 part /run/media/user/MSDOSBOOT
└─sdb2   8:18   1  463M  0 part 
```

The *MSDOSBOOT* partition contains the following files:

```console
$ ls
BOOTCODE.BIN  FIXUP_CD.DAT  RPI.DTB       START.ELF  UBLDR.BIN
CONFIG.TXT    FIXUP.DAT     START_CD.ELF  UBLDR      U-BOOT.IMG
```

The following files need to be deleted:

```console
$ rm BOOTCODE.BIN FIXUP_CD.DAT FIXUP.DAT START_CD.ELF START.ELF
```

The new files need to be downloaded from the [raspberrypi/firmware Github repo](https://github.com/raspberrypi/firmware/tree/master/boot "The offical Raspberry Pi Github repo").
The files then could be copied to the card.

```console
$ cp bootcode.bin start.elf start_cd.elf fixup.dat fixup_cd.dat /run/media/user/MSDOSBOOT/
```

The card can now be removed and inserted into the Raspberry Pi Zero.

```console
# sync
# umount /run/media/user/MSDOSBOOT
```

## Booting FreeBSD
The first boot takes a lot of time, because the second partition gets resized. After booting up, it is possible to log in over SSH, or locally. The default user and password combos are:
>freebsd / freebsd

>root / root

## dmesg on the Raspberry Pi Zero

```console
KDB: debugger backends: ddb
KDB: current backend: ddb
Copyright (c) 1992-2015 The FreeBSD Project.
Copyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994
	The Regents of the University of California. All rights reserved.
FreeBSD is a registered trademark of The FreeBSD Foundation.
FreeBSD 10.2-STABLE #0 r292855: Tue Dec 29 16:17:05 UTC 2015
    root@releng1.nyi.freebsd.org:/usr/obj/arm.armv6/usr/src/sys/RPI-B arm
FreeBSD clang version 3.4.1 (tags/RELEASE_34/dot1-final 208032) 20140512
VT: init without driver.
CPU: ARM1176JZ-S rev 7 (ARM11J core)
 Supported features: ARM_ISA THUMB2 JAZELLE ARMv4 Security_Ext
 WB enabled LABT branch prediction enabled
  16KB/32B 4-way instruction cache
  16KB/32B 4-way write-back-locking-C data cache
real memory  = 503312384 (479 MB)
avail memory = 483176448 (460 MB)
random device not loaded; using insecure entropy
random: <Software, Yarrow> initialized
kbd0 at kbdmux0
ofwbus0: <Open Firmware Device Tree>
simplebus0: <Flattened device tree simple bus> mem 0x20000000-0x20ffffff on ofwbus0
cpulist0: <Open Firmware CPU Group> on ofwbus0
cpu0: <Open Firmware CPU> on cpulist0
bcm2835_cpufreq0: <CPU Frequency Control> on cpu0
intc0: <BCM2835 Interrupt Controller> mem 0xb200-0xb3ff on simplebus0
systimer0: <BCM2835 System Timer> mem 0x3000-0x3fff irq 8,9,10,11 on simplebus0
Event timer "BCM2835 Event Timer 3" frequency 1000000 Hz quality 1000
Timecounter "BCM2835 Timecounter" frequency 1000000 Hz quality 1000
bcmwd0: <BCM2708/2835 Watchdog> mem 0x10001c-0x100027 on simplebus0
gpio0: <BCM2708/2835 GPIO controller> mem 0x200000-0x2000af irq 57,59,58,60 on simplebus0
gpio0: read-only pins: 46,47,48,49,50,51,52,53.
gpio0: reserved pins: 48,49,50,51,52,53.
gpioc0: <GPIO controller> on gpio0
gpiobus0: <OFW GPIO bus> on gpio0
gpioled0: <GPIO led> at pin(s) 16 on gpiobus0
iichb0: <BCM2708/2835 BSC controller> mem 0x205000-0x20501f irq 61 on simplebus0
iicbus0: <OFW I2C bus> on iichb0
iic0: <I2C generic I/O> on iicbus0
iichb1: <BCM2708/2835 BSC controller> mem 0x804000-0x80401f irq 61 on simplebus0
iicbus1: <OFW I2C bus> on iichb1
iic1: <I2C generic I/O> on iicbus1
spi0: <BCM2708/2835 SPI controller> mem 0x204000-0x20401f irq 62 on simplebus0
spibus0: <OFW SPI bus> on spi0
bcm_dma0: <BCM2835 DMA Controller> mem 0x7000-0x7fff,0xe05000-0xe05fff irq 24,25,26,27,28,29,30,31,32,33,34,35,36 on simplebus0
mbox0: <BCM2835 VideoCore Mailbox> mem 0xb880-0xb8bf irq 1 on simplebus0
sdhci_bcm0: <Broadcom 2708 SDHCI controller> mem 0x300000-0x3000ff irq 70 on simplebus0
mmc0: <MMC/SD bus> on sdhci_bcm0
uart0: <PrimeCell UART (PL011)> mem 0x201000-0x201fff irq 65 on simplebus0
uart0: console (115200,n,8,1)
dwcotg0: <DWC OTG 2.0 integrated USB controller> mem 0x980000-0x99ffff irq 17 on simplebus0
usbus0 on dwcotg0
fb0: <BCM2835 VT framebuffer driver> on ofwbus0
Timecounters tick every 10.000 msec
usbus0: 480Mbps High Speed USB v2.0
bcm2835_cpufreq0: ARM 700MHz, Core 250MHz, SDRAM 400MHz, Turbo OFF
ugen0.1: <DWCOTG> at usbus0
uhub0: <DWCOTG OTG Root HUB, class 9/0, rev 2.00/1.00, addr 1> on usbus0
mmcsd0: 32GB <SDHC 00000 1.0 SN 999A04A8 MFG 03/2015 by 27 SM> at mmc0 41.6MHz/4bit/65535-block
fb0: 640x480(0x0@0,0) 16bpp
fb0: pitch 1280, base 0x5eb64000, screen_size 614400
fbd0 on fb0
VT: initialize with new VT driver "fb".
random: unblocking device.
Root mount waiting for: usbus0
uhub0: 1 port with 1 removable, self powered
ugen0.2: <vendor 0x0b95> at usbus0
Trying to mount root from ufs:/dev/ufs/rootfs [rw]...
warning: no time-of-day clock registered, system time will not be set accurately
GEOM_PART: mmcsd0s2 was automatically resized.
  Use `gpart commit mmcsd0s2` to save changes or `gpart undo mmcsd0s2` to revert them.
axe0: <vendor 0x0b95 product 0x772b, rev 2.00/0.01, addr 2> on usbus0
miibus0: <MII bus> on axe0
ukphy0: <Generic IEEE 802.3u media interface> PHY 16 on miibus0
ukphy0:  none, 10baseT, 10baseT-FDX, 100baseTX, 100baseTX-FDX, auto, auto-flow
ue0: <USB Ethernet> on axe0
ue0: Ethernet address: 00:0e:c6:e0:0d:84
ue0: link state changed to DOWN
ue0: link state changed to UP
```

Thank you for reading.
