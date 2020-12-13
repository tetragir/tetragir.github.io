---
layout:	post
title:	Basic Networking and FreeBSD
date: 2018-01-14 12:00:00
category: networking
tags: networking freebsd cisco vlan subnet layer2 layer3
type: it
platform: FreeBSD
---

In this article I would like to introduce the very basics of networking in order to  allow the reader to be able to separate VMs from each other or to organize them into logically different segments. This is far from a detailed description of networks, of course!

Table of contents:

* TOC
{:toc}

## IP Addresses and Netmasks
Networking is essential for the operation of a bigger environment. But what is a network? And what do we know about one?

A network is defined by it's IP Address and netmask and sometimes a VLAN ID. In order to define an IP Address range a netmask is used to say which bits in an IP Address are part of the network identifier and which are part of the host address. An IP Address contains 4 octets. Every IP Address can be translated into a bit stream.

Take this network for example: **172.16.1.0/26**, or **172.16.1.0** with netmask **255.255.255.192**
Translate it to binary: `10101100 . 0010000 . 00000001 . 00000000` with netmask `11111111 . 11111111. 11111111. 11000000`.
Basically the "1" in the netmask defines which bits in the IP Address are part of the network identifier and where the assignable IP Address space begins.

The first 26 bit is a `"1"` and therefore the first 26 bit in the IP Address is the identifier of the network itself. This means that every IP Address beginning with `10101100 . 0010000 . 00000001 . 00xxxxxx` belongs to one network.

The last 6 bits define the individual IP Address that are part of this network. In other words the first assignable addresses is: `10101100 . 0010000 . 00000001 . 000000000` and the last assignable address is: `10101100 . 0010000 . 00000001 . 00111111`. Translated back to decimal, the first assignable address is: **172.16.1.0** and the last one is **172.16.1.63**.

