# Ansible-exempel

Inventoryn använder dokumentationsadresser i `10.10.30.0/24` och en generisk användare. Den innehåller inga nycklar eller lösenord.

Playbooken uppdaterar en nod i taget och kräver att version samt SHA-256 anges vid körning. Hämta checksumman via en separat, betrodd kanal och jämför den med leverantörens signerade releaseinformation.

```bash
cd ansible
ansible-playbook playbooks/rolling-k3s-update.yml \
  --syntax-check
```

En faktisk körning ska använda en separat, privat inventory och extra variabler från CI/Semaphore eller en godkänd secrets manager. Testa först i en icke-produktionsmiljö.
