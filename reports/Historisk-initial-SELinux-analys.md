> Historisk preliminär rapport skriven innan matchande 5.433-policy fanns lokalt. Dess slutsats om okänd apppolicy ersattes av den senare extraktionsbaserade analysen i `sony-5433-firmware-extraction-and-policy.md`.

# Sony KD-55XD9305: SELinux-policy och Mali-åtkomst

**Datum:** 2026-11-04\
**Mål:** Sony KD-55XD9305 / `SVP-DTV15`, ADB `<private-TV-IP>:5555`\
**Omfattning:** Passiv läsning av lokal firmwarekopia och TV-egenskaper/grafikmetadata. Ingen root, policyändring, enhetsöppning, Mali-ioctl, exploit eller fuzzning.

## Sammanfattning

Den exakta aktiva SELinux-policyn kunde inte hämtas från den lokala firmwarekopia som hittades. Downloads innehåller `sony_tvupdate_2015_5457_eub_auth.zip`, men ZIP:en innehåller bara ett stort Sony `.pkg`-objekt. Ingen extraherad system/vendor-avbildning, policyfil, `file_contexts` eller policykälla hittades i de sökta Downloads- och Codex-analyskatalogerna. En strömmande passiv sökning genom `.pkg`-payloaden (1,603,358,316 byte) hittade inga av markörerna `gpu_device`, `untrusted_app`, `platform_app`, `file_contexts`, `se_policy`, `sepolicy` eller `android.permission`. Detta visar inte att innehållet är krypterat; payloadformatet kunde inte avkodas här.

Följaktligen kan jag inte redovisa faktiska `allow`-/`neverallow`-/`allowxperm`-regler eller avgöra exakt vilka domäner som har `open`, `read`, `write`, `ioctl`, `map` och `getattr` på `/dev/mali0`. Jag fyller inte i sådana rättigheter med antaganden.

Bekräftat från TV:n:

- SELinux är `Enforcing`; `/dev/mali0` har etiketten `u:object_r:gpu_device:s0`.
- `/dev/mali0` har DAC-läge `0666`, `root:root`. Detta tar bort DAC-begränsning per UID, men beviljar inte i sig SELinux-behörighet.
- ADB-shell kör i `u:r:shell:s0`. Shell kunde lista nodens metadata, vilket bekräftar metadataåtkomst/`getattr` för shell i den observerade miljön; det bekräftar inte `open`, `read`, `write`, `ioctl` eller `map`.
- Andra processers FD-tabeller nekades. Policyn på `/sys/fs/selinux/policy` nekades vid läsning.
- Grafikmetadata bekräftar Mali-T624 och driversträngen `Midgard r14p0-01rel0`; TV:ns EGL-konfiguration väljer Mali. SurfaceFlinger rapporterar denna EGL-implementation.

## Källor och metod

### Lokal firmware

Sökta lokala platser: `<local-downloads>` och relevanta Codex-kataloger under `<local-Codex-workspace>`. Filträffarna omfattade inte någon extraherad policy, `file_contexts`, `.cil`, `.te`, `.fc`, `sepolicy` eller system/vendor-image. Den hittade uppdateringsfilen är:

- `<local-downloads>\sony_tvupdate_2015_5457_eub_auth.zip`
- ZIP-medlem: `sony_dtv0FA50A09A0A9_00014100_155100c1.pkg`, 1,603,358,316 byte efter ZIP-dekomprimering.

Filnamnet antyder en annan/nyare paketversion än den TV-egenskap som tidigare lästs (`PKG5.433.0183EUB`), men paketversionen har inte verifierats ur dess payload. Jag behandlar därför inte detta paket som bevis för att dess policy matchar den installerade builden. Den råa payloaden söktes strömmande efter policy-/domänmarkörer; inga träffar. Ingen okänd firmware-extraktor kördes.

Inga lokalt tillgängliga `seinfo`, `sesearch`, `checkpolicy`, `secilc`, `apol` eller `binwalk` hittades på PATH i denna körning. Standardplatserna för SELinux-policy på den aktuella Android-versionen hade tidigare kontrollerats på TV:n men ingen läsbar policykälla hittades; `/sys/fs/selinux/policy` gav `Permission denied` som shell.

### TV och grafikstack

Läsande ADB-resultat:

- `getenforce` → `Enforcing`.
- `ls -lZ /dev/mali0` → `crw-rw-rw- root root u:object_r:gpu_device:s0`, major/minor `10:49`.
- `/dev/mali` saknas.
- `/system/lib/egl/egl.cfg` innehåller `0 0 android` och `0 1 mali`.
- `/system/lib/egl/libGLES_mali.so` finns (ca 4.8 MB); `/system/lib/libEGL.so`, GLES-bibliotek och `/system/lib/hw/gralloc.mt5890.so`, `/system/lib/hw/hwcomposer.mt5890.so` finns.
- `dumpsys SurfaceFlinger` rapporterar `EGL implementation : 1.4 Midgard-"r14p0-01rel0"` och `GLES: ARM, Mali-T624, OpenGL ES 3.1 ...`; dess tillstånd visar även hårdvarucompositor/HWC.
- Processlistan visar `surfaceflinger` som UID `system`, domän `u:r:surfaceflinger:s0`, Android `system_server` i `u:r:system_server:s0`, och appdomäner inklusive `untrusted_app`, `platform_app`, `priv_app`, `system_app` och `sony_platform_app`.

Dessa resultat kommer från läsande kommandon. Inget program öppnade `/dev/mali0` och inga ioctl-anrop gjordes.

## DAC och SELinux är separata kontroller

