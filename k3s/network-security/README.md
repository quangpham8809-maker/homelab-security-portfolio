# Kubernetes Network Security – Rose Nails

## Projektöversikt

Det här arbetsprovet beskriver en statisk granskning av Kubernetes NetworkPolicies för applikationen Rosa Nails. NetworkPolicies kan begränsa vilka pods som får kommunicera med varandra och med externa nätverk. De hjälper till att minska oavsiktlig åtkomst och lateral rörelse, förutsatt att klustrets nätverksplugin verkställer reglerna.

Dokumentationen bygger på en granskning av produktionskonfigurationen, men portföljens YAML-exempel är fristående dokumentationskopior. De är inte produktionsfiler och ska inte användas som GitOps-källa. Endast ett urval av YAML-exemplen har publicerats manuellt; denna text påstår inte att alla granskade policyfiler finns i det publika repot.

## Syfte

Målet är att visa hur nätverkssegmentering, Default Deny och Least Privilege kan användas för att begränsa trafik mellan frontend, backend, databas, Redis och schemalagda jobb. Arbetet omfattar inventering, analys av policyreglernas samverkan och dokumentation av verifieringsbehov.

## Teknisk miljö

Den granskade konfigurationen använder K3s/Kubernetes, Argo CD/GitOps, Traefik, Nginx-frontend, FastAPI-backend, MariaDB, Redis och CronJobs. Produktionsoverlayen tilldelar applikationsresurserna namespace `nagelsalong`. Policymanifesten i basen saknar eget namespacefält; Kustomize sätter namespace genom overlayen.

Detta är en statisk granskning av lokala manifests. Körande pods, faktisk nätverkstrafik och aktiv CNI-verkställighet har inte verifierats.

## Säkerhetsprinciper

- **Default Deny:** policyn `default-deny` väljer alla pods i sitt namespace och isolerar dem för både ingress och egress. Den policyn tillåter i sig ingen trafik. Andra policies kan lägga till tillåtelser.
- **Least Privilege:** tillåt endast de källor, mål, portar och protokoll som arbetslasten behöver. Portbegränsning är användbar, men breda destinationsregler kan fortfarande ge större åtkomst än avsett.
- **Network Segmentation:** frontend, backend, databas, Redis och jobb har separata, deklarerade kommunikationsvägar.
- **Ingress och Egress:** ingress avser trafik in till en vald pod; egress avser trafik ut från en vald pod. När båda kommunicerande pods är isolerade behöver källans egress och destinationens ingress tillåta anslutningen.
- **Zero Trust:** varje anslutning ska ha ett motiverat syfte och begränsas. NetworkPolicies är ett nätverkskontrollager och ger inte ensamma fullständig Zero Trust-säkerhet.
- **Defense in Depth:** nätverkspolicy kompletterar autentisering, applikationsbehörigheter, hemlighetshantering, TLS och andra skydd.

NetworkPolicies är additiva: om någon policy tillåter en anslutning i en riktning så tas den tillåtelsen inte bort av en annan policy. Default Deny ger isolering; separata allow-regler öppnar undantag. En policykonfiguration visar deklarerad avsikt, inte att CNI faktiskt verkställer den eller att trafiken har testats.

## Policyöversikt från den granskade konfigurationen

Tabellen inventerar resurser i den granskade lokala konfigurationen. Den är **inte** en lista över YAML-filer som alla har publicerats i portföljrepot.

