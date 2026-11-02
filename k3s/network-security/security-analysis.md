
# Säkerhetsanalys

## Omfattning och metod

Granskningen omfattar 12 policy-YAML-filer med 15 NetworkPolicy-resurser samt Kustomize-referenser i applikationens bas och produktionsoverlay. Analysen är statisk. Inga ändringar eller klusterkommandon har utförts. I materialet sätts namespace av produktionsoverlayen till `nagelsalong`; policyfilerna saknar eget namespacefält.

Kubernetes NetworkPolicies är additiva. Egress bedöms på källpodden och ingress på destinationspodden. En ingressregel på DB och en egressregel på klienten behöver båda tillåta samma kommunikation när respektive sida är isolerad.

## Fynd

### Medium – Extern egress tillåter breda publika destinationer (verifierat i manifest)

**Problem:** Backend, GDPR-jobb, Instagram-refresher och statistikjobb har `ipBlock: 0.0.0.0/0` med undantag för en lista privata/särskilda nät. Tillåtna portar är TCP 443, och för backend även 587/465.

**Risk och möjlig exploatering:** En komprometterad pod kan försöka skicka data till angriparkontrollerade publika servrar eller nå andra publika tjänster på de tillåtna portarna. Undantagen begränsar destinationer men ger inte destinationsallowlisting. Egress mot CIDR kan också påverkas av CNI, routing och NAT.

**Rekommendation:** Verifiera de faktiska beroendena. Överväg kontrollerad egressproxy/FQDN-filtering eller hanterade egressregler med dokumenterad destinationslista där plattformen stödjer det. Kontrollera service-CIDR, node/LAN-rutter och NAT-beteende innan slutsats om isolering dras.

**Status:** Konfigurationsbredden är verifierad statiskt; exploaterbar väg och faktisk nätverksräckvidd behöver testas i icke-produktionsmiljö.

### Low – Selektorer och täckning är beroende av etiketter och overlay (behöver verifieras)

**Problem:** Policies matchar exakta `app`-etiketter och förlitar sig på att overlayens namespace-transformering och workload-etiketter är korrekta. PodSelectors utan namespaceSelector gäller bara inom policyns namespace. Traefik- och CoreDNS-regler kräver att både namespace- och podselektorn matchar.

**Risk och möjlig exploatering:** Om podar saknar förväntade etiketter eller använder fel etikett kan nödvändig trafik blockeras. Om en bred delad etikett används på oavsedd workload kan den få regelns nätverksrättigheter. En selector kan inte tillåta trafik som saknas, men annan additiv policy kan ge ytterligare åtkomst.

**Rekommendation:** Jämför renderad Kustomize-konfiguration med faktiska podetiketter och namnrymder. Dokumentera avsiktliga matchningar och kontrollera selectors i staging.

**Status:** Selektoruttrycken är verifierade; faktisk workload-täckning är ej testad.

### Informational – Default deny finns deklarerad; verkställighet är inte verifierad

**Problem:** `default-deny` väljer alla pods och anger både Ingress och Egress. Den inkluderas i nätverkspolicy-Kustomize-resurslistan och basen inkluderas av produktionsoverlayen. Manifestet i sig bevisar inte att aktivt kluster använder ett kompatibelt CNI eller att Argo CD lyckats applicera den.

**Risk:** Om NetworkPolicy inte verkställs saknas det förväntade segmenteringsskyddet. Om den verkställs men undantag saknas kan arbetslaster tappa DNS eller beroenden.

**Rekommendation:** I godkänd staging eller read-only klustergranskning, verifiera aktiv policy, CNI-stöd och Argo CD:s synkstatus; genomför därefter kontrollerade anslutningstester i isolerad testmiljö.

**Status:** Förekomst och Kustomize-referens verifierade statiskt; aktiv enforcement ej testad.

### Informational – Flera separata DB-ingressundantag (verifierat i manifest)

**Problem:** Fyra policies tillåter åtkomst till `app=rosanails-db`: backend, GDPR-jobb, statistikjobb och retention-jobb på TCP 3306. Detta är separata, avsiktliga arbetslaster enligt namngivning/etiketter, inte dubbletter av samma regel.

**Risk:** Varje ytterligare klient utökar mängden pods som kan nå databasen. En komprometterad klient kan nå DB-tjänsten, även om databasskiktets autentisering/behörigheter separat begränsar åtkomst.

**Rekommendation:** Bekräfta att alla fyra jobb behöver DB-åtkomst och att etiketter är unika nog. Behåll applikations- och databasautentisering; NetworkPolicy ersätter inte dessa kontroller.

**Status:** Reglerna är verifierade statiskt; faktisk nödvändighet och DB-behörigheter ej testade.

## Granskning per område

1. **Default-deny-isolering:** Ja, manifestet väljer alla pods och isolerar ingress samt egress. Verklig enforcement ej verifierad.
2. **Ingress:** Frontend tillåts från Traefik på TCP 80. Backend tillåter Traefik, frontend och två jobb på TCP 8000. DB tillåter fyra namngivna klienttyper på TCP 3306. Redis tillåter backend och Instagram-refresher på TCP 6379.
3. **Egress:** Frontend och cronjobs har snäva interna flöden. Backend och vissa jobb har bred publik egress på angivna TCP-portar. Övriga workloads saknar särskild egresspolicy i denna katalog och ska, om default deny gäller, vara egressisolerade.
4. **Pod- och namespace-selectors:** Samma-namespace-villkoret gäller för fristående podSelector. Kombinationer av namespaceSelector och podSelector kräver båda villkoren. Overlayens namespace är `nagelsalong`.
5. **DNS-undantag:** Flera workloadregler tillåter UDP och TCP 53 till pods märkta `k8s-app=kube-dns` i `kube-system`. Matchning av labels, DNS serviceväg och eventuellt node-local DNS behöver verifieras.
6. **Databaskommunikation:** TCP 3306 från backend samt GDPR-, statistik- och retention-jobb tillåts av DB-ingressregler; motsvarande egress finns för dessa klienter.
7. **Redis:** TCP 6379 från backend och Instagram-refresher tillåts. Backend har motsvarande egress; Redis egress behövs inte för dessa inkommande sessioner.
8. **Lateral movement:** Default deny och selektiva selectors minskar lateral rörelse mellan matchade pods. Effekt beror på CNI enforcement och andra policies som kan addera tillåtelse.
9. **Överdrivna regler:** Extern egress på 0.0.0.0/0 är bred i destination. Inga duplicerade resursnamn eller exakta upprepade regler hittades i den inventerade katalogen. DNS-regler upprepas avsiktligt per workload.
10. **Möjliga förbättringar:** Validera renderad manifest och etiketter, klargör externa destinationer, granska cluster/service CIDR/NAT/CNI, och testa policy i staging. Ändra inte produktionsmanifest utan separat granskning och godkännande.

## Begränsningar

Analysen kan inte fastställa vilka pods som kör, om portarna faktiskt lyssnar, om all egress är nödvändig, hur CNI hanterar `ipBlock`, om NodeLocal DNS används eller om det finns andra NetworkPolicies utanför katalogen. Läs [testresultat](test-results.md) för status.
