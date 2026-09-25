Backup och restore
Mål
Ett exempel på servicenivå för demoapplikationen:
RPO: högst 24 timmars dataförlust.
RTO: tjänsten ska kunna återställas inom 4 timmar.
Värdena är exempel. Ett verkligt system behöver mål som bestäms av verksamhetens konsekvensanalys.
Backupkedja
Ta schemalagda K3s datastore-snapshots och förvara kopior utanför control-plane-noderna.
Säkerhetskopiera Longhorn-volymer till ett separat NAS-mål.
Använd retention med flera tidshorisonter och skydda äldre recovery points mot ändring.
Övervaka jobbstatus, ålder på senaste lyckade backup och kapacitet på målet.
Dokumentera versioner, checksumma, kryptering och ansvarig operatör.
Restore-övning
En kvartalsvis övning bör ske i ett isolerat nät:
välj en recovery point enligt dokumenterad retention;
återställ datastore eller volym till en separat testmiljö;
validera objekt, applikationsstart och dataintegritet;
mät faktisk RPO och RTO;
kontrollera att inga notifieringar eller integrationer når produktion;
dokumentera avvikelser och skapa uppföljningsåtgärder;
radera testmiljön enligt dess dataklassning efter godkänt resultat.
En lyckad backupstatus utan en verifierad restore är inte tillräcklig evidens.
