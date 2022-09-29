---
title: "DN42 Part 1: Connecting to the DN42 BGP Mesh"
date: 2021-11-22T15:45:37+01:00
draft: false
searchHidden: false
cover:
    image: "/images/DN42/dn42logo.JPEG"
    alt: "alt"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
ShowToc: true
TocOpen: true
tags: ["dn42", "bgp"]
---

---

## What is DN42?

DN42 is a big network, which employs WAN technologies (BGP, whois database, DNS, etc) to create an internet like mesh.

Members connect to each other using VPN tunnels (GRE, OpenVPN, WireGuard, IPsec) and exchange routes via BGP.


![DN42 Mesh](/images/DN42/map.JPG "DN42 Mesh")

*DN42 currently has 410 nodes/users, advertising ~600 prefixes. [realtime map](https://map42.0x7f.cc/)*

---

## Why DN42?

DN42 allows you to experiment with mentioned internet technologies, without the logistical difficulties and high expenses of registering with real AS registries, on the live internet.


More importantly, if you misconfigure something, you don't have to worry about causing global outages - [like a Facebook Engineer did.](https://blog.cloudflare.com/october-2021-facebook-outage/)

![Facebook BGPMeme](/images/DN42/bgpmeme.png "FB BGP Meme")


---

## How I connected to the mesh.

### The Registry
![DN42 Git Registry](/images/DN42/gitrepo.JPG "DN42 Git Registry")
The registry is based on a [Git repository](https://git.dn42.dev/dn42/registry/). This allows users to create branches & pull requests, to register, create objects and make changes to their details.

### Claiming an IP Range and AS Number.

DN42 uses the `172.20.0.0/14` range for IPv4 addresses. 

Available IP ranges and ASNs are can be shown in the [DN42 Free Explorer.](https://explorer.burble.com/free#/) It randomly shows subnets, that are free and useable, from the range.

![DN42 Free Explorer](/images/DN42/free-explorer.JPG "DN42 Free Explorer")

DN42's recommendation is to go with a `/27` subnet, which has 30 usable IP addresses, and should be sufficient for most home users.

Following the same process for an ASN and IPv6, I decided to go with:
* An v4 subnet of `172.22.132.160/27`
* A v6 range of `fd5f:2bd0:1feb::/48`
* An Autonomous System Number of `AS4242421869`

### Making changes to the git repo.


After cloning git the repo to my local machine, I created the following files in the `data/` directory:

`data/aut-num/AS4242421869` - the AS Number.
```
aut-num:            AS4242421869
as-name:            AS-USMAN-DN42
admin-c:            USMAN-DN42
tech-c:             USMAN-DN42
mnt-by:             USMAN-MNT
source:             DN42
```

`data/inet6num/fd5f:2bd0:1feb::_48` - IPv6 Range.
```
inet6num:           fd5f:2bd0:1feb:0000:0000:0000:0000:0000 - fd5f:2bd0:1feb:ffff:ffff:ffff:ffff:ffff
cidr:               fd5f:2bd0:1feb::/48
netname:            USMAN-NETWORK
descr:              Network of Usman
admin-c:            USMAN-DN42
tech-c:             USMAN-DN42
mnt-by:             USMAN-MNT
status:             ASSIGNED
source:             DN42
```

`data/inetnum/172.22.132.160_27` - IPv4 Range.
```
inetnum:            172.22.132.160 - 172.22.132.191
cidr:               172.22.132.160/27
netname:            USMAN-NETWORK
admin-c:            USMAN-DN42
tech-c:             USMAN-DN42
mnt-by:             USMAN-MNT
status:             ASSIGNED
source:             DN42
```
`data/mntner/USMAN-MNT` - data about the maintainer, including my SSH public key (for verification purposes).
```
mntner:             USMAN-MNT
admin-c:            USMAN-DN42
tech-c:             USMAN-DN42
mnt-by:             USMAN-MNT
auth:               ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDOVIdUelAhke7DImFiLdLmK5fIqDrj+UC0fMNwaWzzE/Do9s+8OrlLbyA7UNbnLpXEh6xCaE6ckL1j3Nywv8pSgwgZpnZY1smgCDvMqqRpsfqVTYzCy/6f9tzL80km53JLDKeaJSSikzPhcADpp7eCy6QlWPu+cV3om5cDIEuUvpthGfgFNlrtN88o7ryyInnxaHcr+NYWMcyoocVVieCKKbuBhOe+sCj5IFHuHq5IDX7wNbKRauHE47arhrT6aKWBHFkzMNO/XFo5/yhzeC1gzu9J+efmjf6FrjlP5Mvi6PT8tbAmIwpe6y4NZUXfC4cp3bF7L08Av7E8Lb56fBVdc9aWrGyub2Vc8WRGk1UcZoLKFzGO6DAzH2XMqUe7ZY5qHjqbojGgQXNdwCmUIVhAVjs/XCdd2J48/6PdBf2v0zwqf5x6sSwFgntL7a7+WrmIjXDRQx3aIU79djjL6DvFT1kmB6WN9T3zZAqJ5AXq76taRyPcmNmm3dKThCt34Ns=
source:             DN42
```
`data/person/USMAN-DN42` - contact info.
```
person:             Usman
e-mail:             usman@email.com
nic-hdl:            USMAN-DN42
mnt-by:             USMAN-MNT
source:             DN42
```

`data/route/172.22.132.160_27` - route for v4.
```
route:              172.22.132.160/27
origin:             AS4242421869
mnt-by:             USMAN-MNT
source:             DN42
```

`data/route6/fd5f:2bd0:1feb::_48` - route for v6.
```
route6:             fd5f:2bd0:1feb::/48
origin:             AS4242421869
max-length:         48
mnt-by:             USMAN-MNT
source:             DN42
```

I then ran the provided scripts to squash, verify and sign (with my public SSH key) my commits.

![DN42 Git Registry](/images/DN42/gitrepowssh.JPG "DN42 Git Registry")
Once I pushed my commit, I created a pull request and waited for it to be approved.
![DN42 Git Registry](/images/DN42/register-usman.JPG "DN42 Git Registry")

---

## My Routing Daemon.
*Onto the actual BGP stuff...*
### The Router.
While most DN42 users opt for a Bird/Quagga routing daemon on a VPS, I chose to use my existing Ubiquiti EdgeRouter-X. It's a great router, with a large range of features. [*See my other post for more info.*](https://blog.usman.network/posts/wireguard-vpn-on-a-ubiquiti-edgerouter/)

### Peering with Kioubit.
I peered with the [Kioubit Network](https://dn42.g-load.eu/), as they have automatic peering and a high centrality in the mesh (they have the most neighbours).

I used Kioubit's automatic peering wizard to register my ASN, WireGuard public key and tunnel IP. 
![Registering Kioubit](/images/DN42/kioubit-reg.JPG "Kioubit Registration")
This essentially configures their UK BGP router to listen for a WireGuard peer, which has the correct key. 
 
WireGuard only requires one peer to have a port open, therefore I didn't need to publicly open a port on my router.

Their side has also been configured to allow BGP advertisements from my ASN  and advertise DN42's prefixes, over the WireGuard tunnel.

---

### My Side.
#### WireGuard Tunnel.
```
set interfaces wireguard wg4 address 192.168.219.127/32
set interfaces wireguard wg4 firewall in name DN42_MAIN_IN
set interfaces wireguard wg4 firewall local name DN42_MAIN_LOCAL
set interfaces wireguard wg4 mtu 1420
set interfaces wireguard wg4 peer <peer pubkey> allowed-ips 0.0.0.0/0
set interfaces wireguard wg4 peer <peer pubkey> endpoint 'uk1.g-load.eu:21869'
set interfaces wireguard wg4 private-key <my private key>
set interfaces wireguard wg4 route-allowed-ips false
```
*What this does:*
* Creates a virtual interface called `wg4`.
* Assigns it the IP address `192.168.219.127/37` (configured in the Kioubit's automated peering).
* Sets the allowed-ips value to `0.0.0.0/0` - this requests to/from any IP address to traverse the tunnel.
* Defines the Kioubit's Uk Node's endpoint and port.
* Sets the `route-allowed-ips` value to `false`, so a default route doesn't get added to the route table.
* Adds the interfaces to the firewall group `DN42_IN` & `DN42_LOCAL` *more on this later*
---
#### BGP Peering.
```
set protocols bgp 4242421869 neighbor 172.20.53.104 description kioubit
set protocols bgp 4242421869 neighbor 172.20.53.104 ebgp-multihop 255
set protocols bgp 4242421869 neighbor 172.20.53.104 remote-as 4242423914
set protocols bgp 4242421869 neighbor 172.20.53.104 soft-reconfiguration inbound
set protocols bgp 4242421869 neighbor 172.20.53.104 update-source 172.22.132.161
```
*What this does:*
* Enables the BGP routing daemon with my ASN 4242421869.
* Sets the remote remote AS, with a neighbour IP as `172.20.43.104`(kioubit's IP address in WG tunnel).
* Sets eBGP Multihop to 255 hops - this is required as the peering is established over a VPN tunnel, not a direct connection.

--- 
#### Other BGP and Routing settings.
```
set protocols bgp 4242421869 network 172.22.132.160/27
set protocols bgp 4242421869 parameters router-id 172.22.132.161
set protocols static interface-route 172.20.53.104/32 next-hop-interface wg4
```
*What this does:*
* Advertises the `172.22.132.160/27` prefix (my DN42 subnet).
* Sets the router ID (which is advertised to peers) to `172.22.132.161`.
* Created a static interface route for Kioubit's address `172.20.53.104`, via wg4. 
---
### Firewall Rules and security.

I used firewall rules to limit the traffic between my network and DN42.

Ubiquiti's firewalling works by having different 'directions'. `IN` and `LOCAL`.

* `IN` Matches on established/related and invalid traffic that is passed through the router (WAN to LAN).

* `LOCAL` Matches on established/related and invalid traffic that is destined for the router itself (WAN to LOCAL).

#### DN42_MAIN_IN.
```
set firewall name DN42_MAIN_IN default-action drop
set firewall name DN42_MAIN_IN rule 1 action accept
set firewall name DN42_MAIN_IN rule 1 description 'Allow established/related'
set firewall name DN42_MAIN_IN rule 1 log disable
set firewall name DN42_MAIN_IN rule 1 protocol all
set firewall name DN42_MAIN_IN rule 1 state established enable
set firewall name DN42_MAIN_IN rule 1 state invalid disable
set firewall name DN42_MAIN_IN rule 1 state new disable
set firewall name DN42_MAIN_IN rule 1 state related enable
```
The `IN` ruleset has a default action to drop any packets, and only allow any traffic that is established/related. Meaning only any traffic that is initiated from a device in my LAN (not the router) will be allowed. 

If I wanted to host any services for any other DN42 members, this is where I would add another rule to allow traffic to a specific host & port. Along with a port forward to bypass NAT.

#### DN42_MAIN_LOCAL.
```
set firewall name DN42_MAIN_LOCAL default-action drop
set firewall name DN42_MAIN_LOCAL rule 1 action accept
set firewall name DN42_MAIN_LOCAL rule 1 description 'Allow established/related'
set firewall name DN42_MAIN_LOCAL rule 1 log disable
set firewall name DN42_MAIN_LOCAL rule 1 protocol all
set firewall name DN42_MAIN_LOCAL rule 1 state established enable
set firewall name DN42_MAIN_LOCAL rule 1 state invalid disable
set firewall name DN42_MAIN_LOCAL rule 1 state new disable
set firewall name DN42_MAIN_LOCAL rule 1 state related enable
set firewall name DN42_MAIN_LOCAL rule 2 action accept
set firewall name DN42_MAIN_LOCAL rule 2 description 'Allow BGP'
set firewall name DN42_MAIN_LOCAL rule 2 destination port 179
set firewall name DN42_MAIN_LOCAL rule 2 log disable
set firewall name DN42_MAIN_LOCAL rule 2 protocol tcp
```

The `local` ruleset also has the same default action and established/related rule. I also have a rule to allow incoming TCP connections on port 179, for BGP.

This is where I would create another rule if I wanted to allow DN42 members to access something directly on the router. For example, SSH for administrative purposes.

---

### Source NAT.

I also implemented NAT so that I can access DN42 from any device in my LAN. 

```
set service nat rule 5050 description 'source NAT for DN42 Peer Kioubit'
set service nat rule 5050 log disable
set service nat rule 5050 outbound-interface wg4
set service nat rule 5050 outside-address address 172.22.132.162
set service nat rule 5050 protocol all
set service nat rule 5050 source
set service nat rule 5050 type source
```

Connections destined for the DN42 prefix, will be NAT'ed through the address `172.22.132.162` (one of the IPs in my claimed range).

For example, my Raspberry Pi (which is in a local VLAN) can access the DN42 network and run a traceroute to a DN42 IP.

![NAT Trace](/images/DN42/nat_trace.JPG "NAT Traceroute")

---

### Maps.
At the moment, I have connections with 5 peers. This improves redundancy and provides better access to all areas of the mesh. 
![Current DN42 Map](/images/DN42/map2.JPG "Current DN42 Map")