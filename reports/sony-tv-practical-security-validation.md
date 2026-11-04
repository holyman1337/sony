# Sony KD-55XD9305 – praktisk säkerhetsvalidering

**Datum:** 2026-11-04\
**Omfattning:** Läsande runtime-verifiering och riskbedömning. Ingen GPU-enhet öppnades och inga Mali-ioctl-anrop utfördes.

## Sammanfattning

TV:ns läsbara runtime-identitet stämmer med den lokala 5.433 EUB-firmwarekopian: build fingerprint, Android-version, kernelversion samt SHA256 för `build.prop`, `mali_kbase.ko` och `libGLES_mali.so` matchar. `/dev/mali0` finns med DAC-läge `0666`, ägare/grupp `root:root` och SELinux-etiketten `gpu_device`; SELinux rapporteras som Enforcing.

Den tidigare statiska policyanalysen visar en regel som tillåter `untrusted_app` `open`, `read`, `write` och `ioctl` på `gpu_device:chr_file`. Det ger starkt stöd för att vanliga appdomäner tillåts nå Mali-enheten enligt SELinux. Det bevisar inte genom runtime-test att en appens `open()` eller senare drivrutinkall faktiskt lyckas. Att verifiera det skulle kräva att en app öppnar `/dev/mali0` eller går genom grafikstacken, vilket uttryckligen inte gjordes.

**Rekommendation:** behandla enheten som direkt exponerad för vanliga appar på policynivå och stanna vid verifieringen av statisk policy och artefaktmatchning. För dynamiska felvägar krävs i praktiken ett separat, återställningsbart exemplar med motsvarande SoC och drivrutin. En standardemulator kan testa Android-ramverk och appbeteende, men ersätter inte Mali-T624 och Sony-kärnmodulen.

## 1. Runtime-verifiering

Anslutningen gjordes endast till den angivna ADB-enheten `<private-TV-IP>:5555`. `adb devices -l` rapporterade en auktoriserad `device` med produkt `SVP4KDTV15_EU`, modell `BRAVIA_4K_2015` och enhet `SVP-DTV15`.

| Egenskap | Runtime-resultat | Lokal 5.433-källa / kontroll | Bedömning |
|---|---|---|---|
| Build fingerprint | `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys` | Extraherad system-build metadata | Matchar angiven build `NRD91N.S34/5.433` |
| Android / API | Android `7.0`, SDK `24` | Firmware build metadata | Matchar |
| Incremental / display ID | `5.433`; `SVP4KDTV15_EU-user 7.0 NRD91N.S34 5.433 release-keys` | Firmware build metadata | Matchar release 5.433 |
| Säkerhetspatch | `2019-02-01` | Runtime property | Bekräftat runtime |
| Kernel | `3.10.79`; `/proc/version` anger build 8 mars 2019 | Firmware/kernelmetadata | Versionen matchar; detta identifierar inte ensamt exakt kernelkonfiguration |
| SELinux | `Enforcing` | `getenforce` | Bekräftat runtime |
| ADB-shell | UID 2000 (`shell`), SELinux `u:r:shell:s0` | `id`, `/proc/self/attr/current` | Bekräftat runtime |
| `/dev/mali0` | `crw-rw-rw-`, `root:root`, `u:object_r:gpu_device:s0`, major/minor `10,49` | Firmware `ueventd`-konfiguration och statisk policy | Bekräftat runtime; DAC 0666 |
| `/dev/mali` | Saknas | `ls -lZ` | Inte en separat befintlig nod under det sökta namnet |
| Laddad modul | `/proc/modules` innehåller `mali_kbase`, storlek `296429`, användarreferenser `15` | Lokal `mali_kbase.ko` | Modulnamnet bekräftat; modulens procfs storlek är inte filstorlek |
| Driver-version via sysfs | Läsning av `/sys/module/mali_kbase/version` nekades | Ingen privilegierad åtkomst försöktes | Ej verifierbar via shell |

