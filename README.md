# Sony Security Research

Arbetsyta för dokumentation och statisk säkerhetsanalys av Sony KD-55XD9305 / firmware SVP-DTV15, release 5.433 EUB.

## Syfte
Samla fasrapporter, verifierade fynd, evidens och överlämning till ChatGPT på ett ställe så att Codex kan publicera uppdateringar och ChatGPT kan granska dem direkt via GitHub.

## Viktiga regler
- Arbeta endast med firmwarematerial som användaren har rätt att analysera.
- Ingen exploatering, fuzzning, destruktiv testning eller ändring av TV:ns firmware utan separat uttryckligt beslut.
- Skilj alltid mellan statiska observationer, emulatorresultat och verifierade resultat på fysisk Sony-hårdvara.
- Bekräfta inte CVE-träffar enbart utifrån versionsnummer eller liknande kod. Kräv relevant kod-/konfigurationsbevis och dokumentera osäkerhet.
- Lägg aldrig upp firmwarebilder, proprietära binärer, privata nycklar, tokens, lösenord eller andra känsliga artefakter i detta repo.
- Publicera endast rapporter, metadata, hashvärden, reproducerbara analysnoteringar och ofarliga skript.

## Arbetsflöde
1. Codex analyserar lokalt och sparar en fasrapport under `reports/`.
2. Codex uppdaterar `PROJECT-STATUS.md`, `FINDINGS.md` och `CHATGPT-HANDOFF.md`.
3. Codex granskar `git diff`, kontrollerar att inga binärer/hemligheter följer med, committar och pushar till `main`.
4. ChatGPT läser överlämningen och relevanta rapporter direkt från repot.

Se `CODEX-INSTRUCTIONS.md` för publiceringsrutinen.
