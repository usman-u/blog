---
title: "Hurricane Electric IPv6 Tunnel Broker On a Ubiquiti EdgeRouter"
date: 2022-03-11T15:01:49Z
draft: False
cover:
    image: "/images/he-v6/helogo.png"
    alt: "alt"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
ShowToc: true
TocOpen: true
---
## What is Tunnel Broker?
Tunnel Broker (provided by Hurricane Electric) is a service that allows users to connect the IPv6 internet, over IPv4. 

## How it works
It essentially works by establishing a GRE tunnel to one of Hurricane Electric's PoPs, via the IPv4 internet, and then running IPv6 ranges over it.

## Why Tunnel Broker?
### Accessibility
* I decided to setup Tunnel Broker, as my ISP doesn't support IPv6. 
* This would allow access to IPv6-only services on the internet.

### Hosting 

* Due to the abundance of IPv6 addresses, Tunnel Broker can provide multiple `/48` ranges.
* Publicly hosting services would be easier as NAT wouldn't be needed (my end devices would have publicly routable IPv6 addresses). 
* I wouldn't have to mess around with *port forwarding/destination NAT*. I would only have to modify WAN firewall rules, to allow inbound WAN traffic to my devices.

---

# Configuration
## Hurricane Electric's Side

HE's side of the configuration was straight forward. I made an account and created a regular tunnel. 

This involved providing my public IPv4 address, selecting a HE PoP location, and claiming the `/48` and `64` IPv6 ranges.
![HE Configuration](/images/he-v6/tunnel-example.png "HE Configuration")

*For the purposes of this article, I've replaced my real IPv4 and IPv6 ranges.*
### IP Addressing
For reference:
* HE's public IPv4 address - `1.1.1.1`
* HE's public IPv6 address - `aaa:aaa:aaa:aaa::1/64` - HE's address for their GRE tunnel interface.
* My client IPv4 address - `6.6.6.6`
* My client IPv6 address - `aaa:aaa:aaa:aaa::2/64` - my address for the local GRE tunnel interface.

Routed Ranges - IPv6 ranges that my end devices can use:

* My Routed `/64` - `ccc:ccc:ccc:ccc::/64`
* My Routed `/48` - `ddd:ddd:ddd::/48`

---

## EdgeRouter X Configuration

### Establishing the GRE Tunnel
```
set interfaces tunnel tun0 address 'aaa:aaa:aaa:aaa::2/64'
set interfaces tunnel tun0 description 'HE.NET IPv6 Tunnel'
set interfaces tunnel tun0 encapsulation sit
set interfaces tunnel tun0 local-ip 6.6.6.6
set interfaces tunnel tun0 multicast disable
set interfaces tunnel tun0 remote-ip 1.1.1.1
set interfaces tunnel tun0 ttl 255
set protocols static interface-route6 '::/0' next-hop-interface tun0
```

This:
* Creates a GRE tunnel called `tun0`.
* Sets the GRE encapsulation to SIT.
* Sets the local and remote tunnel endpoint IPs.
* Sets an IPv6 default interface route to `tun0`, so that all IPv6 internet traffic goes down the interface.   

### Testing the GRE Tunnel
To verify the GRE tunnel connectivity, I pinged the HE server endpoint (`aaa:aaa:aaa:aaa::1`).
```
usman@usman-erx:~$ ping6 aaa:aaa:aaa:aaa::1
PING aaa:aaa:aaa:aaa::1(aaa:aaa:aaa:aaa::1) 56 data bytes
64 bytes from aaa:aaa:aaa:aaa::1: icmp_seq=1 ttl=120 time=13.7 ms
64 bytes from aaa:aaa:aaa:aaa::1: icmp_seq=2 ttl=120 time=13.7 ms
64 bytes from aaa:aaa:aaa:aaa::1: icmp_seq=3 ttl=120 time=20.7 ms
64 bytes from aaa:aaa:aaa:aaa::1: icmp_seq=4 ttl=120 time=12.0 ms
--- aaa:aaa:aaa:aaa::1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3009ms
rtt min/avg/max/mdev = 12.038/15.082/20.748/3.347 ms
usman@usman-erx:~$
```
From my Edgerouter, I was able to ping the HE IPv6 server endpoint, over the GRE tunnel. 