ADB-relaterade properties: `ro.secure=1`, `ro.adb.secure=1`, `ro.debuggable=0`; `adbd` kör och `service.adb.tcp.port=5555`. Detta visar att ADB-auktorisering är konfigurerad och att denna värd redan är accepterad. Det bevisar inte kryptering av ADB-trafik eller nätverkssegmentets filtrering. Någon nätverksskanning utfördes inte.

### Lokala hashjämförelser

Filerna hämtades via vanlig, läsande `adb pull` från TV till `work/runtime-snapshot-2026-11-04/`. Hasharna beräknades lokalt. Enheten saknar `sha256sum`; ingen privilegierad metod användes.

| Fil | Runtime-snapshot SHA256 | Lokal 5.433 SHA256 | Resultat |
|---|---|---|---|
| `/system/build.prop` | `f47be96685979a122f157d0296c45276b8f589a8be9ed5075127004b27b4a54b` | `f47be96685979a122f157d0296c45276b8f589a8be9ed5075127004b27b4a54b` | Identisk |
| `/system/lib/modules/mali_kbase.ko` | `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c` | `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c` | Identisk* |
| `/system/lib/egl/libGLES_mali.so` | `a6f463e9cb34591c5818e25437ca346130f113e2f85e5b5e0f90b3e96adcf835` | `a6f463e9cb34591c5818e25437ca346130f113e2f85e5b5e0f90b3e96adcf835` | Identisk |


Den fullständiga runtime- och lokala artefaktlikheten stöder att just dessa tre filer överensstämmer byte för byte. Den bevisar inte att varje partition på TV är identisk med arkivet eller att ingen annan runtime-komponent har ändrats.

## 2. Bekräftade fakta, slutsatser och okänt

### Bekräftat

- Build fingerprint och runtime-egenskaper anger Sony Android 7.0, API 24, SVP-DTV15, release 5.433.
- Kernel rapporterar 3.10.79.
- `mali_kbase` är laddad enligt `/proc/modules`.
- De tre hämtade filerna ovan har SHA256 som matchar motsvarande lokala 5.433-filer.
- SELinux är Enforcing; ADB-shell kör som `shell` i `u:r:shell:s0`.
- `/dev/mali0` har DAC `0666` och SELinux-label `gpu_device`.
- Tidigare lokal analys av den matchande firmwarepolicyn fann `allow untrusted_app gpu_device:chr_file { append getattr ioctl lock open read write };`.
- Den tidigare statiska kodjämförelsen fann att den analyserade Mali-modulen saknar Googles CVE-2023-4211-fix. Se [jämförelserapporten](sony-5433-cve-2023-4211-google-fix-comparison.md).

### Slutsatser med stöd i bevisen

- DAC tillåter läsning och skrivning av device-noden för alla UID/GID, eftersom lägesbitarna är 0666. SELinux kan ändå neka en process om dess domain saknar relevant `allow`.
- För `untrusted_app` finns i den analyserade policyn en uttrycklig tillåtelse för `open/read/write/ioctl` på objekt av typen `gpu_device` och klassen `chr_file`. Eftersom `/dev/mali0` bär den typen är den direkta öppningen tillåten av dessa två policy-/DAC-lager på konfigurationsnivå.
- Kombinationen av vanlig appdomänens SELinux-regel, nodens DAC-läge och saknad drivrutinsfix gör den sårbara komponenten praktiskt relevant för appar som kan köra i den domänen och nå grafikgränssnittet.

### Inte verifierat

