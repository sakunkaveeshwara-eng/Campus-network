# Martin Luther King University: Secure Two-Campus Network (Cisco Packet Tracer)

A hierarchical, redundant and secured network design for a university with a **main campus (HQ)** and a **branch campus** about 100 miles apart. It was designed and implemented in Cisco Packet Tracer. The two campuses are linked through ISP routers and protected by Cisco ASA firewalls with a site-to-site IPsec VPN. All DHCP, DNS, web, email and FTP services run in a DMZ server farm at the main campus.

> **Status:** completed and tested (DHCP, CAPWAP/WLC, inter-VLAN routing, OSPF, HSRP, IPsec VPN).

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Topology](#topology)
3. [Technologies Implemented](#technologies-implemented)
4. [IP Addressing Plan](#ip-addressing-plan)
5. [VLAN Plan](#vlan-plan)
6. [Device Inventory](#device-inventory)
7. [Key Configuration Notes](#key-configuration-notes)
8. [Verification Commands](#verification-commands)
9. [Troubleshooting Log (Lessons Learned)](#troubleshooting-log-lessons-learned)
10. [Repository Structure](#repository-structure)
11. [How to Open the Project](#how-to-open-the-project)
12. [Author](#author)

---

## Project Overview

The university has four faculties on both campuses: **Health & Science, Business, Engineering/Computing, and Art/Design**. The main campus also hosts the IT department, which manages both sites. About 30,000 users are expected to double by 2025, so scalability was a core design goal.

**Design goals**

- Confidentiality, integrity and availability of data and communication
- Redundancy at every layer (dual core switches, HSRP, dual DHCP servers, EtherChannel)
- Centralised services in a DMZ, reachable securely from the branch
- Separate segments for LAN, WLAN, voice and management
- A secure IPsec tunnel between the campuses

---

## Topology

> Add your Packet Tracer screenshot here, for example `docs/topology.png`.

```
                         [ Google Cloud ]
                                |
                         [ INTERNET router ]
                         /                 \
                  [ HQ-ISP ]            [ BRANCH-ISP ]
                      |                       |
                  [ HQ-FWL ]  <== IPsec ==>  [ BRANCH-FWL ]
                  /    |    \                  /        \
          [ DMZ ]  [HQ-MLSW1]=[HQ-MLSW2]   [B-MLSW1]=[B-MLSW2]
        DHCP-1/2,     |  (EtherChannel)        |  (EtherChannel)
        DNS, WEB,   5 access switches        4 access switches
        EMAIL, FTP  (HS, BUS, ENG, ART, IT)  (HS, BUS, ENG, ART)
```

**Design model:** hierarchical (core / distribution / access) with redundant core multilayer switches at each campus.

---

## Technologies Implemented

| Area | Implementation |
|---|---|
| Design model | Hierarchical with redundancy |
| Switching | Catalyst 3650 multilayer core, Catalyst 2960 access switches |
| VLANs | 10 (Management), 20 (LAN), 50 (WLAN), 60 and 90 (branch LAN/WLAN), 199 (Blackhole) |
| Link aggregation | LACP EtherChannel between the core switches |
| Layer 2 protection | STP PortFast and BPDU Guard on access ports |
| Inter-VLAN routing | On the multilayer core switches (SVIs) |
| First-hop redundancy | HSRP between the core switches |
| Routing | OSPF process 15, area 0 (switches, firewalls, ISP routers) |
| DHCP | Two DHCP servers in the DMZ, reached through `ip helper-address` relay |
| Wireless | Cisco WLC with lightweight APs per department (CAPWAP) |
| Voice | IP phones in each department |
| Security | Cisco ASA 5506-X firewalls with security zones, ACLs and NAT |
| VPN | Site-to-site IPsec between HQ-FWL and BRANCH-FWL |
| Management | SSH-only VTY access restricted by a standard ACL to the Senior Network Security Engineer PC |
| Basic hardening | Hostnames, console and enable passwords, password encryption, banners, `no ip domain-lookup` |

---

## IP Addressing Plan

### Campus networks

| Purpose | HQ (Main) | Branch |
|---|---|---|
| Management (VLAN 10) | 192.168.10.0/24 | - |
| LAN (VLAN 20 / VLAN 60) | 172.16.0.0/16 | 172.17.0.0/16 |
| WLAN (VLAN 50 / VLAN 90) | 10.10.0.0/16 | 10.11.0.0/16 |
| DMZ server farm | 10.20.20.0/27 | - |

### Point-to-point and WAN links

| Link | Subnet |
|---|---|
| HQ-FWL <-> HQ-MLSW1 | 10.20.20.32/30 |
| HQ-FWL <-> HQ-MLSW2 | 10.20.20.36/30 |
| BRANCH-FWL <-> B-MLSW1 | 10.20.20.40/30 |
| BRANCH-FWL <-> B-MLSW2 | 10.20.20.44/30 |
| HQ-FWL <-> HQ-ISP | 105.100.50.0/30 |
| BRANCH-FWL <-> BRANCH-ISP | 205.200.100.0/30 |
| HQ-ISP <-> INTERNET | 20.20.20.0/30 |
| INTERNET <-> BRANCH-ISP | 30.30.30.0/30 |

### Key host addresses

| Device | Address |
|---|---|
| HQ-FWL (OUTSIDE) | 105.100.50.2 |
| HQ-FWL (DMZ) | 10.20.20.1 |
| BRANCH-FWL (OUTSIDE) | 205.200.100.2 |
| DHCP-1 / DHCP-2 | 10.20.20.5 / 10.20.20.6 |
| DNS server | 10.20.20.7 |
| HQ WLC | 10.10.0.15 |

Servers in the DMZ use **static** addresses. All other devices use DHCP.

---

## VLAN Plan

| VLAN | Name | Subnet | Campus |
|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | HQ |
| 20 | LAN | 172.16.0.0/16 | HQ |
| 50 | WLAN | 10.10.0.0/16 | HQ |
| 60 | LAN | 172.17.0.0/16 | Branch |
| 90 | WLAN | 10.11.0.0/16 | Branch |
| 199 | Blackhole | - | Both (unused ports) |

---

## Device Inventory

| Role | Devices |
|---|---|
| Firewalls | HQ-FWL, BRANCH-FWL (ASA 5506-X) |
| Core switches | HQ-MLSW1, HQ-MLSW2, B-MLSW1, B-MLSW2 (3650-24PS) |
| Access switches | One 2960 per faculty on each campus |
| ISP routers | HQ-ISP, BRANCH-ISP, INTERNET (2911) |
| DMZ servers | DHCP-1, DHCP-2, DNS, WEB, EMAIL, FTP |
| Wireless | HQ-WLC (2504) and a LAP in each department |
| Cloud | Google Cloud server and VM |
| Endpoints | PCs, laptops, tablets, smartphones, printers and IP phones per department |

---

## Key Configuration Notes

### DHCP relay

Access VLAN interfaces on the core switches forward client broadcasts to both DHCP servers.

```
interface Vlan60
 ip address 172.17.0.3 255.255.0.0
 ip helper-address 10.20.20.5
 ip helper-address 10.20.20.6
 standby 60 ip 172.17.0.1
```

Each DHCP server holds a pool for every subnet (HQ LAN/WLAN/management and branch LAN/WLAN). The WLAN pools include the WLC address (10.10.0.15) so lightweight APs can discover the controller.

### OSPF

All routers, multilayer switches and firewalls run OSPF process 15 in area 0. Each device must advertise **its own** connected networks. For example, the branch core switches advertise 172.17.0.0/16, 10.11.0.0/16 and their firewall uplink, not the HQ networks.

### NAT

- **HQ-FWL:** dynamic PAT for the management and LAN subnets toward OUTSIDE. The WLAN subnet (10.10.0.0/16) is **not** translated, because the branch APs must reach the WLC at its real address.
- **BRANCH-FWL:** no NAT toward OUTSIDE. Branch DHCP and CAPWAP traffic crosses the ISP path to the DMZ and WLC, and translating it breaks both (see the troubleshooting log).

### Firewall ACLs

The OUTSIDE interfaces permit ICMP, DHCP (UDP 67/68), DNS, web, SMTP, FTP, CAPWAP (UDP 5246/5247, plus 12222/12223 for Cisco AP discovery), IKE (UDP 500) and ESP for the VPN. The ACL name is **case-sensitive**.

### Site-to-site IPsec VPN

IKEv1 with a pre-shared key and an ESP transform set between HQ-FWL (105.100.50.2) and BRANCH-FWL (205.200.100.2). The crypto ACLs are mirror images and cover the branch LAN/WLAN toward the DMZ and HQ WLAN.

### SSH access

VTY lines accept SSH only, and a standard ACL permits only the Senior Network Security Engineer PC.

---

## Verification Commands

| Purpose | Command | Device |
|---|---|---|
| OSPF neighbors | `show ip ospf neighbor` / `show ospf neighbor` | Switches, routers / ASA |
| Routes | `show ip route` / `show route` | Switches, routers / ASA |
| HSRP state | `show standby brief` | Core switches |
| EtherChannel | `show etherchannel summary` | Core switches |
| DHCP relay | `show ip interface vlan <id>` | Core switches |
| NAT policy | `show nat` | ASA |
| ACL hits | `show access-list` | ASA |
| IKE phase 1 | `show crypto isakmp sa` | ASA |
| IPsec traffic | `show crypto ipsec sa` | ASA |
| End-to-end | `ping 10.20.20.5` from a branch core switch | Branch |

Use **Simulation mode** with a DHCP, UDP or ICMP filter to trace where a packet is dropped.

---

## Troubleshooting Log (Lessons Learned)

These are the real problems found while building the project and how each was resolved.

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | Branch PCs got "DHCP request failed" | B-MLSW1's OSPF advertised the HQ networks instead of the branch networks, so the return path to the branch did not exist | Re-created OSPF with the branch networks and uplink subnet |
| 2 | DHCP request reached the server but got no reply | BRANCH-FWL PAT rewrote the relayed DHCP source to 205.200.100.2, which matches no pool | Removed the branch PAT rules |
| 3 | DHCP Offer dropped at BRANCH-FWL | The reply was addressed to the real 172.17.0.x address, so the ASA's **NAT RPF** check dropped it | Same fix: no NAT on that path |
| 4 | Replies vanished at HQ-ISP | Serial link looked up/up but passed no traffic after config restores | Power-cycled the router |
| 5 | Branch APs got an address but never joined the WLC | CAPWAP lines were in `RES-Access`, a different ACL than the applied `RES-ACCESS` (case-sensitive) | Added the lines to the correct ACL |
| 6 | Branch ping to the WLC (10.10.0.15) failed at HQ-FWL | PAT objects covered 10.10.0.0/16, so NAT RPF dropped traffic to the WLC | Removed those PAT objects |
| 7 | Config reverted after a restart | Changes were never saved | `write memory` on ASAs, `copy run start` on routers/switches |

**Takeaways**

- Check the **source IP** of relayed packets at the DHCP server. A translated source means NAT is interfering.
- On an ASA, NAT applies to every interface pair. Traffic between your own campuses should not be translated.
- ASA ACL names are case-sensitive.
- Always save configs and keep a backup of the `.pkt` file.

---

## Repository Structure

```
.
├── README.md
├── MLK-University-Network.pkt      # Packet Tracer project file
├── docs/
│   ├── topology.png                # Network diagram
│   ├── ip-addressing.xlsx          # (optional) addressing table
│   └── report.pdf                  # (optional) project report
└── configs/                        # (optional) exported running-configs
    ├── HQ-FWL.txt
    ├── BRANCH-FWL.txt
    ├── HQ-MLSW1.txt
    ├── HQ-MLSW2.txt
    ├── B-MLSW1.txt
    └── B-MLSW2.txt
```

Rename the folders and files to match your repository.

---

## How to Open the Project

1. Install **Cisco Packet Tracer** (version 8.x or later, to support ASA 5506-X and WLC 2504).
2. Clone this repository:
   ```bash
   git clone https://github.com/<your-username>/<your-repo>.git
   ```
3. Open the `.pkt` file in Packet Tracer.
4. Wait about a minute for OSPF to converge, then test with a PC set to DHCP.

> If the network seems stuck after opening the file (for example a serial link that is up but passing no traffic), power-cycle the affected router.

---

## Author

**Your Name**
Network Security Engineer (project)
GitHub: [@your-username](https://github.com/your-username)

---

## License

Add a license of your choice (for example MIT) or state that the project is for educational use.
