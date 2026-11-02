# Nätverksflöden och segmentering

Diagrammen visar trafik som deklareras av den granskade policyuppsättningen. De visar inte observerad trafik. För kommunikation mellan två pods som båda isoleras krävs både källans egressregel och destinationens ingressregel. Trafik och aktiv CNI-verkställighet har inte testats.

## Diagram A – Applikationskommunikation

```mermaid
flowchart LR
  T[Traefik<br/>kube-system] -->|TCP 80<br/>Ingress till Frontend tillåts| F[Frontend<br/>Nginx]
  T -->|TCP 8000<br/>Ingress till Backend tillåts| B[Backend<br/>FastAPI]
  F -->|TCP 8000<br/>Frontend egress + Backend ingress| B
  B -->|TCP 3306<br/>Backend egress + MariaDB ingress| DB[MariaDB]
  B -->|TCP 6379<br/>Backend egress + Redis ingress| R[Redis]
```

Flödena är deklarerade av policies. Traefik-poddens egress omfattas inte av applikationsnamespacets default-deny-policy, eftersom Traefik väljs från `kube-system`; diagrammet beskriver därför Traefiks ingressväg utifrån mottagarens ingressregel.

## Diagram B – CronJobs och externa tjänster

```mermaid
flowchart LR
  subgraph Jobs[Jobb och schemalagda arbetslaster]
    G[GDPR-jobb]
    S[Statistikjobb]
    RC[Retention-jobb]
    I[Instagram-refresher]
  end
  DNS[CoreDNS<br/>kube-system]
  DB[MariaDB]
  R[Redis]
  EXT[Externa tjänster<br/>publika IP-adresser]

  G -->|TCP 3306<br/>Egress + DB ingress| DB
  S -->|TCP 3306<br/>Egress + DB ingress| DB
  RC -->|TCP 3306<br/>Egress + DB ingress| DB
  I -->|TCP 6379<br/>Egress + Redis ingress| R

  G -.->|UDP/TCP 53| DNS
  S -.->|UDP/TCP 53| DNS
  RC -.->|UDP/TCP 53| DNS
  I -.->|UDP/TCP 53| DNS

  G -.->|TCP 443<br/>Destination inte begränsad till API| EXT
  S -.->|TCP 443<br/>Destination inte begränsad till API| EXT
  I -.->|TCP 443<br/>Destination inte begränsad till API| EXT
```

GDPR-, statistik- och Instagram-policies använder breda publika CIDR-regler med undantag för listade nät. Källkommentarerna anger Google Calendar/OAuth, Telegram respektive Instagram/Graph API som avsikter, men NetworkPolicy-reglerna begränsar inte destinations-IP till dessa tjänster. Streckade pilar anger policytillåten trafik vars faktiska funktion inte har verifierats i klustret.

## Diagram C – Säkerhetssegmentering

```mermaid
flowchart LR
  DD[Default Deny<br/>isolerar ingress och egress<br/>för alla pods i namespace]
  F[Frontend]
  B[Backend]
  DB[MariaDB]
  DNS[CoreDNS]

  B -->|Tillåten enligt policy<br/>TCP 3306; båda riktningar matchar| DB
  F -.->|Blockerad enligt granskad regelmängd<br/>Frontend egress till DB saknas<br/>och DB ingress från Frontend saknas| DB
  B -.->|DNS tillåten i policy<br/>UDP/TCP 53; faktisk trafik ej verifierad| DNS
```

Default Deny betyder att matchade pods isoleras; övriga granskade policies lägger till de tillåtelser som visas. Frontend → MariaDB är ett exempel på en anslutning som saknar både frontend-egressundantag och DB-ingressundantag i den granskade regelmängden. Därför skulle flödet blockeras om den granskade uppsättningen är fullständig för namespace och CNI verkställer reglerna. Blockeringen har inte verifierats med ett praktiskt nätverkstest. Andra policies utanför granskningen kan addera tillåtelser.

### Teckenförklaring

- Heldragen pil: deklarerat tillåtet flöde enligt policyreglerna.
- Streckad pil: deklarerad blockering eller tillåten policyväg vars faktiska trafik inte har verifierats; etiketten anger vilket.
- Diagrammen beskriver policyavsikt och policyanalys, inte mätresultat från ett aktivt kluster.
