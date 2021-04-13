---
title: "Soft routing and home network configuration"
date: 2021-04-13T23:20:35+08:00
---

__Note: This article is mainly translated by Google translator, please turn to the Chinese version for more accurate expression if possible.__

I usually like to toss something at home. For many services, soft routing is an infrastructure. The concept of soft routing is opposite to hard routing. Hard routing refers to a device that forwards network packets by proprietary hardware, while soft routing is a device that forwards network packets by a general-purpose computer through software. Compared with hard routing at the same price, soft routing has better hardware and higher configurability, but it requires certain network knowledge to be effective. Let's briefly talk about the deployment and functions of soft routing.

# Equipment and system

Compared with ordinary computers, soft routers have several main features: more network ports, low power consumption, and small size. The soft routing device must have at least two network ports, one WAN port is used to connect to the public network, and one LAN port is connected to the local area network (but only one network port is enough for the bypass device). Because it needs 24 hours of uninterrupted operation, the power consumption is usually low and stable, and the size is small and convenient to be placed in a weak current box.

In terms of system, it is usually formed by adding customized software sources and related software on the basis of the Linux kernel. The more well-known ones are OpenWRT and Merlin. If you are brushing the system on a hard route, you need to see whether the hardware supports it. I bought NanoPi R2S as a soft router because the memory of the router I bought before was too low, and it came with a customized system, FriendlyWRT. The previous wireless router was only used as a wireless access point.

# Network topology

<center><embed src="topo.svg" type="image/svg+xml" /></center>

The above is the current network topology of my home. Unicom broadband optical fiber is connected to the optical modem. After modulation, the network cable is connected to the soft router WAN port. The LAN port is connected in series with two routers. The router has its own Mesh function. Turn off DHCP and use it as a pure wireless AP.

# Features

The device and network topology are introduced above, and the software deployment and the realization of various functions are discussed below.

## Network dialing

The optical modem of China Unicom's broadband installation comes with its own dial-up and wireless functions, but if the optical modem dials, it cannot use DDNS to penetrate the intranet. (At the beginning, because of the public IP and want DDNS, I tossed for a long time for soft routing dialing. After that, the public IP is gone, so here the optical modem is only used for modem, and the soft router is responsible for dialing. For the benefits of soft routing dialing, please refer to[Here](https://v2ex.com/t/681901#r_9123965) 。

There are two main things you need to do if you want to use soft routing to dial:

1.  Change the light cat to bridge mode
2.  Get broadband username and password

The first item differs depending on the device model and requires a lot of tricks. For the optical cat of China Unicom, you can refer to these two articles:

-   [Modified Unicom optical modem to bridge mode (Beijing area)](https://blog.51cto.com/hatech/2521226)
-   [Remember the experience of modifying Unicom Optical Cat to bridge mode](https://bygeek.cn/2019/08/18/Change-Optical-modem-to-bridge-mode/)

The second item needs to be remembered when installing broadband, or call China Unicom to reset the password.

Then configure the WAN port dial-up in the soft routing interface Network -> Interface.

## Global VPN

PassWall is a good plugin that supports socks, ss, ssr, v2ray, brook and trojan. You can directly subscribe to the airport link and update it regularly. There is also a load balancing function. Nodes of the same protocol can be combined together, and network traffic is automatically distributed by load balancing, which can make full use of multiple nodes in the airport.

There are already many tutorials on the specific configuration, so I won’t go into details here. The page of PassWall itself is also relatively intuitive.

### Host accelerator

If you want to use it for Switch, ordinary ladders are not enough. Delay and stability are both problems, and special accelerators are usually needed. I looked around and found[Greyhound](https://www.lingti.com/) There is a router plug-in. After downloading and running on the router, it can be controlled on the mobile phone. The Switch can enjoy the acceleration without any operation.

## Internet access

If there is a local service that needs to be exposed to the public network, it is usually necessary to bind the IP to the domain name (DNS resolution) and access it through the domain name. However, home broadband generally does not have a fixed IP or public network IP. At this time, DDNS is required to dynamically bind it. Set a changed public IP, or use external services for intranet penetration.

### DDNS

If the home broadband itself has a public network IP, then DDNS can be used to expose the service to the public network. The principle of DDNS is very simple. It is to keep obtaining your own IP. Once you find that the IP has changed, you should request a domain name resolution service provider to resolve a specific domain name to your current IP. In this way, even if the IP changes, you can pass Domain name access to the same service.

There are also some disadvantages to this. First of all, due to policy reasons, telecom operators in most areas of China do not open ports 80 and 443 of home broadband IP. This results in the services we build can only provide services through non-standard ports. Regarding non-standard ports My previous article[Use Caddy to enable HTTPS on ports other than 443](https://ssine.ink/posts/caddy-non-443-port-https/) Also mentioned. The second point is that DDNS only changes the DNS resolution after the IP has changed, which causes the service to be inaccessible during the period from the IP change to when the resolution takes effect.

### FRP intranet penetration

If there is no public IP, you can only find a way to penetrate the internal network. A more complete solution is FRP. You need a server with a public IP, bind the domain name to the server, and run the FRP service on the server. In this way, the FRP client on the soft route can connect to the FRP service, and supports the dynamic addition and deletion of the binding relationship between the domain name and the local IP on the client. If there is no independent server, there are many public service intranet penetration services that can be used online.

The disadvantages of FRP are that the delay will become higher and the need to purchase additional cloud hosts, but the stability of the service is much stronger than that of DDNS.

## Intranet static IP, DNS hijacking

Intranet services can be accessed directly using the intranet IP. For convenience, you can add a permanent fixed IP to the device at the soft-routed Network -> DHCP/DNS.

There is another interesting question. For example, I set up a cloud disk service on the intranet, located at 192.168.2.86, and bound it with the public domain name drive.abc.com through intranet penetration. Then access the cloud disk through drive.abc.com, the data packets will go to the local <->FRP server <->local route, which will increase the traffic of the FRP server and the bandwidth will also be restricted. It is indeed possible to access directly through the intranet IP 192.168.2.86, but in this way the cookie and other information in drive.abc.com cannot be shared, which is not elegant. In order to allow the same domain name to be directly connected to the internal network and use FRP on the public network, we can hijack the drive.abc.com domain name of the internal network and point it directly to 192.168.2.86, so that the above effect can be achieved, but this requires The client DNS points to the soft route. This is the default if the networked device does not have special settings. Just start dnsmasq and DNS redirection on the soft route. The function of setting a custom hijacked domain name is also on the Network -> DHCP/DNS page.
