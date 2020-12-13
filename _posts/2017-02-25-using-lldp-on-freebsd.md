---
layout:	post
title:	Using LLDP on FreeBSD
date: 2017-02-25 10:00:00
category: freebsd/networking
tags: freebsd networking lldp cdp cisco switching 802.3ab
type: it
platform: FreeBSD 11.0
---

LLDP, or Link Layer Discovery Protocol allows system administrators to easily map the network, eliminating the need to physically run the cables in a rack. LLDP is a protocol used to send and receive information about a neighboring device connected directly to a networking interface. It is similar to Cisco's CDP, Foundry's FDP, Nortel's SONMP, etc. It is a stateless protocol, meaning that an LLDP-enabled device sends advertisements even if the other side cannot do anything with it. 
In this guide the installation and configuration of the LLDP daemon on FreeBSD as well as on a Cisco switch will be introduced.

* TOC
{:toc}

## The LLDP Protocol
If you are already familiar with Cisco's CDP, LLDP won't surprise you. It is  built for the same purpose: to exchange device information between peers on a network. While CDP is a proprietary solution and can be used only on Cisco devices, LLDP is a standard: **IEEE 802.3AB**. Therefore it is implemented on many types of devices, such as switches, routers, various desktop operating systems, etc. LLDP helps a great deal in mapping the network topology, without spending hours in cabling cabinets to figure out which device is connected with which switchport. If LLDP is running on both the networking device and the server, it can show which port is connected where. Besides physical interfaces, LLDP can be used to exchange a lot more information, such as IP Address, hostname, etc.

## Installing lldpd on FreeBSD
In order to use LLDP on FreeBSD, [net-mgmt/lldpd](https://www.freshports.org/net-mgmt/lldpd) has to be installed. It can be installed from ports using portmaster:`#portmaster net-mgmt/lldpd`
Or from packages: `#pkg install net-mgmt/lldpd`
By default lldpd sends and receives all the information it can gather , so it is advisable to limit what we will communicate with the neighboring device.

If the machine has for example a lot of tap or bridge interfaces, lldpd will advertise all of them to the neighboring switch. Although this can be useful, it makes the output harder to read. It also sends and receives LLDP advertisements on all interfaces, including taps, loopback, wireless, etc.
lldpd can be told on which interfaces to run on, which interfaces to advertise, etc. The configuration file for lldpd is basically a list of commands as it is passed to lldpcli.
Create a file named `lldpd.conf` under `/usr/local/etc/`
The following configuration gives an example of how lldpd can be configured. For a full list of options, see `%man lldpcli`

~~~
# Hostname
configure system hostname "Voyager"

# Operating System
configure system description "FreeBSD 11.0-RELEASE"

# Interfaces to send LLDP advertisements on
configure system interface pattern em*,!bridge*,!tap*

# Management IP Address
configure system ip management pattern 10.0.42.120,!*:*

# Enable LLDP-MED
configure med fast-start enable

# Physical location
configure med location address country DE city "Frankfurt" street "Some Street" number "Number" name "Some Datacenter"
~~~

To check what is configured locally, run `#lldpcli show chassis detail`

![lldpci show chassis detail](/images/lldp/lldpcli_show_chassis.png){:class="w3-image"}

To see the neighbors run `#lldpcli show neighbors details`

![lldpcli show neighbors detail](/images/lldp/lldpcli_show_neighbors.png){:class="w3-image"}

In order to start lldpd on startup, run `#sysrc lldpd_enable=YES`

## Enabling LLDP on a Cisco Switch
Cisco devices usually run CDP by default with LLDP turned off. CDP and LLDP can coexist, but if CDP is not needed, turn it off: `Switch(config)#no cdp run `
Turn on LLDP: `Switch(config)#lldp run`
Checking the devices connected to the switches: `Switch#show lldp neighbors`

![show lldp neighbors](/images/lldp/manyneighbors.png){:class="w3-image"}

Under the first column, *Device ID*, the hostname of the device is shown without the domain.
The next column is called *Local Intf*. It shows to which local switchport is the device connected to.
*Hold-Time* shows how long the switch holds the information about the connected device. By default the devices are sending advertisements  every 30 seconds. If the switch is not getting any new advertisements in the configured hold time, it will delete the neighbor from the list.
The *Capability* column shows what exactly  are the connected devices capable of. This value is configurable on the host if we want to hide or specifically show something.
The last column is Port ID. This column displays how the neighbor calls the interface.
To list the configured LLDP parameters on the switches enter the following command: `#show lldp`

![show lldp](/images/lldp/shlldp.png){:class="w3-image"}

A lot more is advertised over LLDP, these details can be seen with the `#show lldp neighbors detail` command.

![show lldp neighbors detail](/images/lldp/sh_lld_nei_det.png){:class="w3-image"}

