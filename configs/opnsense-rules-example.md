# OPNsense Firewall Rules - Example

## Overview

OPNsense is used as the central firewall and Layer 3 router between the VLANs in my homelab.

The firewall policy follows a least-privilege approach:

- VLANs are isolated by default
- Only required traffic is explicitly allowed
- Aliases are used for hosts, networks and ports
- Internal RFC1918 networks are protected from unnecessary access
- Access to the firewall itself is restricted
- A final deny policy blocks traffic that has not been explicitly permitted

This keeps the rule set easier to understand and maintain.

---

## Alias-Based Rules

Instead of repeating IP addresses and port numbers in every firewall rule, aliases are used.

Examples:

```text
Vault_IP
Ansible_Pi
NAS_IP
K3s_Cluster
K3s_Cluster_master
DNS_Server_Pihole
adguard_DNS
RFC1918_Private_IPv4
BAD_PORTS_OUT

Port aliases are also used, for example:

SSH
DOMAIN
NTP_HTTP_HTTPS_DNS

This allows a rule to reference a logical name instead of a single IP address or port.

Example 1 - HashiCorp Vault VLAN 18

Vault is placed in a dedicated VLAN and only receives the network access required for its function.

Example policy:

Source	Destination	Port	Action
Vault network	This Firewall	Any	Block
Vault_IP	K3s_Cluster_master	TCP 6443	Allow
Vault network	DNS servers	TCP/UDP 53	Allow
Vault network	External networks	NTP / HTTP / HTTPS / DNS	Allow

Example flow:

Vault VLAN 18
     |
     +--> OPNsense firewall      BLOCK
     |
     +--> K3s API :6443          ALLOW
     |
     +--> DNS :53                ALLOW
     |
     +--> Internet required ports ALLOW
     |
     +--> Other internal networks BLOCK

The main goal is to allow Vault to communicate with the K3s control plane without giving the entire VLAN unrestricted access to other internal systems.

Example 2 - Ansible VLAN 95

The Ansible / Bastion host requires access to selected infrastructure systems for administration and automation.

Example rules:

Source	Destination	Port	Purpose
Ansible_Pi	Vault_IP	TCP 22	Vault administration
Ansible_Pi	NAS_IP	TCP 22	Backup / synchronization
Ansible_Pi	K3s_Cluster	TCP 22	Node administration
Ansible_Pi	K3s_Cluster	ICMP	Connectivity testing
Ansible network	DNS servers	TCP/UDP 53	DNS
Ansible network	External networks	NTP / HTTP / HTTPS / DNS	Internet access

This gives the automation host access to the systems it manages without allowing unrestricted communication to every internal VLAN.

Example 3 - Honeypot VLAN 30

The honeypot network is more restrictive because the system may intentionally interact with hostile traffic.

Example policy:

Honeypot VLAN 30
       |
       +--> This Firewall        BLOCK
       |
       +--> RFC1918 networks     BLOCK
       |
       +--> BAD_PORTS_OUT        BLOCK
       |
       +--> NTP/HTTP/HTTPS/DNS   ALLOW
       |
       +--> Everything else      BLOCK

The honeypot is prevented from reaching internal private networks.

Only limited outbound services are allowed when required.

This reduces the risk that a compromised honeypot could be used to attack other systems in the homelab.

Rule Order

OPNsense evaluates rules using first-match behavior.

This means rule order is important.

A simplified rule design is:

1. Block access to the firewall
2. Allow explicitly required services
3. Allow DNS / required external services
4. Block internal networks when not required
5. Final deny

For example:

ALLOW  Vault -> K3s API TCP 6443
ALLOW  Vault -> DNS TCP/UDP 53
BLOCK  Vault -> Internal networks
ALLOW  Vault -> Required Internet ports
Security Principle

The firewall design follows:

Default isolation
       +
Aliases
       +
Explicit allow rules
       +
Required ports only
       +
Final deny

The goal is not to allow full communication between VLANs.

Instead, each VLAN receives only the access required for its function.

This reduces unnecessary attack paths and makes the firewall rules easier to audit.