Unix/DAC-läget `0666` betyder att ägare, grupp och övriga inte skiljs åt av mode-bitarna vid läs-/skrivåtkomst. SELinux gör en separat kontroll utifrån processens subjektkontext (`untrusted_app`, `surfaceflinger`, etc.), nodens objektkontext (`gpu_device`) och klass/permissons (`chr_file`). Ett `allow` krävs fortfarande för relevant operation när SELinux är enforcing.

För `ioctl` kan vanlig `ioctl`-behörighet i `chr_file` vara relevant; policy kan dessutom begränsa kommandon med `allowxperm`/extended permissions. Utan policyfil eller en auktoriserad policy-query går det inte att avgöra vilka ioctl-kommandon, om några, är tillåtna. `/dev/mali0`-läget kan därför inte användas som ersättning för SELinux-analys.

## Domänmatris för `/dev/mali0`

`Ej fastställd` betyder att faktisk policybehörighet inte kunde avgöras. Tabellen anger inte att rättigheten saknas eller finns.

| Processdomän | DAC för device node | open | read | write | ioctl / allowxperm | map | getattr | Evidens/anteckning |
|---|---|---|---|---|---|---|---|---|
| `untrusted_app` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Processdomän finns; policy/FDer ej tillgängliga |
| `platform_app` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Processdomän finns; policy/FDer ej tillgängliga |
| `priv_app` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Processdomän finns; policy/FDer ej tillgängliga |
| `system_app` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Processdomän finns; policy/FDer ej tillgängliga |
| `shell` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | **Bekräftad metadataåtkomst** | `ls -lZ` kunde läsa metadata/etikett; inga enhetsöppningar gjordes |
| `surfaceflinger` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Domän/process bekräftad och EGL rapporterar Mali; nod-FD nekad |
| `system_server` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Domän/process bekräftad; direkt Mali-användning ej belagd |
| `sony_platform_app` | Mode-bitarna tillåter | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Domän finns; policy/FDer ej tillgängliga |

Matrisen är avsiktligt konservativ. Inga exakta policyregler eller ioctl-kommandon kunde citeras eftersom policykälla saknas/är oläsbar.

## Grafikstack och sannolik åtkomstväg

**Bekräftat:** EGL-loaderns konfiguration väljer `mali`; Sony/MediaTek-bibliotek för Mali EGL/GLES, gralloc och HWC finns; SurfaceFlinger rapporterar Mali-T624 och Midgard r14p0. SurfaceFlinger kör i sin egen SELinux-domän. HWC visas som aktiv del av SurfaceFlinger-tillståndet.

**Teknisk slutsats, inte direkt observerad åtkomst:** Android-appar som använder GLES/OpenGL ES går normalt genom Android EGL/HWUI-gränssnittet, varefter EGL-loadern laddar leverantörens Mali-implementering. GPU-arbete kan därmed ske i app-processen; SurfaceFlinger använder EGL/GLES för komposition där HWC inte hanterar lager direkt. Exakt implementation kan använda direkta `/dev/mali0`-öppningar från klientprocesser och/eller leverantörskomponenter. Metadata från denna TV gör en sådan direkt Mali-väg sannolik, men utan processkartor/FD-tabeller och SELinux-policy kan ingen av dessa två mekanismer eller en eventuell broker fastställas för varje Sony-domän.

Jag hittade ingen separat, tydligt namngiven Mali-brokerprocess i den tidigare processlistan. Frånvaro av en uppenbar process i listan bevisar inte att grafikstacken saknar annan mediation.

## CVE-2023-4211 och praktisk exponering

Den tidigare statiska analysen uppges ha identifierat en starkt belagd sårbar kodväg i Mali-T624-r14p0. Denna SELinux-analys varken validerade sårbarheten eller testade drivrutinsgränssnittet.

- Om `untrusted_app` har `open` och tillräcklig `ioctl`-behörighet mot `gpu_device`, kan en vanlig appdomän potentiellt nå den sårbara kärnvägen direkt. Det är **inte bekräftat**.
- Om `untrusted_app` nekas, men `surfaceflinger`/privilegierad grafikstack tillåts, begränsas direkt åtkomst från vanliga tredjepartsappar, men sårbarheten kan fortfarande nås via en privilegierad komponent om den exponerar en pass-through-yta. Sådan pass-through har **inte** undersökts.
- `0666` innebär att DAC inte är ett skyddslager mellan dessa app-UID:er. I enforcing-läget blir SELinux-policyn den avgörande MAC-gränsen för device node.

Därför kan den praktiska app-till-kärna-exponeringen för CVE:n inte klassificeras exakt förrän policyreglerna och eventuell grafikbroker/pass-through är kända. Högsta prioritet är att få tag på policy som matchar exakt TV-build `PKG5.433.0183EUB` och analysera `gpu_device`-regler plus eventuella `allowxperm`-regler lokalt.

## Slutsats och begränsningar

**Bekräftat:** node-label, DAC-läge, SELinux enforcing, domännamn i processlistan, aktiv Mali-EGL-konfiguration, installerad EGL/GLES/Gralloc/HWC samt rapporterad Mali-T624/r14p0.

**Ej bekräftat:** varje efterfrågad SELinux-behörighet förutom shell-metadata/`getattr`; exakt policykälla och `file_contexts` för installerad build; processer med en öppen Mali-FD; om tredjepartsappar har direkt åtkomst eller om vissa vägar medieras; specifika tillåtna ioctl-nummer; policykopplingens exakta relation till CVE-2023-4211.

**Begränsning:** den enda lokala Sony-firmwarefil som hittades är ett opakt `.pkg` i ett auth-zip och matchar inte bevisligen den installerade builden. Den råa payloaden kunde inte avkodas eller rekonstrueras till policy i denna analys. Jag har inte ändrat eller extraherat över TV-firmwaren.