- Ingen vanlig app startades eller installerades och ingen `open()` mot `/dev/mali0` gjordes. Faktisk syscall-framgång från en appprocess är därför inte empiriskt bekräftad.
- Inga Mali-ioctl, mmap, EGL/GLES-rendering eller andra GPU-operationer testades. Drivrutinens efterföljande kontrollflöde, minneshantering och stabilitet har inte dynamiskt validerats.
- `map` är inte en separat SELinux-permission i den relevanta typen/klassen. SELinux-allow för device-node betyder inte automatiskt att en viss `mmap()` lyckas; den beror även på drivrutinens fops, VMA-flaggor och kernelkontroller. Detta testades inte.
- `allowxperm`/ioctl-kommandobegränsningar kunde inte verifieras som effektiva runtime-begränsningar; den tidigare policyanalysen fann ingen relevant målregel som skulle kunna användas för att hävda en begränsad ioctl-lista. Ingen ioctl skickades.
- `/sys/module/mali_kbase/version` nekades; versionen r14p0-01rel0 stöds i stället av den matchande lokala modulens innehåll och tidigare statiska analys, inte av läsbar runtime-versionsträng.
- Installationssubversionen `.0183EUB` kan inte härledas enbart från fingerprinten `5.433`; den här runtime-kontrollen verifierar 5.433-builden, inte Sony-paketets underrevision separat.
- ADB TCP-portens nåbarhet utanför den befintliga anslutningen, nätverksisolering och auktoriseringsnycklarnas förvaltning undersöktes inte.

## 3. Säkerhetsbedömning av möjliga tester

| Möjligt test | Vad det skulle visa | Risk | Beslut enligt denna validering |
|---|---|---|---|
| Läsande properties, `uname`, `getenforce`, `ls -lZ`, procfs | Runtime-identitet, DAC, label, SELinux-läge | Låg; läsning kan nekas men ändrar inte systemet | Utfört |
| `adb pull` av läsbara build- och drivrutinsfiler, lokal hash | Byteidentitet mot firmwareextrakt | Låg för TV; lagrar kopior lokalt | Utfört |
| Installera eller köra en testapp som endast gör `open()` på `/dev/mali0` | Empirisk direktåtkomst från appdomän | Kräver förbjuden nodöppning; öppning kan initiera drivrutinstillstånd och exponera felvägar | Avstått |
| EGL/GLES-appsmoketest | Om grafikstacken initierar GPU för app | Kan indirekt öppna noden och skicka ioctl; skapar drivrutinsarbete, kan hänga GPU eller påverka skärm/TV | Avstått |
| Mali-ioctl-, mmap- eller lifecycle-test | Funktionella drivrutinsvägar | Kan nå den sårbara kodvägen; risk för use-after-free, minneskorruption, kernel panic eller omstart | Förbjudet och avstått |
| Körning via `run-as`, root eller ändrad SELinux-policy | Ökad insyn/rättigheter | Bypass/privilegieförändring och utanför omfattningen | Inte försökt |
| Ren läsning av policy och kod på lokala firmwarefiler | Vilka åtkomstregler och sårbara kodvägar är inbyggda | Låg, statisk evidens kan inte styrka runtime-syscall-beteende | Rekommenderad grundmetod |

Att inte öppna enheten innebär att man inte säkert kan bekräfta den praktiska syscall-gränsen på TV:n. Det är en avsiktlig gräns: de åtgärder som skulle ge starkare dynamisk evidens är samma åtgärder som börjar aktivera den sårbara drivrutinen.

## 4. Isolerad reproduktionsmiljö

### Vad som krävs för hög fidelitet

En trovärdig reproduktionsmiljö för den faktiska Sony-drivrutinen skulle sannolikt kräva ett separat SVP-DTV15-kort eller en kompatibel Sony-TV med samma SoC/board-revision och Mali-T624 Midgard-GPU, tillsammans med Sony-kärnan 3.10.79, exakt `mali_kbase` r14p0-01rel0, rätt device tree, IOMMU/MMU-konfiguration, avbrott, klockor/regulatorer samt matchande firmware- och userspace-bibliotek. Att bara bygga en Android-app eller starta samma Android-version på annan hårdvara reproducerar inte denna kernelmodul eller dess minnes-/MMU-beteende.

Riktig T624-hårdvara behövs sannolikt för att validera hårdvaru- och IOMMU-interaktioner. En emulator kan däremot användas till app-/frameworkflöden eller ett abstrakt test av ägar- och livscykelmodellen, men inte som bevis på att Sony-modulens CVE-kodväg fungerar på samma sätt.

