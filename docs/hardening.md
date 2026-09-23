
# Network Hardening

## Overview

This document summarizes the main network hardening implemented in my cybersecurity homelab.

The goal was to protect management interfaces, reduce unnecessary Layer 2 access and create clearer separation between infrastructure, servers and lab networks.

The environment uses Proxmox VE, OPNsense, Cisco WS-C2960L and TP-Link TL-SG108E.

---

## Management and VLAN Hardening

Management traffic was moved from the default VLAN to a dedicated management network:

| System | VLAN | Address |
|---|---:|---|
| OPNsense | 10 | `172.16.10.1/27` |
| Proxmox VE | 10 | `172.16.10.3/27` |
| Cisco switch | 10 | `172.16.10.10/27` |

Proxmox management now uses `vmbr1.10` instead of assigning an IP directly to the base bridge.

```text
vmbr1
└── VLAN-aware trunk

vmbr1.10
└── VLAN 10
    172.16.10.3/27

Cisco management was moved from Vlan1 to Vlan10.

Vlan1  -> unassigned / administratively down
Vlan10 -> 172.16.10.10/27

VLAN 1 is no longer used for infrastructure management.

Trunk and Native VLAN Hardening

The important trunk links no longer use VLAN 1 as their active native VLAN.

Link	Tagged VLANs	Native / Black-hole VLAN
TP-Link Port 7 → Cisco Gi0/24	10,15,18,20,25,30,35,40,50,55,60,95	999
TP-Link Port 8 → Proxmox	10,15,18,20,25,30,35,40,50,55,60,95	998
Cisco Gi0/20 → Minisforum	15,18,25,40,50,55	999

VLAN 998 and VLAN 999 are not used by normal clients or servers. They are used as black-hole VLANs for unexpected untagged traffic.

VLAN 10 is intentionally removed from the Cisco trunk toward the Minisforum server.

The Minisforum host therefore cannot obtain direct Layer 2 access to the management VLAN.

Management Laptop
      VLAN 10
         |
         v
      OPNsense
    Firewall rules
         |
         v
      VLAN 25
         |
         v
Minisforum Proxmox
172.16.25.3

Administrative access between VLAN 10 and VLAN 25 is instead controlled by OPNsense.

Access and Firewall Hardening

TP-Link access ports are assigned only to their required VLAN.

Port	VLAN	Mode
1	10	Access
2	15	Access
3	30	Access
4	35	Access
5	50	Access
6	10	Access

Unused Cisco switch ports are administratively disabled with shutdown.

OPNsense controls communication between the VLANs and uses explicit firewall rules instead of allowing unrestricted communication between internal networks.

For example, VLAN 25 can use required services such as DNS and NAS access while management access is restricted.

Verification

Cisco trunk configuration is verified with:

show interfaces trunk
show vlan brief
show ip interface brief

Management connectivity is tested from the administration laptop:

ping 172.16.10.1
ping 172.16.10.3
ping 172.16.10.10

After the hardening changes, management access was successfully verified to:

OPNsense
Proxmox VE
Cisco switch
TP-Link switch
Minisforum Proxmox
ASUS/AP infrastructure
Internet
Result

The homelab now uses a dedicated management VLAN, restricted trunk VLANs, black-hole native VLANs, firewall-controlled inter-VLAN routing and disabled unused switch ports.

The main security model is:

VLAN Segmentation
        +
Dedicated Management VLAN
        +
OPNsense Firewall
        +
Restricted Trunks
        +
Native VLAN Hardening
        +
Disabled Unused Ports

This provides a more segmented and least-privilege network design while keeping the environment manageable from the dedicated administration network.
