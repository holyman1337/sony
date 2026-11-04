> Historisk firmwarekällsökning före lokal extraktion av 5.433. Senare verifieringsstatus finns i `sony-5433-firmware-extraction-and-policy.md`.

# Sony KD-55XD9305: exakt firmware och SELinux-policy

**Datum:** 2026-11-04\
**TV:** Sony KD-55XD9305 / SVP-DTV15\
**ADB:** `<private-TV-IP>:5555`\
**Analys:** Passiv versionsmetadata, lokal filinventering och statisk firmware-/policykontroll. Ingen root, SELinux-bypass, bootloaderåtgärd, firmwareändring, Mali-enhetsöppning eller ioctl-anrop.

## Sammanfattning

TV:n rapporterar `PKG5.433.0183EUB` och build fingerprint `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys` direkt från ADB. En historisk Sony Community-post från april 2019 dokumenterar version 5.433 för 2015/2016 Android-TV-apparater och listar uttryckligen KD-55XD9305. Sony Community-posten länkar till en Sony-supportsida för nedladdningen. En dokumenterad firmwarearkivpost namnger motsvarande EUB-paket `sony_tvupdate_2015_5433_eub_auth.zip`, versionerar det som `PKG5.433.01EUB` och listar KD-55XD9305 som kompatibel.

**Matchningsbedömning:** 5.433 **EUB är den starkt matchande officiella releasefamiljen** för TV:ns build 5.433 EUB. Det faktiska 5.433-paketet finns dock inte i de lokala sökvägarna, så jag kunde inte kontrollera dess payload, build `NRD91N.S34`, hash eller SELinux-policy. Den exakta `.0183`-builden är bekräftad på TV:n men kunde inte verifieras mot 5.433-paketets binära metadata.

Den lokala extraherade katalogen `<superseded-local-firmware-extract>` är **inte** 5.433. Dess `sysm/build.prop` säger `NRD91N.S44/5.457`, och den innehåller endast ett `sysm`-träd – ingen boot/ramdisk och ingen SELinux-policy. Den tidigare extraherade Mali-modulen hör därmed till en 5.457-katalog och används inte som ersättning för 5.433.

**SELinux-resultat:** ingen regel kunde fastställas från en verifierad 5.433-policy. Domänbehörigheter för `/dev/mali0` markeras därför som **ej fastställda** nedan. Inga behörigheter har härletts från nodens DAC-läge 0666.

## 1. Identifiering av firmware

### TV:ns aktiva version – direkt ADB-evidens

`getprop` och `/system/build.prop` gav:

| Egenskap | Värde |
|---|---|
| `ro.svp.package_version` | `PKG5.433.0183EUB` |
| Build fingerprint | `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys` |
| Android / API | 7.0 / 24 |
| Säkerhetspatch | `2019-02-01` |
| Build date | `Fri Mar 8 22:28:24 JST 2019` |
| Produkt/enhet | `SVP4KDTV15_EU` / `SVP-DTV15` |

Paketversionen och fingerprinten är överens om 5.433 och EUB.

### Historiska Sony-versioner och regionala paket

| Version | Dokumentation/status | Betydelse för denna analys |
|---|---|---|
| **5.433 EUB** | Sony Community-post från april 2019 rubricerar 5.433 och listar KD-55XD9305. Den länkar till Sonys supportsida för nedladdningen. En dokumenterad arkivpost anger `sony_tvupdate_2015_5433_eub_auth.zip`, `PKG5.433.01EUB`, publiceringsdatum på Sony 10 april 2019 och listar KD-55XD9305. | Rätt releasefamilj/region för TV:n. Binären saknas lokalt; innehållet är inte verifierat. |
| **5.435** | Sony meddelade officiellt utrullning från 26 juni 2019 för 2015–2016 Android TV-serier inklusive XD93. | Senare release. Används inte som ersättare. Exakt EUB-paket/hash har inte hämtats. |
| **5.457 EUA/EUB** | Sony Community dokumenterar 5.457 och länkar till separata `...5457_eua...` och `...5457_eub...` Sony-hostade paket. En Sony-forumanvändare rapporterar att EUA-variant krävdes när EUB-paketet inte accepterades på en EUA-TV. | Senare och regionalt särskiljd. EUB-kopian finns lokalt men är inte TV:ns installerade version och används inte för policyanalysen. |

