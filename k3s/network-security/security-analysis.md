# Säkerhetsanalys av Kubernetes NetworkPolicies

## Omfattning och evidens

Analysen gäller den lokalt granskade policyuppsättningen: 15 NetworkPolicy-resurser i 12 policyfiler samt en separat Kustomization-fil. Produktionsoverlayen tilldelar applikationsresurserna namespace `nagelsalong`. Analysen är statisk och bygger på manifestinnehåll och tillgängliga workload-manifest. Den beskriver deklarerade regler, inte observerade nätverksflöden.

**Verifierat i manifest:** selectors, policytyper, tillåtna portar/protokoll, deklarerad default-deny och att policyreglerna är additiva.

**Tidigare rapporterade kontroller:** lokal Kustomize-rendering av policykatalogen samt jämförelse av funktionellt YAML-innehåll mellan original och dokumentationskopior.

**Ej verifierat:** aktiva pods/etiketter, faktisk pod-kommunikation, CNI-verkställighet, komplett policyuppsättning utanför granskad katalog, RBAC och databasbehörigheter.

Kubernetes NetworkPolicies är additiva. För en anslutning mellan två isolerade pods måste källans egress tillåta utgående trafik och destinationens ingress tillåta inkommande trafik. För externa eller andra-namespaces destinationspods beror vilken sida som är isolerad på deras egna policies.

## Medium – Bred publik egress

### Vad som är deklarerat

Följande policies har egressregler med `ipBlock.cidr: 0.0.0.0/0`, undantag för listade privata/särskilda nät och begränsade portar:

- `allow-backend-egress`: TCP 443, 587 och 465.
- `allow-gdpr-anonymize-egress`: TCP 443.
- `allow-instagram-refresher-egress`: TCP 443.
- `allow-statistics-egress`: TCP 443.

### Risk

`0.0.0.0/0` omfattar alla IPv4-destinationer innan undantagen tillämpas. Portbegränsning begränsar tjänstporten, men inte den publika destinationen. En komprometterad pod kan därför försöka kommunicera med oönskade publika servrar på tillåtna portar, vilket kan ge en kanal för datautflöde eller nå andra publika tjänster. De listade CIDR-undantagen är inte en destinationsallowlist för specifika API:er. Exakt räckvidd kan påverkas av CNI, routing, service-CIDR och NAT.

### Rekommendation

Bekräfta externa beroenden per workload. Om plattformen stödjer det, överväg en kontrollerad egressproxy eller annan destinationskontroll och dokumentera dess begränsningar. Verifiera service-CIDR, NAT och CNI-beteende i en isolerad miljö innan någon slutsats dras om klusternätets åtkomst.

**Status:** Regelbredden är verifierad i manifest. Faktisk nåbarhet och exploaterbarhet är ej testade.

## Low – Åtkomst till MariaDB och Redis

### Deklarerade klienter

MariaDB med selector `app=rosanails-db` tillåter TCP 3306 från fyra klienttyper via separata ingressresurser:

- backend, via `allow-backend-to-db`;
- GDPR-jobb, via `allow-gdpr-anonymize-to-db`;
- statistikjobb, via `allow-statistics-to-db`;
- retention-jobb, via `allow-retention-cleanup-to-db`.

Motsvarande egress till MariaDB TCP 3306 finns för backend och de tre jobben. Tillgängliga workload-manifest visar DB-konfiguration för dessa komponenter; GDPR-, statistik- och retention-jobben innehåller även init-containrar som väntar på MariaDB. Utifrån de manifesterna verkar åtkomsten motiverad. Faktiskt användningsbehov och DB-kontobehörigheter är inte verifierade.

Redis med selector `app=redis` tillåter TCP 6379 från backend och Instagram-refresher. Backend har motsvarande egressregel. Instagram-refresher har Redis-host/port konfigurerade och motsvarande egressregel. Detta ser motiverat ut utifrån tillgängliga manifest.

