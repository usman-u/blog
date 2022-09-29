---
title: "Filtering BGP routes on a Ubiquiti EdgeRouter using Route Maps and Prefix Lists."
date: 2022-01-08T15:45:37+01:00
draft: true
searchHidden: false
cover:
    image: ""
    alt: "alt"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
ShowToc: true
TocOpen: true
---

I recently connected to the DN42 BGP mesh, a big dynamic VPN which employs WAN technologies to create an internet like mesh. 
*Read my other most more info.*



### The Problem
One problem I encountered was that some prefixes (outside of the designated DN42 `172.20.0.0/14` range) were being advertised. Specifically in the `10.0.0.0/8` range.

This was quite inconvinient as it created conflicts between my subnets (i.e. VLANs & `/30` site to site links).

This is a shortened version of my route table, currently containing the unwanted prefixes.
```
usman@Ubiquiti-EdgeRouter:~$ show ip route bgp
IP Route Table for VRF "default"
B    *> 10.99.16.0/22 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.99.36.0/22 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.100.0.0/14 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.100.41.0/24 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.100.232.0/21 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.100.232.0/24 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.100.234.0/23 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.101.66.0/28 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 10.101.66.16/28 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:53
B    *> 172.20.0.53/32 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:53
B    *> 172.20.1.0/24 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.4.32/28 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.4.96/29 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.4.104/29 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.6.224/29 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.15.96/27 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.16.0/25 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.16.128/25 [20/0] via 172.20.16.141 (recursive is directly connected, wg5) ), 00:00:07
B    *> 172.20.18.64/27 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:54
B    *> 172.20.18.224/28 [20/0] via 172.20.53.104 (recursive is directly connected, wg4) ), 00:00:25
---MORE---
```

## The Solution
### Prefix Lists & Route Maps

```
set policy prefix-list DN42 description 'allow only in prefix 172.20.0.0/14'
set policy prefix-list DN42 rule 1 action permit
set policy prefix-list DN42 rule 1 le 32
set policy prefix-list DN42 rule 1 prefix 172.20.0.0/14

set policy route-map DN42-INGRESS rule 1 match ip address prefix-list DN42

set protocols bgp 4242421869 neighbor 172.20.229.116 prefix-list
set protocols bgp 4242421869 peer-group DN42 prefix-list import DN42
```

---

```
usman@ER-X# show policy prefix-list
 prefix-list DN42 {
     description "allow only in prefix 172.20.0.0/14"
     rule 1 {
         action permit
         le 32
         prefix 172.20.0.0/14
     }
 }
usman@ER-X# show policy route-map
 route-map DN42-INGRESS {
     rule 1 {
         action permit
         match {
             ip {
                 address {
                     prefix-list DN42
                 }
             }
         }
     }
 }
usman@ER-X#
```