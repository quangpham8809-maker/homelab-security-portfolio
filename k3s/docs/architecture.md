# Arkitektur

## Omfattning

Dokumentet beskriver en sanerad referensarkitektur. Adressrymder och namn är exempel och ska inte läsas som dokumentation av en verklig miljö.

## Lager och ansvar

| Lager | Exempelteknik | Säkerhetsansvar |
|---|---|---|
| Virtualisering | Proxmox | isolering av virtuella maskiner, patchning och resursgränser |
| Nätverkskant | OPNsense | ingress/egress-filtrering, VLAN-routing och loggning |
| Privat administration | Tailscale | autentiserad åtkomst till managementytor utan publik exponering |
| Kubernetes | K3s | schemaläggning, RBAC, Pod Security och workload-isolering |
| Ingress | Traefik + MetalLB | kontrollerad tjänsteexponering och TLS-terminering |
| Lagring | Longhorn | replikerad persistent lagring och backupmål |
| Leverans | GitHub + Argo CD + Kustomize | spårbara, deklarativa och reproducerbara ändringar |
| Observability | Prometheus + Grafana | mätvärden, dashboards och larm |
| Automation | Ansible + Semaphore | repeterbart underhåll med separerad credential-hantering |

## Exempel på zoner

- `10.10.10.0/24` — management; endast administratörer och automation.
- `10.10.20.0/24` — control plane och infrastruktur.
- `10.10.30.0/24` — workers och applikationslaster.
- `10.10.40.0/24` — lagring och backup.
- `10.10.50.0/24` — klienter.
- `10.10.60.0/24` — IoT; ingen initierad åtkomst till management.

IP-adresserna illustrerar segmentering och motsvarar inte den privata miljön.

## Tillitsgränser

1. OPNsense är gränsen mellan internet och interna zoner. Endast avsedda publicerade tjänster får passera.
2. Routing mellan VLAN är nekad som utgångsläge och öppnas per källa, mål, protokoll och port.
3. Traefik är applikationens externa ingång. Backend, databas och Redis exponeras inte via LoadBalancer eller Ingress.
4. Kubernetes NetworkPolicies begränsar öst-västlig trafik även när workloads delar kluster.
5. GitOps-identiteten ska bara kunna hantera uttryckligen tillåtna namespaces och resurstyper.
6. Backupmålet ligger i separat zon och ska använda en identitet som inte ger klustret generell skrivåtkomst till äldre recovery points.

## Hög tillgänglighet och felmoder

Tre control-plane-noder minskar beroendet av en enskild nod. Flera workers och spridningsregler reducerar applikationspåverkan vid nodfel. Longhorn-replikering hanterar vissa disk- och nodfel, men logisk korruption och felaktig radering kräver oberoende backup.

Tillgänglighet får inte förväxlas med säkerhet: ett fel kan replikeras snabbt. Därför kombineras redundans med versionshantering, retention, övervakning och restore-test.
