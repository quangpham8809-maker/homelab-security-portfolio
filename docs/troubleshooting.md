# Troubleshooting

This document contains a few real networking problems encountered while building and hardening the homelab.

---

## 1. Proxmox reachable but OPNsense was not

### Problem
The laptop could access Proxmox but could not reach OPNsense or the Internet.

### Cause
Management traffic was split between untagged VLAN 1 and tagged VLAN 10.

### Solution
Management was moved completely to VLAN 10.

```text
Proxmox:
vmbr1.10 -> VLAN 10 -> 172.16.10.3/27

OPNsense:
VLAN 10 -> 172.16.10.1/27

TP-Link access ports were also changed to PVID 10.

Result

Both Proxmox, OPNsense and Internet became reachable from the management laptop.

2. Nginx Proxy Manager lost network access
Problem

Nginx Proxy Manager stopped receiving an IPv4 address after the management VLAN change.

Cause

The LXC container was connected to vmbr1 without a VLAN tag.

Solution

The container network was changed to:

Bridge: vmbr1
VLAN Tag: 10
Result

Nginx Proxy Manager received network connectivity again in VLAN 10.

3. Cisco trunk changes returned after reboot
Problem

VLAN 10 reappeared on Gi0/20 and VLAN 1 reappeared on Gi0/24.

Cause

The running configuration had been changed, but the final configuration was not saved correctly before reload.

Solution
Gi0/20:
remove VLAN 10

Gi0/24:
remove VLAN 1

Then:

copy running-config startup-config

The startup configuration was verified afterwards.

Result
Gi0/20 -> 15,18,25,40,50,55
Gi0/24 -> 10,15,18,20,25,30,40,50,55,60,95
4. TP-Link port could not be removed from VLAN 1
Problem

The switch displayed:

Port 3 is not member of any VLAN

when trying to remove Port 3 from VLAN 1.

Cause

Port 3 had PVID 30 but was not yet an untagged member of VLAN 30.

Solution

First:

VLAN 30
Port 3 -> Untagged
PVID -> 30

Then Port 3 could safely be removed from VLAN 1.

The same process was used for Port 5 and VLAN 50.

Result

Each access port now belongs only to its intended VLAN.
