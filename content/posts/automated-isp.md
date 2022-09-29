---
title: "Automating the Deployment of a Small ISP using Python"
date: 2022-03-08T12:55:15Z
draft: true
cover:
    image: "/images/pythonISP/diagram.PNG"
    caption: "Small ISP Topology in EVE-NG"
    alt: "alt"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
ShowToc: true
TocOpen: false

---
## This script *should* automatically:
- Assign IPs to the routers' interfaces (loopback and ethernet).
- Create 2 OSPF domains, for both ISPs.
- Established eBGP peerings between the two ISP's edge routers.
- Advertise the respective prefixes, on all edge routers.
- Redistribute learned BGP routes into the OSPF domains, for both ISPs.

## Usage
- The script uses the Netmiko library and my methods from own object oriented framework in `Netmiko_OOP.py`, (from [my network-automation repo](https://github.com/usman-umer/network-automation)).
- Install the python requirements using `pip3 install -r requirements`.
- Replace the router configs (in `isp-lab.py`) with your own credentials and IPs.
- Run script `python3 isp-lab.py`.

## Assumptions:
- You have access to an EVE-NG VM - which runs VyOS routers, connected to each other, as shown above.
- Each VyOS instance is connected to the bridged MGMT cloud, has a DHCP IP, and is accessible (via SSH) from your PC (or from where you're running the python script). 
- *The MGMT connection may require the EVE-NG hypervisor to have a bridged network adapter.*
