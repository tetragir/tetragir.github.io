---
layout:	post
title:	Running OpenVPN client in a FreeNAS Jail
date: 2019-06-20 16:00
category: networking
tags: freenas freebsd openvpn vpn networking
type: it
platform: FreeNAS 11.2
---

Lately I have had less free time, which has resulted in the fact that this is the first article in over a year. Also I’ve been using mostly CentOS at my job and FreeBSD got out of the focus. But that doesn’t mean that I abandoned FreeBSD, just that solutions that require less time to set up and maintain got my attention.

So, I wanted to replace FreeBSD with FreeNAS on my home NAS as well as on my parents’. Both systems are replicating over any new data overnight so in case any of them breaks, we still have our data. I knew I wanted use the built-in solution in FreeNAS which only requires an SSH connection to the other NAS. I was also sure, that even though SSH is safe, I don’t want to replicate data over plain internet. This would also require a fix public IP address on both sides which is not easy with home commercial internet connections. A workaround would be to use dynamic DNS like duckdns or dyndns, but it still requires setting up NAT on the router not to mention exposing your FreeNAS to the public internet (in my experience, most ISP-provided routers do not have a firewall or it’s so dumb that it cannot filter based on source addresses).

Long story short, I needed VPN. I have a fix IP ar my home, which makes setup more easy, but this can be also achieved with a cheap VPS running somewhere, acting as a relay.

* TOC
{:toc}

## Requirements
As I mentioned, you need at least one fix IP address somewhere which will be used by the OpenVPN Server. It can be anything, as OpenVPN can run on different platforms. This guide assumes that an OpenVPN Server is already set up.

## The OpenVPN Server
This guide won’t go into details of setting up the OpenVPN Server, but I’ll mention the configuration options that are needed in order to make this connection work.

The easiest and safest authentication method for OpenVPN is to use certificates. If a client is trying to connect, the server will check the clients certificate against its CA. If it’s a match, the client is authenticated. The client can/will also check the server’s certificate in order to make sure that it connects to a valid server. There are a lot of guides out there that explain how to set up a PKI (Private Key Infrastructure) with easyrsa (which is installed along with OpenVPN) or other tools.

