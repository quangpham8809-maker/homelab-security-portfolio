
# Kubernetes Network Security – Rosa Nails

## Projektöversikt

Detta arbetsprov granskar hur Kubernetes NetworkPolicies begränsar nätverkstrafik mellan pods i applikationen Rosa Nails. Utan nätverkssegmentering kan en komprometterad pod ofta försöka nå fler interna tjänster än den behöver. NetworkPolicies uttrycker tillåtna trafikflöden på podnivå, förutsatt att klustrets nätverksplugin verkställer dem.

Filerna här är sanerade dokumentationsexempel. De är kopior för portföljgranskning, inte produktionsfiler och inte en källa för GitOps.

## Syfte

Målet är att visa inventering, statisk säkerhetsanalys och dokumentation av en befintlig policyuppsättning. I produktionsoverlayen sätts namespace `nagelsalong`. En default-deny-policy isolerar ingress och egress för alla pods i det namespace som policyn tilldelas. Övriga NetworkPolicies öppnar trafik selektivt.

## Teknisk miljö

Konfigurationen refererar till K3s/Kubernetes, Traefik i `kube-system`, Nginx-frontend, FastAPI-backend, MariaDB, Redis samt schemalagda arbetslaster. Portföljkopiorna har inga interna IP-adresser, hemligheter eller domännamn. De faktiska arbetslaster och nätverksflödena verifieras inte mot ett aktivt kluster i denna granskning.

## Säkerhetsprinciper

- **Zero Trust:** utgångspunkt att varje anslutning ska motiveras och begränsas. NetworkPolicies är ett nätverkskontrollager och utgör inte ensamma en fullständig Zero Trust-arkitektur.
- **Default Deny:** `default-deny` väljer alla pods i namespace och isolerar båda riktningarna.
- **Least Privilege:** separata regler begränsar valda klienter och portar.
- **Network Segmentation:** frontend, backend, databas, Redis och jobb får olika kommunikationsvägar.
- **Ingress och Egress:** ingress gäller trafik in till valda pods; egress gäller trafik ut från valda pods.
- **Defense in Depth:** nätverkspolicyn kompletterar autentisering, TLS, applikationsbehörigheter, hemlighetshantering, brandväggar och andra skydd.

Kubernetes NetworkPolicies är additiva: en anslutning tillåts om någon policy som isolerar destinationspodden (ingress) respektive källpodden (egress) tillåter den. En regel i en policy tar inte bort rättigheter som en annan policy beviljar.

## Policyöversikt

| NetworkPolicy | Skyddar | Tillåten trafik | Säkerhetssyfte |
|---|---|---|---|
| `default-deny` | Alla pods i tilldelat namespace | Ingen ingress eller egress i sig | Grundisolering; övriga policies öppnar uttryckliga undantag |
| `allow-traefik-to-frontend` | Frontend | Traefik från `kube-system` → TCP 80 | Begränsar klienter till ingresskomponenten |
| `allow-frontend-egress` | Frontend | CoreDNS UDP/TCP 53; backend TCP 8000 | Tillåter namnuppslagning och API-trafik |
| `allow-traefik-frontend-and-cronjobs-to-backend` | Backend | Traefik, frontend, send-reminders och watch-renewer → TCP 8000 | Begränsar inkommande API-klienter |
| `allow-backend-egress` | Backend | DNS UDP/TCP 53; DB TCP 3306; Redis TCP 6379; offentlig TCP 443/587/465 | Tillåter interna beroenden och externa API-/SMTP-anslutningar |
| `allow-cronjobs-egress` | Två jobb: send-reminders och watch-renewer | DNS UDP/TCP 53; backend TCP 8000 | Begränsar schemalagda jobbs utgående trafik |
| `allow-backend-to-db` | MariaDB | Backend → TCP 3306 | Begränsar databasåtkomst |
| `allow-gdpr-anonymize-egress` | GDPR-anonymiseringsjobb | DNS, DB TCP 3306, publik TCP 443 | Tillåter DNS, databas och extern HTTPS enligt källkommentar |
| `allow-gdpr-anonymize-to-db` | MariaDB | GDPR-jobb → TCP 3306 | Öppnar databasåtkomst för GDPR-jobbet |
| `allow-backend-and-instagram-to-redis` | Redis | Backend och Instagram-refresher → TCP 6379 | Begränsar Redis-klienter |
| `allow-instagram-refresher-egress` | Instagram-refresher | DNS, Redis TCP 6379, publik TCP 443 | Tillåter Redis och extern HTTPS enligt källkommentar |
| `allow-retention-cleanup-egress` | Retention-jobb | DNS UDP/TCP 53; DB TCP 3306 | Begränsar jobbets utgående trafik |
| `allow-retention-cleanup-to-db` | MariaDB | Retention-jobb → TCP 3306 | Öppnar databasåtkomst för retention-jobbet |
| `allow-statistics-egress` | Statistikjobb | DNS, DB TCP 3306, publik TCP 443 | Tillåter databas och extern HTTPS enligt källkommentar |
| `allow-statistics-to-db` | MariaDB | Statistikjobb → TCP 3306 | Öppnar databasåtkomst för statistikjobbet |

## Säkerhetsdesign och status

Konfigurationen modellerar frontend → backend, backend → MariaDB/Redis, särskilda batchjobb → backend eller MariaDB, samt Traefik → frontend/backend. Dessa interna flöden stöds av uttryckliga policyregler i kopiorna. DNS till CoreDNS och offentliga anslutningar avses av reglerna, men faktisk trafik och nätverkspluginens hantering är inte testade.

Egressregler med `0.0.0.0/0` och undantag för vissa privata/särskilda nät begränsar inte destinationer till specifika externa tjänster. Portar 443, 587 och 465 är därför möjliga mot ett brett urval offentliga IP-adresser enligt policyuttrycket. Kluster-, service- och NAT-beteende behöver kontrolleras i den faktiska CNI-miljön.

PodSelectors utan namespaceSelector avser pods i samma namespace som policyn. DNS- och Traefikreglerna använder både namespaceSelector och podSelector; båda måste matcha. De granskade filerna saknar namespacefält, men Kustomize-overlayen anger namespace `nagelsalong`.

## Resultat och lärdomar

- Implementerat i manifest: ingress- och egressisolering för alla pods genom default deny, med selektiva undantag för angivna arbetslaster, portar och protokoll.
- Granskat: 15 NetworkPolicy-resurser i 11 resursfiler, plus Kustomize-referenser.
- Kräver verifiering: om CNI verkställer policies, att pod-etiketter och faktiska portar stämmer, DNS/service-routing, extern egress och om alla nödvändiga arbetslaster täcks.
- Rekommendationer och verifieringsplan finns i [säkerhetsanalysen](security-analysis.md) och [testresultaten](test-results.md).

## Innehåll

- [`networkpolicies/`](networkpolicies/): kommenterade, oförändrade policykopior.
- [`diagrams/network-policy-architecture.md`](diagrams/network-policy-architecture.md): Mermaid-flödesdiagram.
- [`security-analysis.md`](security-analysis.md): fynd, risker och rekommendationer.
- [`test-results.md`](test-results.md): genomförda statiska kontroller och ej utförda klustertester.
