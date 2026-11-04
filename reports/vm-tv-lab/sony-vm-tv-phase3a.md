# Sony VM-TV – Fas 3A: återställ säker testanslutning

Datum: 2026-11-04\
Omfattning: **endast passiv kontroll av Windows-processer, sockets och projektfiler. Inga ADB-kommandon skickades.**

## Sammanfattning

Vid denna kontroll kör AVD-processen `SonyVM_API24_TV_x86`; QEMU har TCP-listeners på loopback för emulatorport 5554/5555 och projektets SDK-ADB-server lyssnar på `127.0.0.1:5038`. Alltså saknas inte 5038-listenern i det observerade nuläget.

Den föregående Fas 3-kontrollen rapporterade att port 5038 saknade listener eftersom `Get-NetTCPConnection` inte returnerade någon rad. I Fas 3A returnerar samma PowerShell-cmdlet fortfarande ingen rad, medan `netstat -ano` och processlistan tydligt visar listenern med PID 36728. Orsaken till skillnaden mellan verktygen kan inte fastställas med passiv inventering. Den tidigare slutsatsen att servern saknades var därför inte tillförlitlig och bör ersättas av denna observation.

ADB-servern och emulatorn är redan igång; inget behöver startas. Jag skickade inga kommandon till någon Android-enhet, startade/stoppade/återanslöt inte ADB eller emulatorn och ändrade inga brandväggsregler.

## 1. Aktuella processer

| PID | Process / sökväg | Kommandorad / roll |
|---:|---|---|
| 23984 | `emulator.exe` — `work/vm-tv/sdk/emulator/emulator.exe` | `-avd SonyVM_API24_TV_x86 ... -ports 5554,5555 -no-window`; identifierar AVD-namnet i emulatorns argument |
| 22912 | `qemu-system-i386-headless.exe` — projektets Android Emulator QEMU | Barnprocess till emulatorn; VM-processen som äger emulatorns TCP-portar |
| 36728 | `adb.exe` — `work/vm-tv/sdk/platform-tools/adb.exe` | `adb -L tcp:5038 fork-server server ...`; projektlokal ADB-server |
| 11344 | `adb.exe` — `C:\platform-tools\adb.exe` | `adb -L tcp:5037 fork-server server ...`; separat global ADB-server, inte VM-TV-servern och inte använd av denna analys |

Projektets ADB-serverprocess har parent PID 13408 enligt processinformationen, men den parentprocessen fanns inte i processlistan vid uppslagningen. ADB-servern kan leva vidare som frikopplad/forkad serverprocess; den observerade processen och dess portbinding bekräftar att den nu körs.

## 2. Aktiva sockets och portar

Uppgifterna nedan kommer från `netstat -ano` och PID-matchning i processlistan. Ingen anslutning gjordes för att testa portarna.

| Protokoll | Lokal bindning | Status | PID / tolkning |
|---|---|---|---|
| TCP | `127.0.0.1:5038` | LISTENING | 36728, projektets SDK ADB-server; endast IPv4 loopback listener observerad |
| TCP | `127.0.0.1:5554`, `[::1]:5554` | LISTENING | 22912, emulatorns console/control-port; loopback |
| TCP | `127.0.0.1:5555`, `[::1]:5555` | LISTENING | 22912, emulatorns ADB-port; loopback |
| TCP | `127.0.0.1:51451`, `[::1]:51451` | LISTENING | 22912, ytterligare emulatorintern listener; exakt funktion inte fastställd |
| TCP | `127.0.0.1:51453` ↔ `127.0.0.1:5555` | ESTABLISHED | 36728 ↔ 22912; projektserverns loopback TCP-session till emulatorporten |
| UDP | `0.0.0.0:5353` | bunden socket | 36728, projektets ADB-process. Troligen ADB mDNS/discovery; inte aktivt testad. Wildcard-bindning innebär att den inte är begränsad till loopback på socketnivå. |
| UDP | `0.0.0.0:50279`, `53344`, `55342`, `58579` | bunden socket | 22912, QEMU; ändamål ej fastställt och inte TCP-listeners |