An average of 11ms latency - which is more than acceptable.

---

I was also able to ping Google's IPv6 DNS server.

As per the IPv6 routing table, any unspecified traffic was routed over the `tun0` interface.
```
usman@usman-erx:~$ ping6 2001:4860:4860::8888
PING 2001:4860:4860::8888(2001:4860:4860::8888) 56 data bytes
64 bytes from 2001:4860:4860::8888: icmp_seq=1 ttl=120 time=19.4 ms
64 bytes from 2001:4860:4860::8888: icmp_seq=2 ttl=120 time=19.8 ms
64 bytes from 2001:4860:4860::8888: icmp_seq=3 ttl=120 time=13.8 ms
64 bytes from 2001:4860:4860::8888: icmp_seq=4 ttl=120 time=14.0 ms
^C
--- 2001:4860:4860::8888 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3009ms
rtt min/avg/max/mdev = 13.854/16.802/19.838/2.836 ms
```
```
usman@usman-erx:~$ show ipv6 route
IPv6 Routing Table
Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
       IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type 2, B - BGP
Timers: Uptime

IP Route Table for VRF "default"
S      ::/0 [1/0] via ::, tun0, 01w3d04h
C      ::1/128 via ::, lo, 02w1d22h
C      aaa:aaa:aaa:aaa::/64 via ::, tun0, 01w3d04h
C      fe80::/64 via ::, tun0, 01w3d04h
```

---

### IPv6 Router Advertisements
Once my EdgeRouter was IPv6 enabled, I set about connecting my end devices. I used router adverts to automatically assign the addresses my devices. 

It uses a `/64` IPv6 prefix (from my routable `/48`) and the client's MAC address, to create an IPv6 address.  This is a form of Stateless Address Auto Configuration (SLAAC).

I use *router on a stick VLANs*, where virtual interfaces are defined on my EdgeRouter. This made is easy to configure the RA.

E.g. For VLAN 20, I created a `/64` subnet out of my original `/48` - `ddd:ddd:ddd:20::/64`.
```
set interfaces switch switch0 vif 20 ipv6 address eui64 'ddd:ddd:ddd:20::/64'
set interfaces switch switch0 vif 20 ipv6 router-advert name-server '2001:4860:4860::8888'
set interfaces switch switch0 vif 20 ipv6 router-advert prefix 'ddd:ddd:ddd:20::/64'
```
This:
* Sets `vif 20` an ipv6 address, comprised of `ddd:ddd:ddd:20::/64` and `vif 20`'s MAC address.
* Sets Google's IPv6 DNS as the name server.
* Advertises the prefix to the clients.

`vif 20` was set a IPv6 address of `ddd:ddd:ddd:20:7683:c2ff:fef5:dadb/64` - which would be the default gateway for end devices on VLAN 20.
```
usman@usman-erx:~$ show interfaces
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface    IP Address                            S/L  Description
---------    ----------                            ---  -----------
switch0.20   10.0.20.1/24                          u/u  IoT
             ddd:ddd:ddd:20:7683:c2ff:fef5:dadb/64
[..]
```
### Verifying Connectivity On End Devices
Using SLAAC, my Debian based Raspberry Pi had been delegated the IPv6 address:

`inet6 ddd:ddd:ddd:20:e7d5:6b06:84cd:9ae7` from the `ddd:ddd:ddd:20::/64` prefix.
```
usman@raspberrypi:~$ sudo ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.10.14  netmask 255.255.255.0  broadcast 10.0.10.255
        inet6 fe80::e6ea:3e88:42f3:53e5  prefixlen 64  scopeid 0x20<link>
        inet6 ddd:ddd:ddd:20:e7d5:6b06:84cd:9ae7  prefixlen 64  scopeid 0x0<global>
        ether e4:5f:01:25:18:aa  txqueuelen 1000  (Ethernet)
        RX packets 667315319  bytes 748041609 (713.3 MiB)
        RX errors 108935  dropped 108935  overruns 0  frame 0
        TX packets 648634493  bytes 464678115 (443.1 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
usman@raspberrypi:~$
```
To verify connectivity, I also ran a traceroute to Google's IPv6 DNS server.

