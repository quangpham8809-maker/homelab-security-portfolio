Säkerhetsmodell
Skyddsvärden
administrativa identiteter och åtkomstvägar;
Kubernetes control plane och GitOps-identitet;
applikationsdata och backupkopior;
konfigurationsintegritet i Git;
loggar och mätvärden som behövs vid incidentanalys.
Förenklad hotmodell
Hot	Exempel	Huvudsakliga kontroller
Exponerad administration	managementgränssnitt nås från internet	Tailscale, management-VLAN, brandväggsregler, MFA där det stöds
Lateral rörelse	komprometterad frontend når databasen direkt	default-deny, explicit frontend→backend, inga onödiga Services
Supply-chain-angrepp	manipulerad image eller dependency	pinnade versioner, image scanning, kontrollerad registry, granskad ändring
Credential-läckage	hemlighet hamnar i Git eller logg	separerad secrets-hantering, scanning, kortlivade credentials, rotation
Privilege escalation	sårbar container får nodbehörighet	icke-root, seccomp, drop capabilities, ingen privilege escalation, RBAC
GitOps-missbruk	controller får för bred klusteråtkomst	avgränsat AppProject, minsta RBAC, branch protection, audit logs
Dataförlust	radering, ransomware eller korruption	isolerade backuper, retention, integritetskontroll, restore-övningar
Antaganden och kvarvarande risk
NetworkPolicy skyddar bara om vald CNI verkställer policyn. Namespace-labels, DNS-selektorer och ingress-controller-labels måste verifieras mot målklustret. Exemplens container-images är platshållare och ska ersättas med betrodda, skannade och helst digest-pinnade images.
Administratörskonton, hypervisor och brandvägg ligger utanför Kubernetes skydd. De behöver separat hardening, patchning, loggning och återställningsplan. Inget enskilt lager antas vara ofelbart.
Publiceringsgrind
Före offentlig publicering:
granska samtliga nya och ändrade filer;
sök efter nyckelmaterial, tokens, interna domäner och verkliga IP-adresser;
kör ett specialiserat secret-scanning-verktyg över arbetskatalog och historik;
rendera alla Kustomize-overlays;
verifiera att exempelbilder, loggar och terminalutskrifter är sanerade;
rotera omedelbart varje credential som kan ha exponerats, även om filen tas bort.
