# Nätverksflöden

Heldragna pilar visar trafik som uttryckligen tillåts av policykopiorna. Streckade pilar visar tillåten generell DNS- eller publik egress där faktisk destination/funktion behöver verifieras. Diagrammet visar policyavsikt, inte observerad kommunikation.

```mermaid
flowchart LR
  T[Traefik<br/>kube-system] -->|Tillåten TCP 80| F[Frontend]
  T -->|Tillåten TCP 8000| B[Backend]
  F -->|Tillåten TCP 8000| B
  B -->|Tillåten TCP 3306| DB[MariaDB]
  B -->|Tillåten TCP 6379| R[Redis]
  C[Send-reminders / watch-renewer] -->|Tillåten TCP 8000| B
  G[GDPR-jobb] -->|Tillåten TCP 3306| DB
  S[Statistikjobb] -->|Tillåten TCP 3306| DB
  RC[Retention-jobb] -->|Tillåten TCP 3306| DB
  I[Instagram-refresher] -->|Tillåten TCP 6379| R
  F -.->|DNS UDP/TCP 53; ej klustertestat| DNS[CoreDNS]
  B -.->|DNS UDP/TCP 53; ej klustertestat| DNS
  C -.->|DNS UDP/TCP 53; ej klustertestat| DNS
  G -.->|DNS UDP/TCP 53; ej klustertestat| DNS
  S -.->|DNS UDP/TCP 53; ej klustertestat| DNS
  RC -.->|DNS UDP/TCP 53; ej klustertestat| DNS
  I -.->|DNS UDP/TCP 53; ej klustertestat| DNS
  B -.->|Publik TCP 443/587/465; destination bred, ej verifierad| EXT[Externa tjänster]
  G -.->|Publik TCP 443; destination bred, ej verifierad| EXT
  S -.->|Publik TCP 443; destination bred, ej verifierad| EXT
  I -.->|Publik TCP 443; destination bred, ej verifierad| EXT
```

Varje pil motsvarar en deklarerad regel. DNS är tillåten i manifest men dess faktiska funktion beror bland annat på klustrets DNS- och CNI-konfiguration. Extern egress anges med breda publika CIDR-regler och är inte verifierad mot tjänstspecifika IP-adresser. Ingen trafik ska tolkas som faktiskt uppmätt.