| NetworkPolicy | Skyddar | Deklarerad trafik | Säkerhetssyfte |
|---|---|---|---|
| `default-deny` | Alla pods i tilldelat namespace | Isolerar ingress och egress; inga tillåtelser i policyn | Grundisolering |
| `allow-traefik-to-frontend` | Frontend | Traefik från `kube-system` → TCP 80 | Begränsar inkommande trafik |
| `allow-frontend-egress` | Frontend | CoreDNS UDP/TCP 53; backend TCP 8000 | DNS och API-trafik |
| `allow-traefik-frontend-and-cronjobs-to-backend` | Backend | Traefik, frontend, watch-renewer och send-reminders → TCP 8000 | Begränsar API-klienter |
| `allow-backend-egress` | Backend | DNS UDP/TCP 53; MariaDB TCP 3306; Redis TCP 6379; publika mål TCP 443/587/465 | Interna beroenden och extern egress |
| `allow-cronjobs-egress` | watch-renewer och send-reminders | DNS UDP/TCP 53; backend TCP 8000 | Begränsar jobbens utgående trafik |
| `allow-backend-to-db` | MariaDB | Backend → TCP 3306 | Databasåtkomst för backend |
| `allow-gdpr-anonymize-egress` | GDPR-jobb | DNS UDP/TCP 53; MariaDB TCP 3306; publika mål TCP 443 | Jobbets utgående beroenden |
| `allow-gdpr-anonymize-to-db` | MariaDB | GDPR-jobb → TCP 3306 | Databasåtkomst för GDPR-jobbet |
| `allow-backend-and-instagram-to-redis` | Redis | Backend och Instagram-refresher → TCP 6379 | Begränsar Redis-klienter |
| `allow-instagram-refresher-egress` | Instagram-refresher | DNS UDP/TCP 53; Redis TCP 6379; publika mål TCP 443 | Redis och extern egress |
| `allow-retention-cleanup-egress` | Retention-jobb | DNS UDP/TCP 53; MariaDB TCP 3306 | Jobbets utgående beroenden |
| `allow-retention-cleanup-to-db` | MariaDB | Retention-jobb → TCP 3306 | Databasåtkomst för retention-jobbet |
| `allow-statistics-egress` | Statistikjobb | DNS UDP/TCP 53; MariaDB TCP 3306; publika mål TCP 443 | Databas och extern egress |
| `allow-statistics-to-db` | MariaDB | Statistikjobb → TCP 3306 | Databasåtkomst för statistikjobbet |

Inventeringen omfattar **15 NetworkPolicy-resurser i 12 YAML-filer som innehåller policyresurser**. Därutöver finns en separat `kustomization.yaml` som listar resurserna; den är inte en NetworkPolicy. Tre policyfiler innehåller två resurser vardera: GDPR-, statistik- och retention-filerna.

## Säkerhetsdesign och fynd

Deklarerade interna flöden omfattar Traefik → frontend på TCP 80, Traefik/frontend/utvalda jobb → backend på TCP 8000, backend → MariaDB på TCP 3306 och backend → Redis på TCP 6379. GDPR-, statistik- och retention-jobben har också deklarerade flöden till MariaDB på TCP 3306. För anslutningar mellan isolerade pods behöver både källans egressregel och destinationens ingressregel tillåta trafiken.

**Medium – bred publik egress:** backend, GDPR-jobb, Instagram-refresher och statistikjobb har regler med `0.0.0.0/0`, undantag för listade nät och portbegränsningar. Portbegränsningarna hindrar inte i sig anslutning till oönskade publika destinationer. Granska i första hand dessa fyra egresspolicies och verifiera vilka externa beroenden som faktiskt behövs.

**Low – databas- och Redis-åtkomst:** backend har nätverksåtkomst till MariaDB och Redis. GDPR-, statistik- och retention-jobben har nätverksåtkomst till MariaDB. Workload-manifesten visar databasberoenden för samtliga DB-klienter. Om en tillåten klient komprometteras kan den ändå nå tjänsten på tillåten port; NetworkPolicy ersätter inte databasautentisering och behörighetskontroll.

**Low, villkorat – pod-etiketter och RBAC:** nätverksrättigheter följer pod-etiketter. Om en aktör har för generös RBAC och kan skapa pods i namespace med etiketter som matchar policies kan den potentiellt få motsvarande nätverksväg. RBAC och faktisk möjlighet att skapa sådana pods ingick inte i den statiska granskningen.

## Verifieringsstatus

Tidigare genomförd lokal Kustomize-rendering och funktionell jämförelse har rapporterats för policykatalogen och dokumentationskopiorna. Det är en kontroll av manifestinnehåll, inte ett klustertest eller bevis på API-serverns schemaacceptans.

**Ej testat:** faktisk pod-kommunikation, att CNI verkställer policies, fullständig rendering av produktionsoverlayen, DNS/service-routing i klustret samt extern egress. Se [testresultat](test-results.md) och [säkerhetsanalys](security-analysis.md).

## Material

- [`networkpolicies/`](networkpolicies/): urval av kommenterade YAML-exempel; publicerat urval kan skilja sig från den lokala granskningskopian.
- [`diagrams/network-policy-architecture.md`](diagrams/network-policy-architecture.md): tre diagram över deklarerade flöden, jobb och segmentering.
- [`security-analysis.md`](security-analysis.md): risker, status och rekommendationer.
- [`test-results.md`](test-results.md): tidigare rapporterade statiska kontroller och ej testade delar.