The routes of the client FreeNAS’s network need to be configured in the OpenVPN server. You can use this tool to find out the network and subnet mask: [http://www.subnet-calculator.com/](http://www.subnet-calculator.com/)
Please note, that the location of the openvpn configuration file could be different.  Here I’m using Debian.
Once you have it, add it to the server’s configuration:

```console
route 192.168.0.0 255.255.255.0
```

Besides this, the server needs to know over which client this subnet is reachable. This is achieved with client specific configurations in the Client Config Directory (CCD). If there is none yet, define the CCD location as follows:

```console
client-config-dir /etc/openvpn/ccd
```

Inside this directory, there should be a file corresponding to the client’s name (defined in its certificate). The content of the file is:

```console
# cat /etc/openvpn/ccd/client
iroute 192.168.0.0 255.255.255.0
```

You may notice that this is the same subnet defined in the OpenVPN Server configuration file. In the main configuration file, we define that a subnet will be handled on the system by OpenVPN. But because the client will not communicate to the server that this specific subnet is reachable through it, we need to tell the OpenVPN server. This way, once the connection is estbalished, a new route will be added to the operating system’s routing table pointing to the correct client.

The client also needs to be informed about the server’s subnet. Add the subnet you want to push to the client:

```console
push "route 10.0.1.0 255.255.255.0"
```

Once the client is connected, it will be informed about this route.

## FreeNAS Settings
Creating a Jail for OpenVPN is mostly straightforward, but there are a few gotchas.
Create a new Jail by navigating to **Jails > Add** and continue with **Advanced Jail Creation**. Assign a name and IP address to the Jail.
It is important to check the box to **VNET**! The Jail has to use VNET in order to make this work.
Open Custom Properties and check **allow_tun**.
Besides this, as we will route traffic through the OpenVPN Jail, we need to tell FreeNAS how that server is reachable. Open **Network > Static Routes** and then **Add**. Enter the *Destination* (which is the network we would like to reach this FreeNAS from, the gateway (which is the IP address of the newly created Jail) and a description.

## OpenVPN Setup in the Jail
Once the Jail is running, enter it through the CLI (either by SSH into the NAS or using the Shell on the FreeNAS GUI and then iocage console jailname).
Install openvpn

```console
pkg install openvpn
```

(or install through ports if you prefer that way). After installation, either copy an OpenVPN configuration file that is already in use or set up a new one. Copy a template:

```console
cp /usr/local/share/examples/openvpn/sample-config-files/client.conf /usr/local/etc/openvpn/
```

Modify the configuration file as needed (most importantly the remote part and the certificates). You can use the following as a template:

```console
client #We are a client
dev tun #Using TUN instead of TAP
proto udp #Using UDP
remote your.doma.in 1194 #This can be an IP address and a different port as well
resolv-retry infinite #If we are connecting to a domain name, try to resolve it infinitely
nobind #No need to bind
persist-key #Reserve states between restarts
persist-tun
user nobody #Drop privilege
group nobody
ca /path/to/ca.crt #Path to the Certificate Authority
cert /path/to/client.crt #Path to the certificate used by this client, signed by the CA
key /path/to/client.key #Path to the key belonging to the certificate
remote-cert-tls server #Check the server certificate if the certificates were set up for this
```

Please note, that this is just a template, you need to modify it to reflect your actual setup.
After all files are in place including the configuration and the certificates, you can try to connect to the server.

```console
# openvpn path/to/the/configuration.conf
```

If the connection is unsuccessful, you will see an explanation in the output.

## Test
If the client is successfully connected, check if the routing is done correctly.
On the client, the subnet of the server should be there:

```console
root@client:~ # netstat -r4
Routing tables

Internet:
Destination        Gateway            Flags     Netif Expire
default            192.168.0.1	   UGS     epair0b
10.8.0.0/24        10.8.0.1           UGS        tun0
10.8.0.1           link#3             UH         tun0
10.8.0.2           link#3             UHS         lo0
localhost          link#1             UH          lo0
10.0.1.0/24	       10.8.0.1           UGS        tun0
```

The route 10.0.1.0/24 was pushed from the server. On the server, check if the client’s subnet is present.

```console
root@server:~ # netstat -r4
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         10.0.1.1        0.0.0.0         UG        0 0          0 eth0
10.0.100.0      0.0.0.0         255.255.255.240 U         0 0          0 eth0
10.8.0.0        0.0.0.0         255.255.255.0   U         0 0          0 tun0
192.168.0.0     10.8.0.2        255.255.255.0   UG        0 0          0 tun0
```

If both are correct, we should be able to ping the client from the server over the VPN tunnel.

```console
root@server:~# ping -S 10.0.1.2 192.168.0.2
PING 192.168.0.9 (192.168.0.9) 56(84) bytes of data.
64 bytes from 192.168.0.9: icmp_seq=1 ttl=64 time=43.7 ms
64 bytes from 192.168.0.9: icmp_seq=2 ttl=64 time=41.3 ms
```

```-S``` defines the source IP address, in this case this is the local interface of the server.
If the route was successfully added on the FreeNAS, ping should be also successful to its local IP address.
If you need to reach the FreeNAS from a machine other then the VPN server, you either need to add a static route on the other machine or the same on your default gateway.

# It’s done
This is a fairly easy setup and makes sure that the traffic between you and your FreeNAS installation is secure over an OpenVPN tunnel. It is also possible to use a cheap VPS on DigitalOcean, Vultr or any other provider if you want to reach FreeNAS from a location that does not have a fix IP or is behind CGNAT (Carrier Grade NAT).
