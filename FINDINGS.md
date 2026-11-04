# Fyndregister

Detta register innehåller endast fynd som har stöd i publicerad evidens. Preliminära uppgifter märks tydligt.

## Statusdefinitioner
- **Obekräftat** – uppgift från tidigare sammanfattning, saknar ännu granskningsbar evidens här.
- **Indikerat** – statiskt stöd finns, men påverkan eller nåbarhet är inte fastställd.
- **Bekräftat statiskt** – relevant kod/konfiguration och analyskedja är dokumenterad.
- **Verifierat på enheten** – reproducerat på fysisk Sony-enhet med säker, icke-destruktiv metod.
- **Avfärdat** – evidensen visar att hypotesen inte håller.

## Fynd

### F-001 – Mali-enhetsnodens åtkomst
- Status: Obekräftat
- Hypotes: vanlig appkontext kan ha åtkomst till Mali-enhetsnod, tidigare sammanfattning nämnde `/dev/mali0` och DAC-läge `0666`.
- Krävd evidens: faktisk firmwarefil/sökväg, nodens mode/ägare/grupp, SELinux-regler och relevant init-/uevent-konfiguration.
- Kommentar: åtkomst till en nod bevisar inte i sig en exploaterbar sårbarhet.

### F-002 – ION-enheter och minnesgränssnitt
- Status: Obekräftat
- Hypotes: ION-relaterade enhetsnoder eller policyregler kan exponera minneshanteringsgränssnitt.
- Krävd evidens: noddefinitioner, DAC/SELinux-policy, klienter och statiskt belagd data-/kontrollväg.

### F-003 – Möjlig Mali-koppling till CVE-2023-4211
- Status: Obekräftat
- Hypotes: tidigare analys noterade en möjlig versions-/kodmatchning.
- Krävd evidens: exakt berörd komponent och version, relevant sårbar kodväg, patch-/fixjämförelse och bevis att komponenten ingår i aktuell firmware.
- Varning: versionslikhet eller GPU-drivrutinsfamilj räcker inte för att bekräfta CVE.

### F-004 – Sony WebAPI / broadcastLink
- Status: Obekräftat
- Hypotes: tidigare sammanfattning nämnde en WebAPI-deklaration med namnet `broadcastLink`.
- Krävd evidens: manifest/API-definition, behörighetskrav, anropsväg, mottagare och faktisk åtkomstmodell.

## Lägg till nya fynd
Varje fynd ska ha unikt ID, status, berörda filer/symboler, evidens, påverkan, begränsningar, alternativa förklaringar och rekommenderad säker verifiering.
