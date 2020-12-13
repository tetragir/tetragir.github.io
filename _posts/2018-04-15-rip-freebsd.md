---
layout:	post
title:	RIP FreeBSD
date: 2018-04-15 12:00:00
category: networking
tags: networking freebsd routing rip tech
type: it
platform: FreeBSD 11.2
---

I mean Routing Information Protocol of course, not rest in peace! RIP is one of the most basic and easy-to-use routing protocols there is. It has its limitations, but it can be a viable solution for small environments. With RIP it is possible to exchange routing information between devices, and as it is a standard protocol, not only FreeBSD or Linux can use it, but it’s also implemented in many different networking devices, such as routers, firewalls, and even some layer 3 switches.

Table of contents:

* TOC
{:toc}


## Features and limitations
RIPv1 was introduced in 1988 and RIPv2 in 1993 and it became a standard in 1998. It is easy to use, but has a lot of limitations. The biggest such limitation is that the maximum hop count can only be 15, which means in practice that between the farthest two routers, there can be only 13 other routers. The routing domain can be of course bigger. RIPv2 has the ability to use authentication, however the only method for that is using plain passwords or MD5.

## Preparing FreeBSD
In order to send packets between different subnets it is essential to know how one can get to the other subnet. If many subnets have only one common router, this is quite easy, as every local subnet is known and every packet sent to an unknown subnet will be forwarded to another router (aka. to the default gateway).
The information about which network is where is stored in the routing table. The currently known subnets can be listed with the ```netstat``` command, where ```-r``` is telling netstat to give us the routing information, ```-n``` is not to resolve the IP addresses and ```-4``` is to do it for IPv4.

```console
root@router1# netstat -r4n
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.122.1      UGS      vtnet0
10.0.1.1           link#3             UH          lo1
127.0.0.1          link#2             UH          lo0
192.168.122.0/24   link#1             U        vtnet0
192.168.122.45     link#1             UHS         lo0
```

Each network has an associated cost in order to figure out which way is the best. In RIP, this cost is calculated from the number of hops, where one hop is one router. A directly connected subnet has a hop count of 0. With RIP, the whole routing table is sent to the neighbor which then processes it and sends the updated routing table to its neighbor, and so on.
In the following examples, there are two FreeBSD machines on the same subnet (*192.168.122.45/24* and *192.168.122.169/24*). Additionally, they have loopback interfaces:
- lo1 on router1 with the IP Address of **10.0.1.1/24**
- lo2 on router2 with the IP Address of **172.16.2.1/24**

Naturally not only loopback interfaces can be included in RIP, but other, physical or VLAN interfaces too.
Creating a loopback interface and the make it persistent between reboots:

```console
Router 1:
root@router1# ifconfig lo1 create inet 10.0.1.1/24
root@router1# sysrc cloned_interfaces=lo1
cloned_interfaces:  ->  lo1
root@router1# sysrc ifconfig_lo1="inet 10.0.1.1/24"
ifconfig_lo1:  -> inet 10.0.1.1/24
```

```console
Router 2:
root@router2# ifconfig lo2 create inet 172.16.2.1/24
root@router2# sysrc cloned_interfaces=lo2
cloned_interfaces:  -> tap2
root@router2# sysrc ifconfig_lo2="inet 172.16.2.1/24"
ifconfig_lo2:  -> inet 172.16.2.1/24
```

The loopback interfaces on each of the routers exists, though they have no information about the interface on the other machine.

Router1:

```console
root@router1# netstat -r4
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.122.1      UGS      vtnet0
10.0.1.1           link#3             UH          lo1
localhost          link#2             UH          lo0
192.168.122.0/24   link#1             U        vtnet0
router1            link#1             UHS         lo0
```

Router 2:

```console
root@router2# netstat -r4
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.122.1      UGS      vtnet0
localhost          link#2             UH          lo0
172.16.2.1         link#3             UH          lo2
192.168.122.0/24   link#1             U        vtnet0
router2            link#1             UHS         lo0
```

Pinging the loopback address on router2 from router1 fails, because router1 has no idea where to send the packets.

```console
root@router1# ping -v 172.16.2.1
PING 172.16.2.1 (172.16.2.1): 56 data bytes
^C
--- 172.16.2.1 ping statistics ---
7 packets transmitted, 0 packets received, 100.0% packet loss
```