```
usman@raspberrypi:~$ traceroute6 2001:4860:4860::8888
traceroute to 2001:4860:4860::8888 (2001:4860:4860::8888), 30 hops max, 80 byte packets
 1  ddd:ddd:ddd:20:7683:c2ff:fef5:dadb (ddd:ddd:ddd:20:7683:c2ff:fef5:dadb)  0.512 ms  0.421 ms  0.580 ms
 2  tunnel718369.tunnel.tserv5.lon1.ipv6.he.net (2001:470:1f08:213::1)  19.324 ms  26.367 ms  17.280 ms
 3  e0-19.core2.lon2.he.net (2001:470:0:67::1)  17.346 ms  20.102 ms  22.073 ms
 4  2001:7f8:be::1:5169:1 (2001:7f8:be::1:5169:1)  43.789 ms  43.786 ms  43.788 ms
 5  2a00:1450:80f9::1 (2a00:1450:80f9::1)  26.844 ms 2a00:1450:8125::1 (2a00:1450:8125::1)  19.985 ms *
 6  dns.google (2001:4860:4860::8888)  25.651 ms  13.551 ms  17.539 ms
```
The traceroute confirmed that the traffic goes through my EdgeRouter's IPv6 interface `ddd:ddd:ddd:20:7683:c2ff:fef5:dadb`, over the GRE tunnel and HE's infrastructure, and out to Google.

---

### Firewalling

As my devices had publicly routable IPv6 addresses delegated to them, I added firewall rules to inbound limit traffic.

I created two firewall lists `WAN_IN` (from the internet to end devices) and `WAN_Local` (from the internet to the router).



```
set firewall ipv6-name WAN_In default-action drop
set firewall ipv6-name WAN_In rule 10 action accept
set firewall ipv6-name WAN_In rule 10 description 'allow established/related traffic'
set firewall ipv6-name WAN_In rule 10 protocol all
set firewall ipv6-name WAN_In rule 10 state established enable
set firewall ipv6-name WAN_In rule 10 state related enable

set firewall ipv6-name WAN_In rule 20 action drop
set firewall ipv6-name WAN_In rule 20 protocol all
set firewall ipv6-name WAN_In rule 20 state invalid enable

set firewall ipv6-name WAN_In rule 30 action accept
set firewall ipv6-name WAN_In rule 30 description 'allow icmpv6'
set firewall ipv6-name WAN_In rule 30 protocol icmpv6
```
```
set firewall ipv6-name WAN_Local default-action drop
set firewall ipv6-name WAN_Local rule 10 action accept
set firewall ipv6-name WAN_Local rule 10 description 'allow established/related traffic'
set firewall ipv6-name WAN_Local rule 10 protocol all
set firewall ipv6-name WAN_Local rule 10 state established enable
set firewall ipv6-name WAN_Local rule 10 state related enable

set firewall ipv6-name WAN_Local rule 20 action drop
set firewall ipv6-name WAN_Local rule 20 protocol all
set firewall ipv6-name WAN_Local rule 20 state invalid enable

set firewall ipv6-name WAN_Local rule 30 action accept
set firewall ipv6-name WAN_Local rule 30 description 'allow icmpv6'
set firewall ipv6-name WAN_Local rule 30 protocol icmpv6
```

Both firewall lists have 3 rules:
* Default action for inbound traffic is drop.
* Rule 10 - allows established and related traffic, from WAN to end devices.
* Rule 20 - drops invalid state packets.
* Rule 30 - Allows ICMPv6.


I then associated the rule lists with the GRE `tun0` interface, which filtered the traffic.
```
set interfaces tunnel tun0 firewall in ipv6-name WAN_In
set interfaces tunnel tun0 firewall local ipv6-name WAN_Local
```