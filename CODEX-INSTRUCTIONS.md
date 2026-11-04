# Instruktioner för Codex

Detta repository är projektets gemensamma dokumentationsyta.

## Efter varje fas
1. Spara den fullständiga textbaserade rapporten under `reports/`, exempelvis `reports/Fas-3D-Deep-Interface-Security-Analysis.md`.
2. Uppdatera:
   - `PROJECT-STATUS.md`
   - `FINDINGS.md`
   - `CHATGPT-HANDOFF.md`
3. Lägg till evidens som gör varje slutsats granskningsbar: firmwarefil/sökväg, radnummer, offset, symbol, relevant konfigurationsrad eller kommandoresultat.
4. Skilj på observation, tolkning och slutsats. Ange alternativa förklaringar och konfidens.
5. Håll emulatorresultat, statisk firmwareanalys och fysisk enhetsverifiering separerade.
6. Kontrollera ändringarna innan publicering:
   - `git status --short`
   - `git diff --check`
   - `git diff --stat`
   - granska `git diff`
7. Publicera inte firmwarebinärer, kompletta firmwarepaket, APK:er, diskavbilder, privata nycklar, lösenord, tokens eller personuppgifter.
8. Committa och pusha endast textbaserade rapporter, metadata, hashvärden och säkra analysnoteringar.

## Publiceringsregel
Efter att lokal git-anslutning är konfigurerad ska Codex, när en fasrapport är färdig och kontrollerad, skapa en commit och pusha till `origin/main`. Om push misslyckas: rapportera felet och ändra inte eller skriv över fjärrhistorik.

## Säkerhetsgräns
Fortsätt med statisk och passiv analys enligt projektets omfattning. Kör inte exploitkod, fuzzning, destruktiva tester eller okända ioctl-anrop. Ändra inte TV:ns firmware.
