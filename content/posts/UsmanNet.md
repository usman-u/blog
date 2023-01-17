---
title: "UsmanNet 2022 - An Overview of my Network"
date: 2022-07-28T20:29:16+01:00
draft: false
showtoc: false
cover:
    image: "/images/hn/cables.PNG"
    # image: "/images/hn/weathermap.png"
    alt: "alt"
    caption: ""
    relative: false 
---

### Logical Overview
![ Diagram](/images/hn/net_map.png)

---

## Network Infrastructure Breakdown:

* My network mainly consists of two Ubiquiti EdgeRouter Xs, two Cisco Catalyst switches, and cloud VPSs.
* WireGuard is used as the VPN tunneling protocol, to connect sites and VPS Instances.

* DN42 BGP: I'm a member of the [DN42 BGP project](https://dn42.eu).
    * **fr-lil1**, **uk-lon1**, **us-west1** act as the eBGP edge routers, peering peering with various providers, and allowing access the to the DN42 private network.
    * iBGP between **fr-lil1**, **uk-lon1** & **us-west1** (exchanging only external DN42 BGP routes).
    * Prefixes from eBGP peers are filtered using RPKI-ROA (docker), such that invalid prefixed are removed. See my [other post](./bgp-rpki-roa) for more info.

* OSPF:
    * OSPF area 0 between **erx.usman** & **fr-lil1** (advertising only internal routes).
    * **fr-lil1** is the OSPF ABR.
    * OSPF area 1 between **fr-lil1**, **uk-lon1** & **us-west1**.
    * Using MultiArea OSPF, so that a full DN42 BGP route table isn't redistributed into OSPF.
    * **172.20.0.0/14** is summarised into area 0, at the OSPF ABR (fr-lil1).

* Source NAT on **erx.usman** such that traffic destined for DN42 services (**172.20.0.0/14**), is translated into a DN42 IP (from my range).
* Authoritative DNS records (for the **.lan** TLD) are running on **pi.usman.lan** & **plex.usman.lan**, as Unbound Docker Containers.
* Partly deployed using Drone CI/CD and my own Python Configuration Management Framework. 
* See [github.com/usman-u/network-automation](https://github.com/usman-u/network-automation) and [github.com/usman-u/usmannet](github.com/usman-u/usmannet) for more info.

---

### LibreNMS Weathermap
![ Diagram](/images/hn/weathermap.png)

[Click here](https://libre.usman.network/plugins/Weathermap/test.html) to see a live version of the weathermap.

---
## More Info:

---

### My Home LAN

* Ubiquiti Edgerouter X - **erx.usman.lan**
    * WireGuard VPNs to and **dn42-vps.lan**.
    * OSPF area 1 neighbors with and **dn42-vps.lan**.
    * Source NAT on VPN interfaces, such that any outbound traffic destined for DN42, is NATed to a DN42 IP. 
    * *Router-On-A-Stick* VLANs - with a VLAN trunk down to the Cisco 2960G.

* Cisco 2960G - **usman-cisco.lan**
    * Basic Layer 2 switching to normal end user hosts.

* Raspberry Pi 4 8GB - **pi.usman.lan**

    * Some of the Docker Applications I host:
        * [BIND9 DNS Server](https://wiki.debian.org/Bind9) DNS A and PTR records, for the **.usman.network.** domain.
        * [Plex Media Server](https://docs.linuxserver.io/images/docker-plex)
        * [Radarr](https://radarr.video/), [Sonarr](https://sonarr.tv/), [Jackett](https://github.com/Jackett/Jackett), [qBittorrent](https://hub.docker.com/r/linuxserver/qbittorrent).
        * [Syncthing](https://syncthing.net/).
        * [NGINX Proxy Manager](https://nginxproxymanager.com/) - routes a few of the mentioned services through [Cloudfare's WAF](https://www.cloudflare.com/en-gb/waf/), for DDOS mitigation and public IP obfuscation.

* WD 8TB NAS - **nas.usman.lan**
    * SMB Shares.
    * Plex Media Server

---


### My DN42 Nodes

* My DN42 nodes are hosted on cloud infrastructure.
* See [my DN42 peering page](../../dn42) for more info.

---