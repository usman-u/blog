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

### LibreNMS Weathermap
![ Diagram](/images/hn/weathermap.png)

[Click here](https://libre.usman.network/plugins/Weathermap/test.html) to see a live version of the weathermap.

---

## Network Infrastructure Breakdown:

* My network mainly consists of two Ubiquiti EdgeRouter Xs, two Cisco Catalyst switches, and a VyOS cloud VPS.

* WireGuard is used as the VPN tunneling protocol, to connect sites and VPS Instances.
* DN42 BGP:
    * I participate in the DN42 BGP project.
    * `fr-lil1.dn42` and `uk-lon1.dn42` act as the BGP edge routers, peering with providers and allowing access the to the DN42 private network.
    * eBGP peering with various upstream DN42 providers, over point to point WireGuard VPNs.
    * eBGP peerings are filtered using RPKI-ROA, such that invalid prefixed are removed. See my [other post](./bgp-rpki-roa) for more info.
    * iBGP between `uk-lon1.dn42` and `fr-lil1.dn42`, to exchange external DN42 routes.

* OSPF:
    * OSPF area 0 between `uk-lon1.dn42` and `fr-lil1.dn42`, to exchange internal routes (DN42 subnet `172.22.132.160/27`).
    * OSPF area 0 summarisation (range) of `172.20.0.0/14`, so that a full DN42 BGP table isn't redistributed into area 1.
    * OSPF area 1 with `erx.usman.lan` and `erx.zahid.lan`.
    * OSPF area 1 routers only receive the sumamrised route, and internal routes (i.e. `172.22.132.160/27` and `10.0.0.0/16`).
    * `fr-lil1.dn42` is the Area Border Router (ABR) between OSPF area 0 and 1.
* DNS A and PTR records, for the `lan` domain are hosted on an unbound docker container, on `pi.usman.lan`.
* NetDevOps & Automation
    * Partly deployed using Drone CI/CD and my own Python Configuration Management Framework. See [github.com/usman-u/network-automation](https://github.com/usman-u/network-automation) and [github.com/usman-u/usmannet](github.com/usman-u/usmannet) for more info.

---

## More Info:

---

### My Home LAN

* Ubiquiti Edgerouter X - `erx.usman.lan`
    * WireGuard VPNs to `erx.zahid.lan` and `dn42-vps.lan`.
    * OSPF area 1 neighbors with`erx.zahid.lan` and `dn42-vps.lan`.
    * Source NAT on VPN interfaces, such that any outbound traffic destined for DN42, is NATed to a DN42 IP. 
    * *Router-On-A-Stick* VLANs - with a VLAN trunk down to the Cisco 2960G.

* Cisco 2960G - `usman-cisco.lan`
    * Basic Layer 2 switching to normal end user hosts.

* Raspberry Pi 4 8GB - `pi.usman.lan`

    * Some of the Docker Applications I host:
        * [Unbound DNS Server](https://github.com/MatthewVance/unbound-docker-rpi) DNS A and PTR records, for the `.lan` domain.
        * [Plex Media Server](https://docs.linuxserver.io/images/docker-plex)
        * [Radarr](https://radarr.video/), [Sonarr](https://sonarr.tv/), [Jackett](https://github.com/Jackett/Jackett), [qBittorrent](https://hub.docker.com/r/linuxserver/qbittorrent).
        * [Syncthing](https://syncthing.net/).
        * [NGINX Proxy Manager](https://nginxproxymanager.com/) - routes a few of the mentioned services through [Cloudfare's WAF](https://www.cloudflare.com/en-gb/waf/), for DDOS mitigation and public IP obfuscation.

* WD 8TB NAS - `nas.usman.lan`
    * SMB Shares.
    * Plex Media Server

---


### My DN42 Nodes

* My DN42 nodes are hosted on cloud infrastructure.
* See [my DN42 peering page](../../dn42) for more info.

---

### Offsite Network - Owned by Zahid (see [f2ncy.github.io](https://f2ncy.github.io))
* Ubiquiti Edgerouter X - `erx.zahid.lan`
    * WireGuard VPNs and OSPF area 1 to `erx.usman.lan` and `fr-lil1.dn42`
    * Source NAT on VPN interfaces, such that any outbound traffic destined for DN42, is NATed to a DN42 IP. 

* Cisco 3560G - `core.zahid.lan`
    * SVIs for VLANs.
    * OSPF area 1 with `erx.zahid.lan`

* Raspberry Pi 4 8GB - `pi.zahid.lan`
    * Running as a host for Docker apps.
    * 6TB HDD.

---