In order to be able to reach the subnets behind the routers, the information about the networks has to be exchanged between the two routers.
First, enable routing, this makes sure that if router1 receives a packet on the interface with the IP *192.168.122.45*, it won’t drop it, but forward it to the loopback interface. Do this on all of the routers.

```console
root@router1# sysctl net.inet.ip.forwarding=1
root@router1# echo net.inet.ip.forwarding=1 >> /etc/sysctl.conf
```

## Starting routed.
RIP on FreeBSD is handled by [routed(8)](https://www.freebsd.org/cgi/man.cgi?query=routed&sektion=8).
We start routed with 2 flags:

- ```-s``` tells routed to advertise the known routes even if there is only one interface. This is necessary if we want to advertise loopback interfaces.
- ```-P ripv2``` tells routed to use RIPv2.

```console
root@router1/2# sysrc routed_enable=YES
routed_enable: NO -> YES
root@router1/2# sysrc routed_flags="-s -P ripv2"
routed_flags: -q -> -s -P ripv2
root@router1/2# service routed start
```

After around 30 seconds, **10.0.1.1** should be known on router2, and also **172.16.2.1** is known on router1.

```console
root@router1# netstat -r4n
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.122.1      UGS      vtnet0
10.0.1.1           link#3             UH          lo1
127.0.0.1          link#2             UH          lo0
172.16.2.1         192.168.122.169    UGH      vtnet0
192.168.122.0/24   link#1             U        vtnet0
192.168.122.45     link#1             UHS         lo0
```

```console
root@router2# netstat -r4n
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.122.1      UGS      vtnet0
10.0.1.1           192.168.122.45     UGH      vtnet0
127.0.0.1          link#2             UH          lo0
172.16.2.1         link#3             UH          lo2
192.168.122.0/24   link#1             U        vtnet0
192.168.122.169    link#1             UHS         lo0
```

With that, RIP is up and running on our routers. Even though MD5 is the only option for authentication, it is worth it to turn it on, and shortening the hold time from the default 30 minutes is also useful, but having a lot of options in rc.conf is not convenient. RouteD by default looks for the /etc/gateways file for parameters.

- ```md5_passwd=P4ssW0rd|1``` tells routed to use *P4ssW0rd* as the authentication password. RouteD will only accept routes if the incoming routing table uses this password with the key ID 1. There can be more then 1 password, with different key IDs.
- ```rdisc_interval=N``` tells routed how often to send the routing table over. The learned routes will stay in the routing table for 3*N.

```console
# echo “ripv2” >> /etc/gateways
# echo “md5_passwd=P4ssW0rd|1” >> /etc/gateways
# echo “rdisc_interval=10” >> /etc/gateways
```

Make sure that etc/gateway can be only read by root!

```console
# chmod 600 /etc/gateways
```

Remove the ripv2 parameter from rc.conf
```console
# sysrc routed_flags="-s"
routed_flags: -s -P ripv2 -> -s
```

Finally restart routed
```console
# service routed restart
Stopping routed.
Starting routed.
```

If the password matches, the route will still be propagated between the routers.

```console
root@router1# netstat -r4
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.122.1      UGS      vtnet0
10.0.1.1           link#3             UH          lo1
localhost          link#2             UH          lo0
172.16.2.1         router2            UGH      vtnet0
192.168.122.0/24   link#1             U        vtnet0
router1            link#1             UHS         lo0
```

```console
root@router2:/home/tetragir # netstat -r4
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.122.1      UGS      vtnet0
10.0.1.1           router1            UGH      vtnet0
localhost          link#2             UH          lo0
172.16.2.1         link#3             UH          lo2
192.168.122.0/24   link#1             U        vtnet0
router2            link#1             UHS         lo0
```

If the necessary routes are present on both routers, ping will work.

```console
root@router1:/home/tetragir # ping 172.16.2.1
PING 172.16.2.1 (172.16.2.1): 56 data bytes
64 bytes from 172.16.2.1: icmp_seq=0 ttl=64 time=0.433 ms
64 bytes from 172.16.2.1: icmp_seq=1 ttl=64 time=0.549 ms
64 bytes from 172.16.2.1: icmp_seq=2 ttl=64 time=0.499 ms
^C
--- 172.16.2.1 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.433/0.494/0.549/0.048 ms
```

## Security and alternatives
Using MD5 is not the best to secure RIP, but this is unfortunately the only way (other then plain passwords of course...). Malicious hosts can advertise routes with better hop count then the other, forcing traffic to flow to them.
In bigger networks, OSPF or even BGP can be used to propagate routes between routers.