AOSP beskriver att Cuttlefish grafikacceleration skickar grafikkommandon till värdens GPU eller använder SwiftShader som mjukvarurenderare; det är inte en emulering av Mali-T624 eller Sony-kärnmodulen. Cuttlefish är även utformat för framework-funktionalitet och har skillnader vid hårdvaru-HAL:er. [AOSP: Cuttlefish GPU graphics acceleration](https://source.android.com/docs/devices/cuttlefish/gpu), [AOSP: Cuttlefish](https://source.android.com/docs/devices/cuttlefish)

### Isolering och återställning om en separat bänk senare godkänns

- Använd endast ett separat reservkort/enhet, aldrig TV:n som används dagligen.
- Koppla bänken till ett isolerat nät utan WAN och utan åtkomst till hemnätet; begränsa ADB till en dedikerad analyseringsvärd.
- Innan någon framtida felutlösande aktivitet måste en dokumenterad återställningsväg verifieras: återställningsmedia för exakt board/revision, fungerande recovery-/serviceförfarande, seriell konsol om tillgänglig, samt fysisk strömbrytning. Att dra ström kan återställa en hängd enhet men garanterar inte återställning från bestående skada; detta har inte verifierats här.
- Säkra kända firmwarebilder och konfigurationsmetadata separat innan användning. Ändra eller flasha inte TV:n som del av nuvarande uppdrag.
- Den föreslagna miljön är för passiv observation och icke-sårbar funktionell validering. CVE-trigger, exploit eller race-körning ingår inte i planen.

### Vad som kan simuleras utan Mali-hårdvara

- Statisk analys av den exakta `mali_kbase.ko` och jämförelse mot Googles patch.
- Modellering av referensräkning, VMA-/processlivscykel och förväntade kontrollflöden i en fristående, icke-kernelbaserad modell.
- Kontroll av app-/frameworkbeteenden i emulator, om testen inte påstår sig verifiera Mali-driveråtkomst.

Sådan simulering kan kontrollera logik och hypoteser, men kan inte bevisa kernelns faktiska VMA-callbacks, processminneslivslängd, timing, låsning eller T624-MMU-effekter.

## 5. Slutsats och nästa steg

Utan exploitförsök kan man redan dra en betydande, men avgränsad slutsats: **den verifierade firmwarebuilden innehåller den sårbara Mali-modulen, och statisk SELinux-policy ger `untrusted_app` direkt tillåtelse till relevanta character-device-operationer på `gpu_device`; `/dev/mali0` har samtidigt DAC 0666.** Därmed är en vanlig appdomän inte isolerad från Mali på policy-/DAC-lagren. Detta är en exponeringsbedömning, inte en demonstration av exploaterbarhet eller ett bevis på att varje tänkbar GPU-operation lyckas.

För fortsatt arbete rekommenderas att bevara hasharna och rapporterna, kontrollera kod och policy mot officiell patchkälla, och söka en leverantörsfix eller leverantörens interna valideringsresultat. Om dynamisk validering blir nödvändig ska den ske först på ett separat, matchande reservkort efter att återställningsmetod och bänkisolering verifierats. Denna rapport rekommenderar inte att CVE-kodvägen triggas.

## Källor och lokala artefakter

- Runtime-snapshot: `work/runtime-snapshot-2026-11-04/` (`build.prop`, `mali_kbase.ko`, `libGLES_mali.so`).
- Firmwareextraktion och policy: `sony-5433-firmware-extraction-and-policy.md`.
- Statisk fixjämförelse: `sony-5433-cve-2023-4211-google-fix-comparison.md`.
- [AOSP Cuttlefish GPU acceleration](https://source.android.com/docs/devices/cuttlefish/gpu).
- [AOSP Cuttlefish overview](https://source.android.com/docs/devices/cuttlefish).
- [Google Project Zero: CVE-2023-4211 root-cause analysis](https://googleprojectzero.github.io/0days-in-the-wild/0day-RCAs/2023/CVE-2023-4211.html).
- [Google kernel patch](https://android.googlesource.com/kernel/google-modules/gpu/+/35feb9b795cf1cd0d9a0a2edb6ade3c83040f48b%5E!/).
