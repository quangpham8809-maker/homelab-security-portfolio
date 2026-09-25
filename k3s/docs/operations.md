# Drift och validering

## Förutsättningar

- `kubectl` med inbyggt Kustomize-stöd;
- valfritt YAML-lintverktyg;
- ett etablerat secret-scanning-verktyg före publicering;
- Ansible för att syntaxkontrollera automationsprovet.

Kommandona nedan renderar endast lokala filer och kontaktar inget kluster:

```bash
kubectl kustomize kubernetes/overlays/dev >/tmp/homelab-dev.yaml
kubectl kustomize kubernetes/overlays/prod >/tmp/homelab-prod.yaml
kubectl kustomize network-policies >/tmp/homelab-network-policies.yaml
```

En serverside dry-run kräver kontakt med ett avsett testkluster och ingår därför inte som standard:

```bash
kubectl apply --dry-run=server -f /tmp/homelab-dev.yaml
```

## Före förändring

1. Läs diffen och identifiera påverkan på nät, lagring, behörighet och tillgänglighet.
2. Kontrollera att inga riktiga identifiers eller hemligheter finns i ändringen.
3. Rendera berörda overlays och granska den sammanslagna YAML-filen.
4. Dokumentera rollback: normalt Git-revert, men datamigreringar kräver en separat plan.
5. Kontrollera aktuell backup och restore-status för stateful förändringar.

## Efter förändring

Verifiera Argo CD sync/health, rollout-status, applikationsprober, felkvot, latens och nya varningar. Bekräfta också önskade och oönskade nätverksflöden med testpods i en icke-produktionsmiljö.

## Ansible-exemplet

Inventoryn innehåller enbart dokumentationsadresser. Ange verklig inventory och hemligheter utanför repositoryt. Kör alltid check mode där modulerna stödjer det och uppdatera en testnod före bredare utrullning.

```bash
ansible-playbook -i ansible/inventory/hosts.example.yml \
  ansible/playbooks/rolling-k3s-update.yml \
  --syntax-check
```
