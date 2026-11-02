
# Testresultat och verifiering

## Genomfört

- **Statisk inventering:** 12 policy-YAML-filer, 15 NetworkPolicy-resurser identifierade.
- **Kustomize-referenser:** Kontrollerat att networkpolicies-katalogen inkluderas av basen och produktionsoverlayen; overlayen sätter namespace `nagelsalong`.
- **Konfigurationsgranskning:** Granskat selectors, ingress/egress, protokoll, portar, additiva regler, DNS och externa CIDR-regler.
- **Portföljkopior:** YAML-filerna har endast fått svenska kommentarhuvuden; resursernas funktionella YAML-innehåll har bevarats.
- **Klusterkommunikation:** Ej testat.

## Ej testat

- YAML/Kubernetes-schema-validering med `kubectl kustomize` eller API-server.
- Kustomize-rendering av full produktionsoverlay.
- Att aktivt CNI verkställer NetworkPolicies.
- Att podetiketter, DNS-labels, namespace och portar matchar körande workload.
- Faktisk anslutning frontend → backend, backend → DB/Redis, jobb → mål, Traefik → tjänster.
- Extern HTTPS/SMTP och destinationskontroll.
- Service-CIDR, NAT och node-local DNS-effekter.

## Säker verifieringsplan

Kör framtida tester i staging eller annan isolerad miljö med samma CNI och representativa etiketter. Använd endast read-only-inspektion av produktionsklustret om det uttryckligen behövs; starta inga testpods och ändra inga resurser. Validera renderad konfiguration och jämför mot faktiska labels, kontrollera policy enforcement, och testa tillåtna respektive nekade flöden i isolerad testmiljö. Registrera datum, miljö, kommando och faktisk utdata innan resultat märks som verifierade.
