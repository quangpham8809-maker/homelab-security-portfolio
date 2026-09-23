# Network Architecture

## Overview

This project documents the network architecture of my cybersecurity homelab.

The environment is designed around network segmentation, dedicated management networks, firewall-controlled inter-VLAN communication, virtualization and infrastructure hardening.

The core components are:

- Proxmox VE
- OPNsense
- Cisco WS-C2960L
- TP-Link TL-SG108E
- Minisforum Proxmox server
- Raspberry Pi Kubernetes/K3s nodes
- NAS
- NVIDIA Jetson
- HashiCorp Vault
- Pi-hole
- Nginx Proxy Manager
- ASUS access point infrastructure

The main security principle is that different types of systems are separated into dedicated VLANs and communication between them is controlled by OPNsense.

---

# High-Level Architecture

```text
                         Internet
                            |
                            v
                     +-------------+
                     |  OPNsense   |
                     |  Firewall   |
                     +------+------+
                            |
                            |
                     Proxmox VE
                    Intel i3-N305
                            |
                         enp3s0
                            |
                          vmbr1
                    VLAN-aware trunk
                            |
                            v
                  +-------------------+
                  | TP-Link TL-SG108E |
                  +-------------------+
                    |               |
                 Port 7           Port 8
                    |               |
                    |               +------> Proxmox / OPNsense trunk
                    |
                    v
             +----------------+
             | Cisco WS-C2960L|
             +----------------+
                    |
          +---------+-----------+----------------+
          |         |           |                |
        K3s       NAS       Minisforum        Ansible
       VLAN15    VLAN40      VLAN25           VLAN95
