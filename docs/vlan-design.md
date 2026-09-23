# VLAN Design

## Overview

The homelab uses VLAN segmentation to separate management, servers, Kubernetes, storage, IoT and other services.

OPNsense provides Layer 3 routing between VLANs and firewall rules control which networks are allowed to communicate.

The goal is to reduce unnecessary access between systems and create clear security zones.

---

## VLAN Structure

| VLAN | Name | Purpose |
|---:|---|---|
| 10 | Management | Proxmox, OPNsense, Cisco and administration clients |
| 15 | Kubernetes | K3s cluster nodes |
| 18 | HashiCorp Vault | Secrets management |
| 20 | DNS | Pi-hole and DNS services |
| 25 | Minisforum | Minisforum Proxmox host and related services |
| 30 | Honeypot | Security lab / honeypot |
| 35 | Clawbot | Lab service |
| 40 | NAS | Storage, NFS and SMB |
| 50 | Jetson | NVIDIA Jetson / AI workloads |
| 55 | LIA | LIA / test environment |
| 60 | AP | Access point network |
| 70 | IoT | IoT devices |
| 80 | Guest | Guest network |
| 90 | ASUS Management | ASUS/AP administration |
| 95 | Ansible | Ansible / Bastion host |

---

## Management VLAN

VLAN 10 is the dedicated infrastructure management network.

Known systems include:

| System | Address |
|---|---|
| OPNsense | `172.16.10.1/27` |
| Proxmox VE | `172.16.10.3/27` |
| Cisco WS-C2960L | `172.16.10.10/27` |

The administration laptop connects to VLAN 10 through an access port on the TP-Link switch.

Management traffic is transported as tagged VLAN 10 traffic across trunk links.

VLAN 1 is not used for infrastructure management.

---

## Access and Trunk Design

TP-Link access ports are assigned to a single VLAN.

| Port | VLAN | Function |
|---:|---:|---|
| 1 | 10 | Management laptop |
| 2 | 15 | Kubernetes |
| 3 | 30 | Honeypot |
| 4 | 35 | Lab service |
| 5 | 50 | Jetson |
| 6 | 10 | Management |

Port 7 and Port 8 operate as trunks.

```text
Port 7
TP-Link -> Cisco
Tagged VLANs: 10,15,18,20,25,30,35,40,50,55,60,95
Native/PVID: 999
Port 8
TP-Link -> Proxmox
Tagged VLANs: 10,15,18,20,25,30,35,40,50,55,60,95
Native/PVID: 998

VLAN 998 and VLAN 999 are unused black-hole VLANs for untagged trunk traffic.

Minisforum Separation

The Minisforum Proxmox host is placed in VLAN 25.

VLAN: 25
IP:   172.16.25.3/27

The Cisco trunk toward Minisforum allows:

15,18,25,40,50,55

VLAN 10 is intentionally not included.

This prevents direct Layer 2 access from the Minisforum trunk to the management VLAN.

Management access instead follows:

Management Laptop
      |
    VLAN 10
      |
      v
   OPNsense
      |
 Firewall policy
      |
      v
    VLAN 25
      |
      v
Minisforum Proxmox
Firewall Segmentation

VLANs are separated at Layer 2, while OPNsense controls communication between them at Layer 3.

Examples:

VLAN 10 -> VLAN 25
Management access controlled by firewall rules

VLAN 25 -> VLAN 10
Restricted

VLAN 25 -> DNS
Allowed to Pi-hole

VLAN 25 -> NAS
Allowed according to dedicated rules

Guest / IoT
Separated from infrastructure networks

This follows a least-privilege approach where communication is only enabled when required.

Design Goal

The VLAN design provides separation between:

Management
    |
Servers
    |
Kubernetes
    |
Storage
    |
Security Lab
    |
IoT / Guest

The result is a segmented network where a compromised device in one VLAN does not automatically receive access to the rest of the infrastructure.
