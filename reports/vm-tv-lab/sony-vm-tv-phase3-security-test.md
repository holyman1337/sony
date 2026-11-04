# Sony VM-TV – Fas 3: praktiskt säkerhetstest

Datum: 2026-11-04\
Status: **Avbrutet vid obligatorisk målverifiering. Inga säkerhetstester kördes.**

## Sammanfattning

Testningen avbröts innan något kommando skickades till en Android-enhet. Den projektlokala ADB-servern som används av VM-TV ska redan vara igång på `127.0.0.1:5038`; vid förkontrollen fanns ingen lyssnare på port 5038. Därför kunde AVD-serial och gästens fingerprint inte verifieras i denna körning.

Jag startade inte ADB-servern, körde inte `adb devices`, försökte inte återansluta en emulator och använde inte den globala ADB-servern på port 5037. Jag använde inte TV:ns tidigare adress `<private-TV-IP>:5555`, skickade inga kommandon till TV:n och ändrade inte ADB-processer, anslutningar, AVD, snapshots eller brandväggsregler.

**Slutsats:** Det här testet har inte visat ett faktiskt säkerhetsproblem, men det har inte heller verifierat att emulatorn är korrekt konfigurerad. Resultatet är *ej testat*, inte ett godkänt säkerhetsresultat.

## 1. Testmål och verifieringsgrind

| Kontroll | Metod | Resultat | Klassificering |
|---|---|---|---|
| Projektlokal ADB-server redan igång | Läste TCP-listeners för 5038, 5554 och 5555 med `Get-NetTCPConnection -State Listen -LocalPort 5038,5554,5555` | Ingen listener returnerades; kontrollen avbröt innan ytterligare åtgärd | Bekräftad observation för denna kontrollkörning |
| ADB-serverprocess/ägande PID | Skulle läsas först efter funnen 5038-listener | Ej kontrollerad eftersom ingen listener hittades | Ej testad |
| AVD-process för `SonyVM_API24_TV_x86` | Skulle verifieras före ADB-anrop | Ej kontrollerad efter att verifieringsgrinden misslyckades | Ej testad |
| ADB-target serial | `adb -P 5038 devices -l` kördes inte | Ingen serial kunde verifieras | Ej testad |
| Android release/API/fingerprint/ABI | `adb shell getprop` kördes inte | Ingen aktuell runtime-identitet kunde verifieras | Ej testad |
| Fysisk TV som ADB-target | Projektets ADB-lista kunde inte läsas; global ADB/TV-adress användes inte | Ingen TV-kommunikation gjordes. Frånvaro av TV i projektserverns lista kunde inte verifieras eftersom listan inte hämtades | Ej verifierad, men inte kontaktad |

Verifieringsgrinden krävde en redan körande projektlokal server och exakt en verifierad serial `emulator-5554`. Den förutsättningen uppfylldes inte. Därför avbröts testet enligt instruktionen. Port 5038 öppnades inte och någon server startades inte.

Tidigare projektdokumentation anger att AVD:n kör Android 7.0/API 24 med generic x86 fingerprint och att dess ADB/emulatorportar var loopback-bundna. Detta är historiska observationer från en tidigare körning, inte en aktuell identitetsbekräftelse för Fas 3. Se [Fas 2-rapporten](sony-vm-tv-phase2.md) och [laboratorieanvisningarna](README.md).

## 2. Genomförda praktiska tester

Inga tester nedan kördes eftersom målidentiteten inte kunde verifieras.

