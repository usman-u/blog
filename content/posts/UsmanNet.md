---
title: "UsmanNet 2022 - An Overview of my Network"
date: 2022-07-28T20:29:16+01:00
draft: false
showtoc: false
cover:
    image: "/images/hn/cables.PNG"
    alt: "alt"
    caption: ""
    relative: false 
---
![ Diagram](/images/hn/net_map.png)

---

## Network Infrastructure Breakdown:

* My network mainly consists of two Ubiquiti EdgeRouter Xs, two Cisco Catalyst switches, and a VyOS cloud VPS.
* WireGuard is used as the VPN tunneling protocol, to connect the Edgerouters and the Vyos VPS.
* I participate in the DN42 BGP project. BGP is used on `edge.dn42.lan`, to connect to the BGP upstream & downstream peers, providing access to the DN42 private network.
* OSPF is used between `erx.usman.lan`, `erx.zahid.lan` and `edge.dn42.lan`, to exchange internal routes.
* BGP routes from DN42 are also redistributed into OSPF, at `edge.dn42.lan`, so that the DN42 network is accessible from the other routers.

---

## More Info:

---

### My Home LAN

* Ubiquiti Edgerouter X - `erx.usman.lan`
    * WireGuard VPNs to `erx.zahid.lan` and `dn42-vps.lan`.
    * OSPF neighbors with`erx.zahid.lan` and `dn42-vps.lan`.
    * Source NAT on VPN interfaces, such that any outbound traffic destined for DN42, is NATed to a DN42 IP. 
    * *Router-On-A-Stick* VLANs - with a VLAN trunk down to the Cisco 2960G.

* Cisco 2960G - `usman-cisco.lan`
    * Basic Layer 2 switching to normal end user hosts.

* Raspberry Pi 4 8GB - `pi.usman.lan`
    * Running as a host for Docker apps.

* WD 8TB NAS - `nas.usman.lan`
    * SMB Shares.
    * Plex Media Server

---

### Offsite Network
* Ubiquiti Edgerouter X - `erx.zahid.lan`
    * WireGuard VPNs and OSPF area 0 to `erx.usman.lan` and `dn42.erx.lan`
    * Source NAT on VPN interfaces, such that any outbound traffic destined for DN42, is NATed to a DN42 IP. 

* Cisco 3560G - `core.zahid.lan`
    * SVIs for VLANs.
    * iBGP peering with `erx.zahid.lan`

* Raspberry Pi 4 8GB - `pi.zahid.lan`
    * Running as a host for Docker apps.
    * 6TB HDD.

---

### My DN42 Node

* A Cloud VPS running [VyOS](https://vyos.io/) as my DN42 Node. (See my [other posts](https://blog.usman.network/posts/dn42-bgp/) for more info about DN42).
* eBGP peering with three upstream DN42 providers, over point to point WireGuard VPNs.
* OSPF area 0 with `erx.usman.lan` and `erx.zahid.lan`.
* Redistributing BGP into OSPF.
* Filtering BGP advertisements using [GoRTR RPKI ROA](https://dn42.eu/howto/ROA-slash-RPKI).
* Filtering BGP exports using route-maps and prefix lists, such that BGP updates from `AS65535` and `AS65534` aren't advertised to DN42.

---

### Services being hosted:

* On the Raspberry Pi Docker Hosts:
    * [Plex Media Server](https://docs.linuxserver.io/images/docker-plex)
    * [Radarr](https://radarr.video/), [Sonarr](https://sonarr.tv/), [Jackett](https://github.com/Jackett/Jackett), [qBittorrent](https://hub.docker.com/r/linuxserver/qbittorrent).
    * [Nextcloud](https://nextcloud.com/athome/).
    * [Syncthing](https://syncthing.net/).
    * [NGINX Proxy Manager](https://nginxproxymanager.com/) - routes a few of the mentioned services through [Cloudfare's WAF](https://www.cloudflare.com/en-gb/waf/), for DDOS mitigation and public IP obfuscation.

---