### Risk

En komprometterad tillåten pod kan ansluta till databasen eller Redis på nätverksnivå. Nätverksreglerna anger inte vilka SQL-operationer, Redis-kommandon eller data som klienten får använda. Om backend komprometteras finns deklarerade vägar till både MariaDB och Redis; kompromettering av ett av batchjobben ger en väg till MariaDB. Detta är avsiktliga beroenden, men utgör möjliga pivotvägar.

### Rekommendation

Behåll separata applikations- och databasbehörigheter, begränsa DB-konton efter arbetslast där det är praktiskt, och kontrollera att inga andra pods delar de använda `app`-etiketterna. Granska jobben och klientlistan vid förändringar. Inga NetworkPolicy-regler har ändrats i denna granskning.

**Status:** Tillåtna klienttyper och portar verifierade statiskt. Faktiska anslutningar, etikettunikhet och DB-/Redis-autentisering är ej testade.

## Low, villkorat – Pod-etiketter och RBAC

### Vad som är deklarerat

Policies väljer pods med `app`-etiketter. PodSelectors utan namespaceSelector avser pods i samma namespace som policyn. DNS- och Traefikreglerna kombinerar namespaceSelector och podSelector; båda villkoren måste matcha.

### Risk

NetworkPolicy identiteten kopplas till etiketter, inte till arbetslastens namn eller service account. Om en användare med för generös RBAC får skapa pods i samma namespace och kan ange etiketter som exempelvis `app=rosanails-backend`, kan den podden få de nätverksrättigheter som följer av matchande policies. Detta är en villkorad risk: granskningen visar inte vilka användare som har RBAC-behörighet att skapa pods eller ändra workloads.

### Rekommendation

Granska namespace-RBAC och begränsa vem som får skapa eller ändra Pods, Deployments, Jobs och CronJobs. Använd policyer för admission/kontroll av etiketter om klustermodellen kräver det. Jämför selectors med faktiska workload-labels i en godkänd, read-only kontroll.

**Status:** Selectorvillkoren är verifierade i manifest. Möjlighet till etikettmissbruk och RBAC-risk är ej verifierade.

## Övriga kontrollerade områden

- **Default Deny:** `default-deny` väljer alla pods och anger både Ingress och Egress. Den finns med i policykatalogens Kustomization. Att den är aktivt applicerad och verkställs är ej verifierat.
- **Frontend och backend:** frontend egress tillåter DNS samt backend TCP 8000; backend ingress tillåter Traefik, frontend och två namngivna jobb TCP 8000. Traefik till frontend TCP 80 deklareras. Konfigurationen visar ingen deklarerad direkt frontend → MariaDB/Redis-väg i den granskade uppsättningen.
- **DNS:** utvalda workloads tillåter UDP/TCP 53 till pods i `kube-system` med etiketten `k8s-app=kube-dns`. Faktisk DNS-routing, till exempel eventuell NodeLocal DNS, är ej verifierad.
- **Redis:** ingress tillåts från backend och Instagram-refresher på TCP 6379. Övriga policies kan addera ytterligare tillåtelser; fullständiga klusterpolicies utanför granskad katalog ingår inte.
- **Policyöverlapp:** DB-ingressregler är separata klientundantag, inte dubbletter av samma klientregel. DNS-undantag upprepas per workload. Inga oavsiktliga direkta vägar från frontend till DB/Redis identifierades i de granskade 15 resurserna.

## Slutsats

Konfigurationen deklarerar en default-deny-bas med separata undantag för identifierade applikationsflöden. Den tydligaste förbättringsmöjligheten är att granska destinationerna för de fyra breda publika egressreglerna. DB- och Redis-vägarna verkar behövas enligt workload-manifesten, men bör fortsatt betraktas som möjliga pivotvägar vid kompromettering av en tillåten klient. RBAC- och CNI-status behöver verifieras innan operativa slutsatser dras.