Viktig skillnad: projektets ADB **TCP**-serverport 5038 och emulatorns TCP-portar var loopback-bundna. ADB-processen hade även UDP-sockets på wildcard-adressen, däribland port 5353. Om Windows inbound-brandvägg tillåter sådan trafik kontrollerades inte i denna fas. Detta är en möjlig exponeringsfråga att granska före nätverksanslutna tester, inte en bekräftad sårbarhet.

Netstat visade också en socket från den separata globala ADB-processen PID 11344 till den tidigare dokumenterade fysiska TV:ns ADB-endpoint. Detta noterades enbart passivt för att hålla VM-servern åtskild; processen kontaktades inte och inga ADB-kommandon skickades genom den. Den globala servern får inte användas i nästa steg.

## 3. ADB-konfiguration och projektinstruktioner

- Projektets avsedda körbara fil är `work/vm-tv/sdk/platform-tools/adb.exe` (SDK Platform-tools 37.0.1 enligt lokal SDK-metadata).
- Aktiv process lyssnar med `-L tcp:5038`; aktuell bindning verifierad i `netstat` som `127.0.0.1:5038`.
- `sony-vm-tv-lab/Start-VM-TV.ps1` sätter `ANDROID_ADB_SERVER_PORT=5038` och anropar projektets ADB med `-P 5038 start-server` innan emulatorn startas. Skriptet använder emulatorportarna 5554/5555 och accepterar endast den förväntade emulatorn i sin verifiering.
- `sony-vm-tv-lab/README.md` anger port 5038 för projektets lokala server och instruerar att inte använda global ADB på 5037.
- `Stop-VM-TV.ps1` innehåller `kill-server` för port 5038 och får inte köras för detta uppdrag.
- Sökning i laboratoriets dokumentations- och konfigurationsfiler visade port 5038 som den avsedda VM-TV-porten och 5037 som global ADB-port. Inget alternativt projektportnummer identifierades.
- Startskriptet hade inte behövts köras för att återskapa listenern: den aktiva processlistan och sockettabellen visar redan projektservern. Eftersom emulatorn också körs skulle startskriptet dessutom avvisa en ny start när det ser den befintliga emulatorprocessen.

## 4. Krävs kontakt med fysisk TV?

Nej. Process-, socket- och filinventeringen gjordes helt på värddatorn. Ingen `adb devices`, `adb connect`, `adb shell`, `adb start-server`, `adb kill-server` eller enhetsåtgärd utfördes. Den fysiska TV:n användes inte som testmål. Den redan etablerade socketen från den separata globala ADB-processen lämnades orörd.

## 5. Varför rapporterades 5038 saknad?

**Bekräftat nu:** PID 36728 kör projektets ADB från projektets SDK och lyssnar på `127.0.0.1:5038` enligt `netstat -ano`.

**Bekräftat om kontrollverktyget:** `Get-NetTCPConnection -State Listen -LocalPort 5038` returnerade ingen rad i samma kontrollperiod, trots att `netstat` visade listenern. Den tidigare flerportsfrågan hade samma typ av tomt resultat. Därför var det inte säkert att dra slutsatsen att porten saknade listener från cmdlet-resultatet ensamt.

**Inte fastställt:** Varför PowerShell-cmdleten utelämnar socketen i detta värdläge, och om den tidigare fasens tomma resultat var samma cmdlet-/providerproblem eller en kortvarig skillnad. Ingen ändring av processer eller portar gjordes för att undersöka det vidare.

## 6. Exakt rekommenderat nästa steg

ADB-servern behöver inte startas. Nästa steg, **först efter godkännande**, är enbart att läsa target-listan från den redan körande projektservern med dess fullständiga sökväg och explicit port:

```powershell
& '.\work\vm-tv\sdk\platform-tools\adb.exe' -P 5038 devices -l
```

Det kommandot ska köras från projektroten. Fortsätt endast om listan visar exakt den förväntade emulatorn (`emulator-5554`) och ingen fysisk eller okänd enhet. Om listan avviker: avbryt utan shell-kommandon. Använd inte `C:\platform-tools\adb.exe`, port 5037 eller `adb connect`. Inga Android-shelltester har godkänts eller utförts i Fas 3A.

**Nuvarande status:** VM och projektserver är redan igång; inget startas nu. Jag inväntar godkännande innan nästa ADB-kommando eller säkerhetstest.