Sony säger att Android-TV-firmware kan skilja sig mellan regioner och att paket med samma versionsnummer inte nödvändigtvis har samma funktioner/tjänster. Det styrker att suffixet måste beaktas, men källorna fastställer inte en fullständig lands-/suffix-mappning för alla modeller. TV:n är direkt verifierad som EUB, så EUA ska inte väljas.

Den aktuella Sony-produktsidan för KD-55XD9305 visar nu en post om upphörda programuppdateringar, daterad 22 december 2025, och listar inte längre firmwarefilen. Den äldre Sony-supportlänken `.../downloads/00016021` finns i den historiska Sony Community-tråden men gick inte att hämta i denna körning (timeout/403). Jag har inte konstruerat någon nedladdningsadress.

Källor:

- [Sony Community: v5.433 och modellista inklusive KD-55XD9305](https://community.sonylatvija.com/t5/android-tv/new-sony-android-tv-firmware-update-models-2015-first-half-2016/td-p/2603458)
- [Sony Community: Sony-supportlänk och filnamn/storlek för 5.433 EUB](https://community.sony.co.uk/t5/android-tv/neue-software-fur-xc-serie/td-p/2604309)
- [Dokumenterat firmwarearkiv: PKG5.433.01EUB och kompatibla modeller](https://www.touslesdrivers.com/index.php?v_code=60916&v_langue=en&v_page=23)
- [Sony: utrullning av v5.435 från 26 juni 2019](https://www.sony.co.uk/electronics/support/televisions-projectors-lcd-tvs-android-/kd-55xd9305/articles/00229299)
- [Sony: regionberoende skillnader i Android-TV-firmware](https://www.sony.de/electronics/support/articles/00152849)
- [Sony Community: EUA- och EUB-paket kan vara inkompatibla mellan regionala varianter](https://community.sony.es/t5/android-tv/software-update-of-sony-bravia-2015-to-v-5-457-through-usb-is-not/td-p/3960409)
- [Sonys aktuella KD-55XD9305-supportsida](https://www.sony.co.uk/electronics/support/televisions-projectors-lcd-tvs-android-/kd-55xd9305/downloads)

## 2. Lokal firmwareinventering och SHA256

### Lokal kopia i WSL

`<superseded-local-firmware-extract>` innehåller ett extraherat `sysm`-träd (cirka 1,9 GB, 2 026 filer). Dess `sysm/build.prop` visar:

- `SVP4KDTV15_EU-user 7.0 NRD91N.S44 5.457 release-keys`
- fingerprint `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S44/5.457:user/release-keys`
- build date `Sat Oct 17 19:07:32 JST 2020`
- modell/plattform `BRAVIA 4K 2015`, `SVP-DTV15`, `mt5890`

Detta skiljer sig från TV:ns `.S34/5.433`-build. Trädet innehåller ingen separat boot-/ramdisk-avbildning.

SHA256 för filer som faktiskt finns lokalt:

| Fil | SHA256 | Tolkning |
|---|---|---|
| `<local-downloads>\sony_tvupdate_2015_5457_eub_auth.zip` | `479D490A1B19ECDFC2754F52F25FB8EBBC64AD472779C33F7E82514D298BB29D` | Lokal 5.457 EUB-kandidat, inte installerad version |
| `<superseded-local-firmware-extract>/sysm/build.prop` | `9198d5a42ac8992f787987662275d5f8be2bfeb165bcbbe3a4c93662bc1308f7` | Bekräftar att extraherat `sysm`-träd är 5.457 |
| `<superseded-local-firmware-extract>/sysm/lib/modules/mali_kbase.ko` | `ca7fc836eed71f359bcd5388ba9d6e01ae8eb36961932e4957a382aff2bbb140` | Mali-modul i samma 5.457-träd; ELF ARM, `vermagic` 3.10.79, innehåller `K:r14p0-01rel0(GPL)` |

Ingen lokal `sony_tvupdate_2015_5433_eub_auth.zip`, `.pkg` för 5.433 eller 5.433 build-/boot-image hittades. Därför finns ingen SHA256 för en matchande 5.433-binär.

### SELinux-filer i den extraherade katalogen

Sökning i hela `<superseded-local-firmware-extract>` efter `sepolicy`, `file_contexts`, `seapp_contexts`, `property_contexts`, `.cil`, `.te` och `.fc` hittade endast:

- `sysm/etc/security/mac_permissions.xml` (signatur-/behörighetsmetadata, inte den kompilerade SELinux-policyn).

Ingen `sysm/etc/selinux`-katalog, policyfil, policyfragment eller `file_contexts` hittades. Inga lokala policyanalysverktyg (`sesearch`, `seinfo`, `checkpolicy`, `secilc`, `apol`, `binwalk`) fanns på PATH i WSL.

Den saknade policyn kan inte rekonstrueras från `mali_kbase.ko`, `build.prop` eller Android-apparnas katalogstruktur. Ett komplett firmwarepaket som innehåller den för build 5.433 motsvarande boot-/ramdiskdelen krävs; standardkonfiguration på Android kan bära den aktiva policyn där, men Sony-layouten har inte verifierats för denna build.

## 3. Ytterligare läsande ADB-metadata

| Kontroll | Resultat |
|---|---|
| `/system/build.prop` | Läsbar; överensstämmer med TV:ns `.S34/5.433`-fingerprint |
| `/vendor/build.prop` | Saknas |
| `/default.prop` | `Permission denied`; ingen åtkomst kringgicks |
| `/system/etc/security` | Katalogen visar `cacerts`, `mac_permissions.xml`, `otacerts.zip`; deras innehåll lästes inte |
| `/system/etc/selinux` | Saknas |
| `/vendor/etc/selinux` | Saknas |
| Aktiv SELinux-policy `/sys/fs/selinux/policy` | ADB-shell fick `Permission denied` vid tidigare passiv läsning; inget rootförsök gjordes |
| `/system` | `/dev/block/mmcblk0p44`, squashfs, read-only |
| `/linux_rootfs` | `/dev/block/dm-0`, squashfs, read-only |
| `/3rd` | `/dev/block/dm-1`, squashfs, read-only |
| `/perm` | `/dev/block/mmcblk0p4`, ext4, read-only |
| `/data`, `/3rd_rw`, `/cache`, `/upgrade` | Skrivbara mountar enligt `/proc/mounts`; inga skrivförsök gjordes |

ADB-shell tilläts läsa `/system/build.prop`, men nekades `/default.prop` och den aktiva policyn. Jag försökte inte komma runt dessa spärrar.

## 4. SELinux-behörigheter för `/dev/mali0`

Bekräftade fakta från tidigare direkt avläsning av TV:n:

- SELinux: `Enforcing`.
- `/dev/mali0`: `crw-rw-rw- root root`, DAC 0666.
- SELinux-label: `u:object_r:gpu_device:s0`.
- `/dev/mali` saknas.
- ADB-shell-domän: `u:r:shell:s0`.
- ADB-shell kunde läsa nodens metadata/label med `ls -lZ`, men öppnade inte enheten.
- Läsning av andra processers FD-tabeller nekades.

DAC 0666 betyder att mode-bitarna inte skiljer mellan UID:er för läs-/skrivöppning. Det beviljar inte i sig åtkomst i SELinux. Med SELinux enforcing måste den effektiva policyn också tillåta rätt operation för processens domän mot `gpu_device:chr_file`. `ioctl` kan dessutom vara begränsad per ioctl-nummer genom extended-permission-regler (`allowxperm`). Ingen av dessa regler kan bestämmas utifrån DAC-rättigheten.

### Efterfrågad domänmatris

Alla celler utom shell-radens observerade metadataåtkomst är **ej fastställda**, inte bekräftat nekade. Ingen matchande kompilerad 5.433-policy fanns tillgänglig att fråga med `sesearch` eller motsvarande.

| SELinux-domän | `open` | `read` | `write` | `ioctl` / `allowxperm` | `map` | `getattr` |
|---|---|---|---|---|---|---|
| `untrusted_app` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd |
| `platform_app` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd |
| `priv_app` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd |
| `system_app` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd |
| `shell` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Metadataåtkomst observerad; exakt SELinux-regel ej fastställd |
| `surfaceflinger` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd |
| `system_server` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd |
| `sony_platform_app` | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd | Ej fastställd |

## 5. Grafikstack, direktåtkomst och tredjepartsappar

Tidigare läsande metadata från TV:n visar att EGL-konfigurationen väljer `mali`, `/system/lib/egl/libGLES_mali.so` finns och SurfaceFlinger rapporterar `Midgard r14p0-01rel0`, Mali-T624, OpenGL ES 3.1 samt en aktiv HWC/hårdvarucompositor. Android-appar når grafik-API:er via EGL/GLES och SurfaceFlinger/HWC hanterar komposition. Det gör Mali-kärndrivrutinen till en praktiskt relevant del av grafikstacken.

Det går däremot inte att avgöra från dessa metadata om en app-process öppnar `/dev/mali0` direkt, om vissa uppgifter går via en privilegierad grafikprocess, eller vilka Sony-domäner som är tillåtna. Varken FD-/maps-data för andra processer eller SELinux-policy finns tillgängliga i den verifierade 5.433-källan. Ingen särskild Mali-broker har bekräftats.

**Bedömning för vanliga tredjepartsappar:** direkt åtkomst är möjlig i den generella grafikarkitekturen om processdomänens policy ger rätt behörigheter, men är inte bekräftad för denna Sony-build. Det vore fel att dra slutsatsen att appar har åtkomst enbart från 0666 – och lika fel att hävda att de saknar den utan policy.

## 6. Inverkan på CVE-2023-4211

TV:ns build/propertydata och tidigare SurfaceFlinger-utdata pekar på Mali-T624 med r14p0-generationen. Den lokala `mali_kbase.ko`-filen är dock från ett extraherat 5.457-träd och kan inte användas som en verifierad kopia av 5.433-modulen. Den här uppgiften testar inte CVE:n.

Om `untrusted_app` har nödvändiga `open`- och ioctl-behörigheter mot `gpu_device`, skulle vanlig app-kod potentiellt kunna nå drivrutinsgränssnittet direkt; detta är okänt. Om domänen nekas kan privilegierade grafikkomponenter ändå använda GPU:n, men eventuell pass-through till kärn-gränssnittet har inte undersökts. Eftersom den faktiska 5.433-policyn saknas går det inte att klassificera CVE:ns app-till-kärna-räckvidd.

## 7. Nästa steg som krävs

För att slutföra den begärda exakta policyanalysen behövs **det faktiska Sony 5.433 EUB-paketet** (`sony_tvupdate_2015_5433_eub_auth.zip`, dokumenterad storlek 1,644,753,328 byte) eller en verifierad 5.433 boot/ramdisk-avbildning som innehåller policy. Den historiska Sony-produktlänken är dokumenterad, men den binära 5.433-filen kunde inte hämtas i denna körning och finns inte i lokala Downloads/WSL-kataloger. Sony-supportsidan visar numera avslutade programuppdateringar.

När den officiella 5.433-binären finns lokalt kan den extraheras till en separat arbetskatalog, dess build-ID jämföras med `NRD91N.S34/5.433`, och policyfiler hashas innan `seinfo`/`sesearch`-analys. Fram till dess ska 5.457-filerna inte användas som ersättning.

## Avgränsning

**Bekräftat:** TV:ns installerade 5.433.0183EUB, fingerprint, patchnivå, partitionsmounts, lokala sysm-trädets 5.457-version och SHA256, samt historisk dokumentation av 5.433 EUB för modellfamiljen/KD-55XD9305.

**Inte verifierat:** SHA256 eller binär metadata för 5.433-paketet; att arkivets paketinnehåll exakt motsvarar TV:ns `.0183`-subversion; policy från installerad build; `file_contexts` för installerad build; domänrättigheter per operation; direkt Mali-FD hos vanliga appar; specifika tillåtna ioctl-nummer; exploaterbar CVE-åtkomstkedja.

Ingen firmwarefil har laddats ned, extraherats eller ändrats i denna körning. TV:n och dess inställningar har inte ändrats.