| Område | Avsedd kontroll | Kommando/metod i denna körning | Faktiskt resultat | Reproducerbarhet | Risk / fyndklass | Koppling till Sony |
|---|---|---|---|---|---|---|
| Android-konfiguration | SELinux, UID, debug properties, filrättigheter | Inget guest-kommando kördes | Ej testat | Ej bedömd | Inget fynd; risk ej bedömd | Ingen runtime-koppling |
| Applikationsisolering | Paket, UID/sandbox, deklarerade permissions | Inget guest-kommando eller app-test kördes | Ej testat | Ej bedömd | Inget fynd; risk ej bedömd | Ingen runtime-koppling |
| Privat appdata | Åtkomst via normala app-gränssnitt | Ingen app installerades eller kördes; inga privata filer lästes | Ej testat | Ej bedömd | Inget fynd; risk ej bedömd | Ingen runtime-koppling |
| Privilegierade tjänster | Binder-tjänster och manifestdeklarerade behörigheter | `service list`, `dumpsys` och PackageManager-kontroller kördes inte | Ej testat | Ej bedömd | Inget fynd; risk ej bedömd | Ingen runtime-koppling |
| Gästnätverk | Lyssnande portar/tjänster och autentisering | Inget kommando kördes i gästen; ingen portprobe gjordes | Ej testat | Ej bedömd | Inget fynd; risk ej bedömd | Ingen runtime-koppling |
| ADB-exponering | ADB-serverns bindning och gästserial | Endast lokal preflight av TCP-listener; ingen listener på 5038 hittades | Målet kunde inte verifieras; inget ADB-test | Observationen gjordes en gång, inte upprepad | Ej en sårbarhet; testförutsättning saknas | Avser VM-TV-projektets port, inte Sony firmware |
| Extern exponering | Host-portbindning och brandväggsscope | Ingen fullständig aktuell bindnings-/regelkontroll genomfördes efter abort | Ej testat i Fas 3 | Ej bedömd | Inget fynd; risk ej bedömd | Ingen runtime-koppling |

## 3. Fynd och riskbedömning

### Bekräftade sårbarheter

Inga. Ingen komponent utsattes för säkerhetstest i denna körning.

### Osäkra konfigurationer

Inga nya konfigurationer verifierades eller identifierades. ADB-målet kunde inte verifieras; detta är ett testhinder, inte i sig bevis på en osäker ADB-konfiguration.

### Teoretiska risker

De tidigare statiska Sony-firmwarefynden, bland annat Mali/CVE-2023-4211-analysen, är oförändrade men testades inte här. De kan inte användas som bevis för en sårbarhet i den generiska x86 AVD:n.

### Ej testade funktioner

Allt som kräver kontakt med Android-gästen är ej testat i Fas 3: SELinux runtime, användare, package UIDs, sandbox, filåtkomst, binder, tjänstbehörigheter, portar, autentisering och ADB-exponering.

## 4. Begränsningar och säkerhetsgränser

- Testet avbröts före all ADB-trafik eftersom port 5038 saknade listener.
- Global ADB-port 5037 och TV-adressen användes inte eller ändrades inte.
- Ingen exploatering, root, SELinux-bypass, kernel-/drivertrigger, Mali ioctl/mmap, fuzzing, race-test eller avsiktlig krasch förekom.
- Ingen app installerades. AVD-data, snapshots, properties och nätverksinställningar ändrades inte.
- Inga slutsatser om Sony-firmware kan dras från detta avbrutna test.

## 5. Nästa steg

För att upprepa Fas 3 måste den befintliga **projektlokala** ADB-servern vara igång på port 5038 och endast visa `emulator-5554` som ansluten enhet. Därefter ska identiteten verifieras igen innan tester börjar. Använd inte den globala ADB-servern på 5037 eller TV:ns adress. Om en fysisk eller okänd serial visas ska testet avbrytas.

När målgrinden passerar kan testerna begränsas till läsande diagnostik: `getenforce`, `id`, `getprop`, `pm list packages -U`, `dumpsys package`, `service list`, `dumpsys`, `/proc/net/tcp*` eller tillgängligt `ss/netstat`, samt hostens loopback-bindningar och redan aktiva brandväggsregler. Appdata ska inte läsas genom bypass; app-till-app-isolering kräver ett ofarligt testprogram och en separat återställningsbar AVD-kopia.
