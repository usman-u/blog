---
title: "DN42 Part 2: Conecting an AWS VPS to DN42, using iBGP and WireGuard."
date: 2022-01-10T20:58:53Z
draft: false
cover:
    image: "/images/iBGP-AWS/map.png"
    caption: "Overview"
    alt: "alt"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
ShowToc: true
TocOpen: false
---
I recently connected to the DN42 BGP mesh, a big network which employs WAN technologies to create an internet like mesh. 
[*Read my first most for more info on DN42.*](https://blog.usman.network/posts/dn42-bgp/)

In this post, I'll go over how I:
* Provisioned an AWS Linux VPS.
* Created a WireGuard Site2Site VPN Connection between my Ubiquiti EdgeRouter and VPS.
* Utilized the BIRD internet routing daemon to handle internal BGP routing over that VPN connection. 

The end goal was to have the AWS VPS be able to access the DN42 mesh (using iBGP) via my EdgeRouter, and then through the EdgeRouter's 5 eBGP peerings (see  diagram).

## Provisioning the AWS VPS

![The Cloud Business](/images/iBGP-AWS/cloudmeme.jpg "The Cloud Business")

After creating an AWS free tier account, I chose to use a new Debian Instance. Any modern Linux distribution (with BIRD, WireGuard and a Public IP address) should suffice.

![Debian Image](/images/iBGP-AWS/debian.PNG "Debian Image")

I selected the free tier eligible `t2.micro`. It comes with a 2.5Ghz vCPU, 1GB of RAM and 7GB of storage, which is plenty for a BGP router with a route table the size of DN42's (~600 prefixes).
![Debian Image](/images/iBGP-AWS/debian-free-tier.PNG "Debian Image")


One I launched the VPS, I connected to it using my generated AWS SSH private key.
```
ssh -i  C:\Users\Usman\.ssh\AWS_Access.pem admin@AWS-VPS
```

```
root@AWS-VPS:/home/admin# neofetch
       _,met$$$$$gg.          root@AWS-VPS
    ,g$$$$$$$$$$$$$$$P.       --------------------
  ,g$$P"     """Y$$.".        OS: Debian GNU/Linux 10 (buster) x86_64
 ,$$P'              `$$$.     Host: HVM domU 4.2.amazon
',$$P       ,ggs.     `$$b:   Kernel: 4.19.0-18-cloud-amd64
`d$$'     ,$P"'   .    $$$    Uptime: 16 hours
 $$P      d$'     ,    $$P    Packages: 452 (dpkg)
 $$:      $$.   -    ,d$$'    Shell: bash 5.0.3
 $$;      Y$b._   _,d$P'      CPU: Intel Xeon E5-2686 v4 (1) @ 2.300GHz
 Y$$.    `.`"Y$$$$P"'         GPU: Cirrus Logic GD 5446
 `$$b      "-.__              Memory: 74MiB / 987MiB
  `Y$$
   `Y$$.
     `$$b.
       `Y$$b.
          `"Y$b._
              `"""
```

## Installing the required packages

Next, I upgraded the system packages, installed WireGuard and the BIRD routing daemon.
```
root@AWS-VPS:~$ sudo apt update && sudo apt upgrade -y
root@AWS-VPS:~$ sudo apt install wireguard
root@AWS-VPS:~$ sudo apt install bird
```


---

## Establishing the WireGuard VPN Tunnel
### Generating Key Pairs
After the installation, I generated WireGuard Public and Private Key Pairs, on my EdgeRouter and VPS. Where each peer has public/private key pair, and only needs to know the peer's public key.

On the EdgeRouter:
```
usman@ER-X:~$ mkdir /home/usman/wgconfs/AWS
usman@ER-X:~$ cd /home/usman/wgconfs/AWS
usman@ER-X:~/wgconfs/AWS$ wg genkey | tee privatekey | wg pubkey > publickey
usman@ER-X:~/wgconfs/AWS$ ls
private  public
```

On the VPS:
```
root@AWS-VPS:~$ cd /etc/wireguard/
root@AWS-VPS:/etc/wireguard$ wg genkey | tee privatekey | wg pubkey > publickey
usman@AWS-VPS:~/etc/wireguard$ ls
private  public
```
---

## WireGuard on the EdgeRouter
```
set interfaces wireguard wg67 address 10.132.0.2/30
set interfaces wireguard wg67 mtu 1420
set interfaces wireguard wg67 peer LHv3equY4NyDeLVCY1+P5IrCA23+bYLixB18ehHjzzM= allowed-ips 0.0.0.0/0
set interfaces wireguard wg67 peer LHv3equY4NyDeLVCY1+P5IrCA23+bYLixB18ehHjzzM= endpoint 'aws.vps.com:51823'
set interfaces wireguard wg67 private-key /home/usman/wgconfs/privatekey
set interfaces wireguard wg67 route-allowed-ips false
```
What this does:
* Creates a WireGuard Interface, called `wg67 `with an IP address of `10.132.0.2`
* Sets the MTU to 1420
* Creates a peer with the public key of `LHv3equY4NyDeLVCY1+P5IrCA23+bYLixB18ehHjzzM=`
* Allows any ip address to flow over the tunnel.
* Specifies the public IP endpoint of the AWS VPS.
* Associates the previously generated private key with the wg67 interface.
* Set the `route-allowed-ips` value to false, as that would create a default route via the VPS.


### Allowing WireGuard through the EdgeRouter's firewall
```
set firewall name WAN_LOCAL rule 25 action accept
set firewall name WAN_LOCAL rule 25 description iBGP-AWS
set firewall name WAN_LOCAL rule 25 destination port 51823
set firewall name WAN_LOCAL rule 25 log enable
set firewall name WAN_LOCAL rule 25 protocol udp
```
What this does:
* Creates an accept rule in the WAN_LOCAL rule set, which regulates traffic from the WAN interface, to the router.
* Accepts all incoming UDP connections on port 51823, from the AWS VPS.

---

## WireGuard on the AWS Debian VPS
WireGuard on Linux is similar to the EdgeRouter's config.

```
ip link add dev wg0 type wireguard
ip address add dev wg0 10.132.0.1/30
wg set wg0 listen-port 51823 private-key /etc/wireguard/privatekey peer <erxpubkey> allowed-ips 0.0.0.0/0 endpoint <erx.pub.ip>:51823
ip link set up dev wg0

ip addr add 172.22.132.169/32 dev lo
```


What this does:
* Creates an interface called `wg0`, with the type wireguard.
* Assigns the address `10.132.0.1/30` to the `wg0` interface.
* Sets the server listen port to 51823.
* Sets the privatekey to the location of the previously generated private key.
* Allows any ip address over the tunnel.
* Specifies the public IP and port of the EdgeRouter peer.


### Allowing WireGuard through AWSs Security Group

I also modified the VPS's security group, so that it allows incoming udp connections on port 51823, from the EdgeRouter.

![AWS Security Group](/images/iBGP-AWS/Security-Group.PNG "AWS Security Group")

---

### Testing Reachability

After configuring the Firewall Rules, I was able to ping between the VPS and EdgeRouter, over the WireGuard VPN Interface.

From the EdgeRouter, to the VPS:
```
usman@ER-X:~$ ping 10.132.0.1
PING 10.132.0.1 (10.132.0.1) 56(84) bytes of data.
64 bytes from 10.132.0.1: icmp_seq=1 ttl=64 time=31.4 ms
64 bytes from 10.132.0.1: icmp_seq=2 ttl=64 time=31.5 ms
64 bytes from 10.132.0.1: icmp_seq=3 ttl=64 time=35.0 ms
64 bytes from 10.132.0.1: icmp_seq=4 ttl=64 time=34.3 ms
64 bytes from 10.132.0.1: icmp_seq=5 ttl=64 time=31.8 ms
```
  
From the VPS to the EdgeRouter:
```
admin@AWS-VPS:~$ ping 10.132.0.2
PING 10.132.0.2 (10.132.0.2) 56(84) bytes of data.
64 bytes from 10.132.0.2: icmp_seq=1 ttl=64 time=39.4 ms
64 bytes from 10.132.0.2: icmp_seq=2 ttl=64 time=34.7 ms
64 bytes from 10.132.0.2: icmp_seq=3 ttl=64 time=34.8 ms
64 bytes from 10.132.0.2: icmp_seq=4 ttl=64 time=29.5 ms
64 bytes from 10.132.0.2: icmp_seq=5 ttl=64 time=28.9 ms
```

The latency ranges from 30-40 ms, which perfectly acceptable for a VPN connection over the internet.

WireGuard also provides some diagnostic data (e.g. data usage, latest handshake etc.).

On the EdgeRouter:

```
usman@ER-X:~$ sudo wg
interface: wg67
  public key: P1FrJ1IpHCFX3Tk/4aFxbrCbD4VDefwHaieVWDWoXx0=
  private key: (hidden)
  listening port: 51650

peer: LHv3equY4NyDeLVCY1+P5IrCA23+bYLixB18ehHjzzM=
  endpoint: AWS-pub.ip:51823
  allowed ips: 0.0.0.0/0
  latest handshake: 46 seconds ago
  transfer: 252.68 MiB received, 30.28 MiB sent
```

On the VPS:
```
root@AWS-VPS:~# wg
interface: wg67
  public key: LHv3equY4NyDeLVCY1+P5IrCA23+bYLixB18ehHjzzM=
  private key: (hidden)
  listening port: 51823

peer: P1FrJ1IpHCFX3Tk/4aFxbrCbD4VDefwHaieVWDWoXx0=
  endpoint: edgerouter-pub.ip:51650
  allowed ips: 0.0.0.0/0
  latest handshake: 46 seconds ago
  transfer: 30.21 MiB received, 252.70 MiB sent
```
---

## iBGP Configuration

Once the VPN was up, I set about creating a internal BGP peering between the EdgeRouter and VPS (so that the VPS could reach DN42, via the EdgeRouter's upstream transit peers).

![BIRD](/images/iBGP-AWS/bird.png "BIRD")

### Why BIRD Internet Routing Daemon?
I chose to use the *free and open-source* [BIRD routing daemon](https://bird.network.cz/) as its widely used in the DN42 mesh and on the real internet. 

Netflix proudly uses BIRD to run their [global CDN backbone](https://openconnect.netflix.com/en_gb/appliances/#software), which handles ~[100Tbps](https://freebsdfoundation.org/wp-content/uploads/2020/10/netflixcasestudy_final.pdf) at peak. So its probably fine for DN42's few hundred routes.

### BIRD BGP Configuration on the VPS
Upon installation, BIRD created a template configuration file in `/etc/bird/bird.conf`.
I just had to add some parameters regarding my own BGP peering and WireGuard tunnel.

```
root@AWS-VPS:/etc/bird# vim bird.conf

router id 172.22.132.169;

protocol kernel {
  scan time 20;
  import none;
  export filter {
    if source = RTS_STATIC then reject;
    krt_prefsrc = 172.22.132.169;
    accept;
  };
};


protocol device {
        scan time 60;
}

protocol direct {
        interface "*";
}

protocol bgp {
  description "BIRD BGP CONFIG";
  local as 4242421869;
  neighbor 10.132.0.2 as 4242421869;
  graceful restart;
  direct;
  import all;
  export all;
}
```

What this does:
* Sets the router ID `to 172.22.132.169` - an IP address in my designated DN42 range (claimed by my ASN).
* Creates a kernel table. The Kernel protocol is not a real routing protocol. Instead of communicating with other routers in the network, it performs synchronization of BIRD's routing tables with the OS kernel.
* Creates `protocol direct` and `protocol device`, which are BIRD specific settings.
* Initialises the BGP protocol on BIRD, with the local ASN of `4242421869`.
* Sets the neighbour remote ASN to `4242421869` via `10.132.0.2` (the WireGuard IP address for the EdgeRouter).
* The use of the same local and remote ASN triggers internal BGP, as opposed to external BGP. 

### BGP Configuration for the EdgeRouter

BGP on the EdgeRouter similar to BIRD. 

```
set protocols bgp 4242421869 parameters router-id 172.22.132.161
set protocols bgp 4242421869 neighbor 10.132.0.1 description iBGP
set protocols bgp 4242421869 neighbor 10.132.0.1 nexthop-self
set protocols bgp 4242421869 neighbor 10.132.0.1 remote-as 4242421869
set protocols bgp 4242421869 neighbor 10.132.0.1 soft-reconfiguration inbound
```
What this does:
* Creates an instance of BGP, with my local ASN (4242421869) and router id 172.22.132.161.
* Sets the neighbour remote ASN to `4242421869` via `10.132.0.1` (the WireGuard IP address for the AWS VPS).
* Enables `next-hop self`, which forces the router to do a recursive lookup in order to determine which egress interface should be used to send the packets out.
* Enables `soft-reconfiguration inbound`.

---

## Verifying the iBGP Peering

On the EdgeRouter, the BGP state had changed to "Established", its announcing 463 routes to the VPS, at IP `10.132.0.1` (the WireGuard peer IP).

```
usman@ER-X:~$ show ip bgp neighbors 10.132.0.1
BGP neighbor is 10.132.0.1, remote AS 4242421869, local AS 4242421869, internal link
  BGP version 4, remote router ID 172.22.132.169
  BGP state = Established, up for 17:26:23
  Last read 17:26:23, hold time is 180, keepalive interval is 60 seconds
  Neighbor capabilities:
    Route refresh: advertised and received (new)
    4-Octet ASN Capability: advertised and received
    Address family IPv4 Unicast: advertised and received
  Received 5794 messages, 21 notifications, 0 in queue
  Sent 20084 messages, 12 notifications, 0 in queue
  Route refresh request: received 0, sent 0
  Minimum time between advertisement runs is 5 seconds
 For address family: IPv4 Unicast
  BGP table version 43303, neighbor version 43303
  Index 0, Offset 0, Mask 0x1
    Graceful restart: received
  Inbound soft reconfiguration allowed
  NEXT_HOP is always this router
  Community attribute sent to this neighbor (both)
  0 accepted prefixes
  463 announced prefixes

 Connections established 23; dropped 22
Local host: 10.132.0.2, Local port: 179
Foreign host: 10.132.0.1, Foreign port: 54586
Nexthop: 10.132.0.2
Nexthop global: ::
Nexthop local: ::
BGP connection: non shared network
Last Reset: 17:26:23, due to BGP Notification sent
Notification Error Message: (Cease/Administratively Reset.)
```

  
On the AWS VPS:

After entering the `birdc` shell on the VPS, I could also see that the BGP state had been `established`. 

Its accepting those 463 routes that the EdgeRouter advertised, over the WireGuard tunnel (IP address `10.132.0.2`).

```
root@AWS-VPS:~# birdc 
bird> show protocols bgp1
bgp1     BGP      master   up     16:37:06    Established
  Description:    BIRD BGP CONFIG
  Preference:     100
  Input filter:   ACCEPT
  Output filter:  REJECT
  Routes:         463 imported, 0 exported, 463 preferred
  Route change stats:     received   rejected   filtered    ignored   accepted
    Import updates:            466          0          0          0        466
    Import withdraws:            0          0        ---          0          0
    Export updates:            469        466          3        ---          0
    Export withdraws:            0        ---        ---        ---          0
  BGP state:          Established
    Neighbor address: 10.132.0.2
    Neighbor AS:      4242421869
    Neighbor ID:      172.22.132.161
    Neighbor caps:    refresh AS4
    Session:          internal multihop AS4
    Source address:   10.132.0.1
    Hold timer:       147/180
    Keepalive timer:  40/60
bird>
```

---
### Inspecting VPS's Route Table 

The AWS VPS now has DN42's prefixes in its kernel route table.

DN42 prefixes (172.20.0.0/14) have a gateway of the EdgeRouter, (over the WireGuard VPN tunnel).

`route -n` showing a subset of the VPS's route table:
```
root@AWS-VPS:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.31.0.1      0.0.0.0         UG    0      0        0 eth0
172.20.0.53     10.132.0.2      255.255.255.255 UGH   0      0        0 wg0
172.20.1.0      10.132.0.2      255.255.255.0   UG    0      0        0 wg0
172.20.4.32     10.132.0.2      255.255.255.240 UG    0      0        0 wg0
172.20.4.96     10.132.0.2      255.255.255.248 UG    0      0        0 wg0
172.20.4.104    10.132.0.2      255.255.255.248 UG    0      0        0 wg0
172.20.6.224    10.132.0.2      255.255.255.248 UG    0      0        0 wg0
172.20.6.248    10.132.0.2      255.255.255.248 UG    0      0        0 wg0
172.20.8.0      10.132.0.2      255.255.254.0   UG    0      0        0 wg0
172.20.12.192   10.132.0.2      255.255.255.224 UG    0      0        0 wg0
172.20.14.32    10.132.0.2      255.255.255.224 UG    0      0        0 wg0
172.20.14.64    10.132.0.2      255.255.255.224 UG    0      0        0 wg0
172.20.14.192   10.132.0.2      255.255.255.192 UG    0      0        0 wg0
172.20.15.96    10.132.0.2      255.255.255.224 UG    0      0        0 wg0
172.20.16.0     10.132.0.2      255.255.255.128 UG    0      0        0 wg0
172.20.16.128   10.132.0.2      255.255.255.128 UG    0      0        0 wg0
172.20.18.64    10.132.0.2      255.255.255.224 UG    0      0        0 wg0
172.20.18.224   10.132.0.2      255.255.255.240 UG    0      0        0 wg0
172.20.19.64    10.132.0.2      255.255.255.224 UG    0      0        0 wg0
172.20.20.96    10.132.0.2      255.255.255.224 UG    0      0        0 wg0
172.20.28.224   10.132.0.2      255.255.255.224 UG    0      0        0 wg0
```
### Ping Testing
Success! Pinging a device inside the DN42 range `172.23.0.80` confirms that I have access to the DN42 network.
```
root@ip-172-31-0-189:~# ping 172.23.0.80
PING 172.23.0.80 (172.23.0.80) 56(84) bytes of data.
64 bytes from 172.23.0.80: icmp_seq=1 ttl=60 time=69.3 ms
64 bytes from 172.23.0.80: icmp_seq=2 ttl=60 time=72.7 ms
64 bytes from 172.23.0.80: icmp_seq=3 ttl=60 time=71.7 ms
64 bytes from 172.23.0.80: icmp_seq=4 ttl=60 time=83.8 ms
64 bytes from 172.23.0.80: icmp_seq=5 ttl=60 time=74.5 ms
```
### Traceroute Testing
A traceroute to `172.23.0.80` confirms that the packets are flowing:
* First through the Ubiquiti EdgeRouter `10.132.0.2` over the WG VPN tunnel
* Next through the EdgeRouter's eBGP peering with 'highdef' `172.20.229.116`
* Through the DN42 network
* And finally reaches its destination in 5 total hops.
```
root@AWS-VPS:~# traceroute 172.23.0.80
traceroute to 172.23.0.80 (172.23.0.80), 30 hops max, 60 byte packets
 1  10.132.0.2  33.135 ms  33.105 ms  33.088 ms
 2  172.20.229.116  52.967 ms  52.948 ms  52.932 ms
 3  172.20.129.187  60.474 ms  60.459 ms  60.438 ms
 4  172.20.129.169  73.126 ms  73.102 ms  78.113 ms
 5  172.23.0.80  78.097 ms  78.078 ms  78.061 ms
```

## What Next?
In the future, I'll probably launch another VPS, create more BGP peering, but install everything automatically with Ansible. 