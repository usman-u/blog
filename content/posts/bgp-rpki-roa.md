---
title: "DN42 Part 3: BGP ROA/RPKI Filtering using Docker"
date: 2022-07-19T21:37:00+01:00
draft: false
showtoc: true
cover:
    image: "/images/rpki-roa/roa-rpki-cisco.PNG"
    alt: "alt"
    caption: ""
    relative: false 
---

## What is ROA/RPKI?

Route Origin Authentication (ROA) - is a way to verify whether an IP prefix advertised, is actually owned by the Autonomous System (AS) that advertised it.

Resource Public Key Infrastructure (RPKI) - is a protocol that facilitates the exchange of ROA and other related information between ASes.

---

## How  does ROA/RPKI work?


* Essentially, there are central databases which contain a list of all ASes and the IP prefixes that they are allowed to advertise. On the public internet, these databases are usually hosted by Regional Internet Regestries, such as ARIN, APNIC, and RIPE.

* Validators receive cryprograhically-signed ROAs from the Regional Internet Registries, which contains a list of all ASes and the IP prefixes that they are allows to advertise.

* Routers query the validators with recieved BGP advertisements, to ensure that prefixe advertisements are only accepted from ASes that own them.

---

## Why ROA/RPKI?

ROA/RPKI can be used to prevent:
* **BGP hijacking** - where an actor is able to advertise a prefix that they don't own, with malicious intent. For example, the 2018 BGP hijack of Amazon DNS to steal crypto currency. [^1]

* **Route Leaks** - where an AS inadvertently advertises a prefix that they don't own. For example, in 2008, where an ISP in Pakistan accidentally announced IP routes for YouTube by blackholing the video service internally to their network. [^2]


---

## ROA/RPKI for DN42

Similar to the public internet, ROA/RPKI has the same components on DN42:
1. A central database of ASes and the IP prefixes they are allowed to advertise
2. A validator which recieves cryptographically-signed ROAs from the central database.
3. A BGP router that queries the validator.

---

### 1. A central database of ROAs
In DN42, the central authority that maintains the ROA/RPKI database is ["Burble"](https://burble.dn42/). Shown below, there is just .json file that contains all the data. This is what the validator will download.
![DB](/images/rpki-roa/list.PNG)

---

### 2. The Validator - cloudflare/goRTR Docker Container

* The validator is a [cloudflare/goRTR](https://github.com/cloudflare/gortr) docker container that runs on an Ununtu Linux Server, within my network. 

* It is responsible for downloading the ROA/RPKI database from Burble, and then responding to queries from my router.

#### Starting up the Docker Container

The docker run command that I used to start the goRTR container:
```
docker run -ti -p 8082:8082 cloudflare/gortr -cache https://dn42.burble.com/roa/dn42_roa_46.json -verify=false -checktime=false -bind :8082
```
* Roughly translated, this means: create a continer using the cloudflare/gortr image, using burble's .json file, and bind it to port 8082.

---

#### Verifying the ROA/RPKI database connectivity

![Logs](/images/rpki-roa/logs.PNG) 

Checking the container's logs, we can see that the contianer periodically downloads the ROA/RPKI database from Burble and checks to see if there are any changes.

---

### 3. RPKI on VyOS

Shown below, the router that I installed RPKI on is my VyOS cloud router `edge.dn42.lan`, as it is the only node that BGP peerings with other DN42 members.

![VyOS](/images/rpki-roa/routers_only.png)

#### RPKI Configuration
Initially, I configured the connection to the validator container. This creates a cache of the RPKI DB from `10.0.10.13:8082` (the docker container endpoint).
```
 protocols {
     rpki {
         cache GoRTR {
             address 10.0.10.13
             port 8082
         }
     }
```

---

I also created a route map that denies prefixes, that match any RPKI "invalid" prefixes. This means that any prefixes that are not in the RPKI database, or are invalid will be filtered out of the BGP route table.

```
 policy {
     route-map DN42-ROA {
         description DN42RouteMap

         rule 50 {
             action deny
             match {
                 rpki invalid
             }
         }
     }
 }
```

---

Finally, assigning the route map to BGP peers. Below is an example of a BGP peering, on the `edge.dn42.lan` router, with the route map applied.

```
 protocols {
     bgp 4242421869 {
         neighbor 172.20.16.141 {
             address-family {
                 ipv4-unicast {
                     route-map {
                         export DN42-ROA
                         import DN42-ROA
                     }
                 }
             }
             description chrismoos
             ebgp-multihop 250
             remote-as 4242421588
         }
```
---

All Done!

The Vyos router is configured to use ROA/RPKI with the route map, so invalid prefixes are filtered out of the routing table.


[^1]: https://www.thousandeyes.com/blog/amazon-route-53-dns-and-bgp-hijack

[^2]: https://www.wired.com/2008/02/pakistans-accid/