Subnetting could be calculated on paper, or there are subnet calculator tools out there, like: [http://www.subnet-calculator.com/](http://www.subnet-calculator.com/)

There are 2 "special" addresses in every network that can never be assigned to any host. These are the first and the last addresses. The first address identifies the network itself, the last one is the broadcast address. If a host sends a packet to the broadcast address, every other host in this particular subnet gets it and processes it.

To get back to our example, therefore the assignable addresses in this network are: **172.16.1.[1-62]**. It is important to always include the subnet mask with the IP Address because it can cause misunderstandings. For example, one host is configured with the IP Address **172.16.1.100/24** and another with **172.16.1.10/26**. As the network identifier and the broadcast addresses are standard and calculated from the IP Address and the subnet mask, the broadcast address will differ on these two hosts.

The first one will think, that the broadcast address is **172.16.1.255** while the second one will think it is **172.16.1.63**. So if the first host sends a broadcast message to the **172.16.1.255**, the second host will never process it, because this packet exists in another subnet from the second hosts point of view. Obviously, this is a problem. It is thus very important that every host in a network be configured with the same subnet mask.
 
 
Of course this does not mean that in one environment only one kind of a netmask can exist! The netmasks can be tailored to the purpose of the network given that the networks are not overlapping! For example, it is perfectly normal to use the following networks in one environment:

- 172.16.1.0/26 => Network ID is 172.16.1.0, broadcast is: 172.16.1.63
- 172.16.1.64/26 => Network ID is 172.16.1.64, broadcast is: 172.16.1.127
- 172.16.1.128/25 => Network ID is 172.16.1.128, broadcast is: 172.16.1.255
- 172.16.2.0/24 => Network ID is 172.16.2.0, broadcast is: 172.16.1.255

## IP Address ranges
The usable IPv4 Address range is divided and assigned for either private or public use. There are 3 ranges that can be used privately, either at home or in an enterprise environment. These ranges are not routed on the internet! These ranges are:

- 10.0.0.0/8
- 172.16.0.0/12
- 192.168.0.0/16

These address ranges can be divided into smaller subnets. A subnet is a part of a bigger network, for example the network 172.16.1.0/25 is a subnet of 172.16.1.0/24 because because every IP address in the 172.16.1.0/25 subnet is also part of the 172.16.1.0/24 network. Remember, that every host has to have the same netmask configured! Subnet only means, that a given network is a part of another. If the network 172.16.1.0/24 is divided into 2 subnet, it means that we have the following networks:

- 172.16.1.0/25
- 172.16.1.128/25

In order to send packets between these two subnets, a router is needed with an interface in both subnet.

There are more special ranges in IPv4, more about this on the Wikipedia: https://en.wikipedia.org/wiki/IPv4

## VLANs
VLAN is Virtual LAN. In a network where more than one subnet exists, VLANs are used to better separate the different networks from each other. A VLAN is basically an identifier assigned to a given subnet by the network administrator. The identifier number is limited by the software both on the computers or on the switches. For example the switch I use supports up to 255 VLANs and the identifier has to be between 1 and 4094. A few VLAN is pre-defined. VLAN 1 is always the default VLAN.
If no VLAN is configured on the hosts, that means every host is in VLAN 1. There are a few other VLANs that cannot be used, only for special purpuoses.
The VLAN assignment has to be consistent in the whole environment. In the following articles I will use the following networks:

- 172.16.1.0/26 => VLAN 10
- 172.16.2.0/24 => VLAN 20

The VLAN identifier could be completely different from the network identifier, it is only useful if from a VLAN identifier the network could be determined without using the documentation.

On one single cable, more than one VLAN could be present. In order to achieve that, both interfaces has to be configured to support VLAN tags. If one side sends a packet, the other side has to know which VLAN it belongs to. Therefore if a packet is sent in VLAN 20, the VLAN Identifier will be included in the packet, so the other side knows that the given packet belongs to that speicific VLAN. On one single cable, one untagged VLAN could exist, this called the native VLAN. That means that if both sides are configured with the same native VLAN, both side knows, that if a packet arrives without any VLAN tag, it belongs to that specific VLAN. In the following articles the FreeBSD machine which hosts the virtual machines has an interface both in VLAN 10 and VLAN 20. If native VLANs are in use, be sure to configure the same VLAN on both sides!

My network is configured so, that if a packet is being sent in VLAN 10, it is not tagged, but if it belongs to VLAN 20, a VLAN tag is included in the packets.

## The first 3 layers of the OSI Model
the OSI Model describes basically how communication works on a network. There are 8 layers in the OSI Model, I only cover the first 3 as those are needed to understand why we need a router or a switch. I won't cover the OSI model in depth, only so that one can understand what is the difference between a Layer 2 or 3 device. More on Cisco's website: http://docwiki.cisco.com/wiki/Internetworking_Basics
Layer 1 is the physical layer. The function of this layer is to define how data is physically transmitted between devices. A common standard used nowadays is the Ethernet Physical Layer. It defines the physical connector used on the cables, the voltage levels, the speed, basically everything physical. This is the IEEE 802.3 standard. Most common form of this standard is a cable with 8P8C connector, or RJ45. This connector is found on PCs, Notebooks, switches, servers, etc. The "quality" of the cable is also defined. This could be Cat-5, Cat-5e, Cat-6 or other. This defines how the individual cables are shielded (or not).
The commonly used optical connections are also ethernet standards.
A typical example of a Layer 1 networking device is a hub. Hubs usually has more interfaces. If the bridge receives a bit on one interface, it forwards that to every other interface without looking into the packet. There are no addressing in layer 1. Fortunately hubs died out long ago.

There are other standards defining the physical layer of the OSI Model, for example USB, Bluetooth, many IEEE 802.11 standards for Wi-Fi, etc.
The second layer called Data Link. This layer is to define, what we communicate if a Layer 1 connection is already negotiated. Ethernet is also used in Layer 2. It defines how a packet looks like, what should it contain, etc.
In layer 2, addressing exists, these are the MAC Addresses. By design a MAC Address is unique in the whole world. It is although not entirely true as there are a limited number of MAC Address available. Therefore MAC Addresses are reused by manufacturers on computers that has very little chance happening to be on one network, for example 2 interface from the same manufacturer with identical MAC Addresses could be sold in different continents.

A typical Layer 2 device is a switch. It can send frames based on the sender and receiver MAC Address. If the receiver is known by the switch, it only sends frames on a specific cable where the receiver is connected. If the receiver is not known, the switch will send out the packet on all of the interfaces belonging to the same VLAN as the sender, and record the MAC into the MAC Address Table only when an answer is received.
It is Layer 3 where things are getting interesting. An IP Address for example a Layer 3 address. A switch is not able to route between networks, it operates only in Layer 2 (yes, there are several Layer 3 switches, more on that later). In order to route between the subnets, a router is needed. A router is basically a device that can have more than 1 interface in different subnets. This could either be different physical interfaces for all of the subnets, or VLAN interfaces.

## Switching and Routing
As mentioned before, switches and routers are needed to operate a network. The difference between them is what are the used for. 

A switch is used to share a network between hosts and make it possible that all of the hosts in the same network, or VLAN are able to communicate with each other. From the switches perspective, a router is just a host. When a host is sending packets to a router the switch forwards them to the router. The router then forwards it to the desired network.
There are Layer 3 switches which makes networking and routing easy (and cheap). On a layer 3 switch different VLAN Interfaces could coexist and it is able to forward packets between those VLANs. Altough layer 3 switches can route between subnets, a router could do more. Usually a layer 3 switch doesn't know higher level routing protocols, such as OSFP or BGP to begin with.

## Configuring a Cisco switch to support VLANs
In this guide, I use a Cisco switch. It should be similar to configure VLANs on other switches, always refer to the manufacturer's documentation.

The Cisco switches CLI interface is accessible 
from a console interface, Telnet or SSH. Use console for the initial configuration and the switch to SSH. Telnet is not secure, it transmits the passwords, commands unencrypted. Always use SSH. After login, enter global configuration mode:

```console
Switch2#configure terminal
Switch2(config)#
```

Create the VLAN

```console
Switch2(config)#vlan 10
```

The VLAN itself is created, it is useful to name it

```console
Switch2(config-vlan)#name Home
```

In order to check if these commands are executed, exit global configuration mode and check the VLAN:

```console
Switch2(config-vlan)#end
Switch2#show vlan id 10
```

The output will be similar:

```console
Switch2#show vlan id 10

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
10   Home                             active    

VLAN Type  SAID       MTU   Parent RingNo BridgeNo Stp  BrdgMode Trans1 Trans2
---- ----- ---------- ----- ------ ------ -------- ---- -------- ------ ------
10   enet  100010     1500  -      -      -        -    -        0      0   

Remote SPAN VLAN
----------------
Disabled

Primary Secondary Type              Ports
------- --------- ----------------- ------------------------------------------
```

Create a second VLAN:

```console
Switch2#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch2(config)#vlan 20
Switch2(config-vlan)#name Guest 
Switch2(config-vlan)#end
Switch2#sh vlan id 20

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
20   Guest                            active    

VLAN Type  SAID       MTU   Parent RingNo BridgeNo Stp  BrdgMode Trans1 Trans2
---- ----- ---------- ----- ------ ------ -------- ---- -------- ------ ------
20   enet  100020     1500  -      -      -        -    -        0      0   

Remote SPAN VLAN
----------------
Disabled

Primary Secondary Type              Ports
------- --------- ----------------- ------------------------------------------
```

Assign the the 2 VLANs created before to a phisycal interface. VLAN 10 will be the native VLAN. I define the VLANs which are alloved on the wire, this way if the switch gets a packet other then VLAN 10 or 20, the packet will be dropped. Tailer the commands to the interfaces accordingly, I use FastEthernet0/2 on the switch:

```console
Switch2#conf t
Switch2(config)#interface FastEthernet0/2
Switch2(config-if)#switchport trunk native vlan 10
Switch2(config-if)#switchport trunk allowed vlan 10,20
Switch2(config-if)#switchport mode trunk
Switch2(config-if)#end
```

Check if the commands are executed:

```console
Switch2#show interfaces FastEthernet 0/2 trunk 

Port        Mode             Encapsulation  Status        Native vlan
Fa0/2       on               802.1q         trunking      10

Port        Vlans allowed on trunk
Fa0/2       10,20

Port        Vlans allowed and active in management domain
Fa0/2       10,20

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/2       10,20
```

If the result is satisfactory, save the configuration

```console
Switch2#copy running-config startup-config
Destination filename [startup-config]? 
Building configuration...
[OK]
Switch2#
```

Next step is to configure the FreeBSD machine connected to the port configured above.

## Configuring an interface on FreeBSD to support VLANs

On my machine, the physical interface is em0, all the commands are referenced to em0. Check the physical interface with ifconfig.

```console
% ifconfig
em0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=4219b<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,VLAN_HWCSUM,TSO4,WOL_MAGIC,VLAN_HWTSO>
	ether 18:03:73:c7:17:14
	inet 172.16.1.11 netmask 0xffffffc0 broadcast 172.16.1.63 
	nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
	media: Ethernet autoselect (100baseTX <full-duplex>)
	status: active
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=600003<RXCSUM,TXCSUM,RXCSUM_IPV6,TXCSUM_IPV6>
	inet6 ::1 prefixlen 128 
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2 
	inet 127.0.0.1 netmask 0xff000000 
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
	groups: lo 
```

If the physical interface differs, tailer the commands accordingly!
em0 has already got an IP address. As VLAN 10 is the native VLAN to this machine, without any further configuration, this machine will operate in VLAN 10 without knowing it.
Creating VLAN 20 and adding an IP Address to it:

```console
# ifconfig em0.20 create vlan 20 vlandev em0 inet 172.16.2.10/24
```

This command will create and interface, em0.20, assign it to the physical interface, em0 and use tagging. Check the results with ifconfig:

```console
# ifconfig
em0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=4219b<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,VLAN_HWCSUM,TSO4,WOL_MAGIC,VLAN_HWTSO>
	ether 18:03:73:c7:17:14
	inet 172.16.1.11 netmask 0xffffffc0 broadcast 172.16.1.63 
	nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
	media: Ethernet autoselect (100baseTX <full-duplex>)
	status: active
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=600003<RXCSUM,TXCSUM,RXCSUM_IPV6,TXCSUM_IPV6>
	inet6 ::1 prefixlen 128 
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2 
	inet 127.0.0.1 netmask 0xff000000 
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
	groups: lo 
em0.20: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=103<RXCSUM,TXCSUM,TSO4>
	ether 18:03:73:c7:17:14
	inet 172.16.2.10 netmask 0xffffff00 broadcast 172.16.2.255 
	inet6 fe80::1a03:73ff:fec7:1714%em0.20 prefixlen 64 scopeid 0x3 
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
	media: Ethernet autoselect (100baseTX <full-duplex>)
	status: active
	vlan: 20 vlanpcp: 0 parent interface: em0
	groups: vlan 
```

In order to preserve the VLAN between reboots, add the following lines to /etc/rc.conf:

```console
vlans_em0="20"
ifconfig_em0_20="inet 172.16.2.10/24"
```

If another host exists in VLAN 20, use ping to prove it works:

```console
% ping 172.16.2.1
PING 172.16.2.1 (172.16.2.1): 56 data bytes
64 bytes from 172.16.2.1: icmp_seq=0 ttl=64 time=0.425 ms
```

Specify the source interface to be sure the packets are not sourced from VLAN 10 and then routed by a router:

```console
% ping -S 172.16.2.10 172.16.2.1
PING 172.16.2.1 (172.16.2.1) from 172.16.2.10: 56 data bytes
64 bytes from 172.16.2.1: icmp_seq=0 ttl=64 time=0.455 ms
```

The first ping is okay to be lost, but after that there should be an answer for every ping request.
