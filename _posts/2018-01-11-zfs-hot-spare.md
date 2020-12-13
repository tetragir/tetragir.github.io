---
layout:	post
title:	ZFS Hot Spare with ZFSd
date: 2018-01-11 20:00:00
category: freebsd/zfs
tags: freebsd zfs zfsd hotspare
type: it
platform: FreeBSD 11.0
---

Since FreeBSD 11.0 was released, it's been possible to use a disk as a hot spare in a ZFS zpool. This means, that under a few circumstances, zfsd will take a predefined disk to replace another one.

* TOC
{:toc}

## Notes

This guide is written on, but not limited to FreeBSD. The commands should work on every other system where zfsd is present and should behave the same. I’m writing this guide and take the output from a FreeBSD 11.1 BETA-1 machine with virtual disks.

## The Spare

Usually a zpool contains identical disks. ZFS wont stop you from adding a smaller disk as a spare to a zpool, though it won’t be able to replace any disk with it. It is therefore highly recommended to add a disk which is identical, or has a greater capacity as the other disks in the zpool.

## Adding disks as spares 

One spare can be a part of more then one zpool. Once a disk fails in a zpool, it will be replaced with the spare. Understandably in this case, if we had only one spare, if one more disk fails, it cannot be replaced automatically. It is possible however to define more than one spare. After consideration, the spare disks can be added to an already existing pool. In the following example, we add two spare disks to two different already existing zpools.
We start with 2 datasets, each with 2 disks in a mirror 

```console
# zpool status tank1
  pool: tank1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	tank1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd3   ONLINE       0     0     0
	    vtbd4   ONLINE       0     0     0

errors: No known data errors
# zpool status tank2
  pool: tank2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	tank2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd5   ONLINE       0     0     0
	    vtbd6   ONLINE       0     0     0

errors: No known data errors
```

Adding two new disks as spares to both of the zpools:

```console
# zpool add tank1 spare vtbd1 vtbd2
# zpool add tank2 spare vtbd1 vtbd2
```

Both of the spares are showing up in the zpools.

```console
# zpool status tank1
  pool: tank1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	tank1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd3   ONLINE       0     0     0
	    vtbd4   ONLINE       0     0     0
	spares
	  vtbd1     AVAIL   
	  vtbd2     AVAIL   

errors: No known data errors
root@zfsd:/home/tetragir # zpool status tank2
  pool: tank2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	tank2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd5   ONLINE       0     0     0
	    vtbd6   ONLINE       0     0     0
	spares
	  vtbd1     AVAIL   
	  vtbd2     AVAIL   

errors: No known data errors
```

## Remove spares from datasets

It is also possible to remove the spares from the zpools, even if the pools are online. Make vtbd1 exclusive for tank1 and vtbd2 exclusive for tank2:

```console
# zpool remove tank1 vtbd2
# zpool remove tank2 vtbd1
```

This requires the drives not to be in use by any of the zpools.

``` console
# zpool status tank1
  pool: tank1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	tank1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd3   ONLINE       0     0     0
	    vtbd4   ONLINE       0     0     0
	spares
	  vtbd1     AVAIL   

errors: No known data errors
# zpool status tank2
  pool: tank2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	tank2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd5   ONLINE       0     0     0
	    vtbd6   ONLINE       0     0     0
	spares
	  vtbd2     AVAIL   

errors: No known data errors
```

Zfsd only cares about the zfs partition, so it will only replace the failed disk, or the zfs partition on the disk. If there were other partitions on the disk, those won’t be affected. For example if a partition is used on the disk as a root on zfs, the bootloader on the other partitions won’t be replicated to the spare. It is of course possible to put the bootloader on the spare at any time. But be careful! If that disk or partition is part of another zpool as a spare, the partition has to be at least as big as the other disks or partitions!
The replacement
In order to take any action, zfsd has to run. To start it on boot:

```console
# sysrc zfsd_enable=YES
zfsd_enable: NO -> YES
```
Start it right away:

```console
# service zfsd start
```

## Making sure it works

These two actions, adding a spare to a zpool and making sure that zfsd runs, are enough to initiate a replacement upon a disk failure, or because of other reasons.
In the following examples, we have a pool named tank with two mirrors and two spare disks:

```console
# zpool create tank mirror vtbd3 vtbd4 mirror vtbd5 vtbd6 spare vtbd1 vtbd2
# zpool status tank
  pool: tank
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	tank        ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd3   ONLINE       0     0     0
	    vtbd4   ONLINE       0     0     0
	  mirror-1  ONLINE       0     0     0
	    vtbd5   ONLINE       0     0     0
	    vtbd6   ONLINE       0     0     0
	spares
	  vtbd1     AVAIL   
	  vtbd2     AVAIL   

errors: No known data errors
```

Vtbd6 has failed and zfsd replaced it with a spare:

```console
# zpool status tank
  pool: tank
 state: DEGRADED
status: One or more devices could not be opened.  Sufficient replicas exist for
	the pool to continue functioning in a degraded state.
action: Attach the missing device and online it using 'zpool online'.
   see: http://illumos.org/msg/ZFS-8000-2Q
  scan: resilvered 51.5K in 0h0m with 0 errors on Sun Jun 18 11:38:34 2017
config:

	NAME                       STATE     READ WRITE CKSUM
	tank                       DEGRADED     0     0     0
	  mirror-0                 ONLINE       0     0     0
	    vtbd3                  ONLINE       0     0     0
	    vtbd4                  ONLINE       0     0     0
	  mirror-1                 DEGRADED     0     0     0
	    vtbd5                  ONLINE       0     0     0
	    spare-1                UNAVAIL      0     0     0
	      9567955613261044063  UNAVAIL      0     0     0  was /dev/vtbd6
	      vtbd1                ONLINE       0     0     0
	spares
	  11651124579463483589     INUSE     was /dev/vtbd1
	  vtbd2                    AVAIL   

errors: No known data errors
```

The pool is in *DEGRADED* mode, showing that a disk is missing, though currently the mirror is still redundant, thanks to the spare that replaced the failed disk.
Replacing the failed disk
If the autoreplace property is off (`# zpool get autoreplace tank`), - which is the default property -, and if a new device appears on the same interface, the replacement of the failed disk in zpool has to be initiated manually.

```console
# zpool replace tank 9567955613261044063 vtbd6
The disk is then resilvered and vtbd1 becomes available again.
# zpool status tank
  pool: tank
 state: ONLINE
  scan: resilvered 63.5K in 0h0m with 0 errors on Sun Jun 18 12:23:02 2017
config:

	NAME        STATE     READ WRITE CKSUM
	tank        ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    vtbd3   ONLINE       0     0     0
	    vtbd4   ONLINE       0     0     0
	  mirror-1  ONLINE       0     0     0
	    vtbd5   ONLINE       0     0     0
	    vtbd6   ONLINE       0     0     0
	spares
	  vtbd1     AVAIL   
	  vtbd2     AVAIL   

errors: No known data errors
```

In the case of autoreplace=on, the replacement disk will be automatically used, eliminating the need of the zpool replace step.
Setting the property:

```console
# zpool set autoreplace=on tank
# zpool get autoreplace tank
NAME  PROPERTY     VALUE    SOURCE
tank  autoreplace  on       local
```

## When a spare is activated

zfsd will activate a spare in any of the following cases:
- Device failure: if a device is removed (failed, or just disconnected)
- I/O Errors: If a device generates more than 50 I/O Errors in 60 seconds
- Checksum errors: If a device generates more than 50 checksum errors in 60 seconds
