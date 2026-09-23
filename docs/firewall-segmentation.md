# Firewall Segmentation

## Overview

OPNsense is used as the central firewall and Layer 3 router between the VLANs in my homelab.

The firewall design follows a least-privilege approach:

- VLANs are isolated by default
- Only required services are allowed
- Aliases are used instead of repeating IP addresses and port numbers
- Access to the firewall itself is restricted
- Internal networks are protected from unnecessary cross-VLAN communication
- Internet access is limited to required protocols

OPNsense evaluates rules using a first-match policy, so rule order is important.

---

## Alias-Based Firewall Design

To keep the firewall rules easier to manage, I use aliases for hosts, networks and ports.

Examples include:

```text
Vault_IP
Ansible_Pi
NAS_IP
K3s_Cluster
K3s_Cluster_master
DNS_Server_Pihole
adguard_DNS
RFC1918_Private_IPv4

SSH
DOMAIN
NTP_HTTP_HTTPS_DNS
BAD_PORTS_OUT

This makes the rules easier to understand and update.

For example, instead of creating separate rules for each Kubernetes node, the alias:

K3s_Cluster

can contain all required K3s hosts.

The firewall rule can then simply reference that alias.

HashiCorp Vault - VLAN 18

Vault is placed in a dedicated VLAN and is not allowed unrestricted access to the internal network.

Important rules include:

Source	Destination	Port	Action
Vault network	This Firewall	Any	Block
Vault_IP	K3s_Cluster_master	TCP 6443	Allow
Vault network	DNS servers	53	Allow
Vault network	External networks	NTP / HTTP / HTTPS / DNS	Allow

The Kubernetes API rule allows Vault to communicate with the K3s control plane on:

TCP 6443

Other internal access is restricted.

This allows Vault to perform its required function without giving the entire VLAN unrestricted access to other internal systems.

Ansible / Bastion - VLAN 95

The Ansible host requires access to several infrastructure systems for automation.

Only specific services are allowed.

Examples:

Source	Destination	Port	Purpose
Ansible_Pi	Vault_IP	SSH 22	Vault administration
Ansible_Pi	NAS_IP	SSH 22	Backup / synchronization
Ansible_Pi	K3s_Cluster	SSH 22	Node administration
Ansible_Pi	K3s_Cluster	ICMP	Connectivity testing
Ansible network	DNS servers	53	DNS resolution

Internet access is restricted to the protocols included in the alias:

NTP_HTTP_HTTPS_DNS

This means the Ansible host can perform administration tasks without receiving unrestricted access to every internal service.

Honeypot - VLAN 30

The honeypot VLAN is more heavily restricted because the system is intentionally exposed to potentially hostile activity.

The main policy is:

Honeypot
   |
   +--> Firewall          BLOCK
   |
   +--> Internal RFC1918  BLOCK
   |
   +--> Internet          Limited access
   |
   +--> Everything else   BLOCK

The honeypot is prevented from reaching private RFC1918 networks.

Internet access is limited to required protocols such as:

NTP
HTTP
HTTPS
DNS

A separate alias is also used to block unwanted outbound ports:

BAD_PORTS_OUT

A final block rule prevents traffic that has not been explicitly allowed.

This reduces the risk that a compromised honeypot could be used to attack other systems in the homelab.

Security Model

The overall firewall design follows this principle:

Default isolation
       +
Explicit allow rules
       +
Aliases
       +
Required ports only
       +
Final deny

Instead of allowing full communication between VLANs, each network receives only the access required for its function.

Examples:

Vault -> K3s API      TCP 6443
Ansible -> K3s        SSH 22
Ansible -> NAS        SSH 22
VLANs -> DNS          UDP/TCP 53
Honeypot -> Internal  BLOCK

This makes the firewall configuration easier to audit and reduces unnecessary attack paths between security zones.
