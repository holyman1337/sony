# Sony KD-55XD9305 – extraktion av firmware 5.433 EUB och SELinux/Mali-analys

**Analysdatum:** 2026-11-04\
**Omfattning:** Lokal, statisk analys av den tidigare hämtade Sony 5.433 EUB-firmwarekopian. TV:n kontaktades inte under denna analys. Ingen firmware installerades och ingen enhet, inställning eller behörighet på TV:n ändrades.

## Sammanfattning

Firmwarearkivet klarade förkontrollen: filen finns, storleken är exakt **1 644 753 328 byte**, SHA256 matchar det angivna värdet och ZIP CRC-testet passerar. Extraktion skedde därefter till `work/sony-5433-extracted/`.

Unixtract identifierade payloaden som det nya MediaTek PKG-formatet, valde sin inbyggda nyckelprofil **“Sony 2015 Android”** och extraherade 32 komponenter. `sysm`, `3rdp` och `rfsA` är SquashFS-bilder; `boot` och `recv` är Android boot images. Den normala boot-ramdisken innehåller en kompilerad SELinux-policy (v30), kompilerade filkontexter och Androids app-contextkonfiguration.

Den extraherade `sysm/build.prop` matchar TV:ns buildfingerprint exakt: `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys`. `cfig.bin` innehåller dessutom internidentifieraren `..._15390183`, vilket är starkt stöd för subversionsdelen `.0183`. Eftersom ingen dokumenterad Sony-mappning av den interna identifieraren till det fullständiga produktversionsnamnet hittades och paketets yttre `.pkg`-ID är `153900b7`, bedöms buildmatchningen som **bekräftad**, medan exakt identitet `PKG5.433.0183EUB` är **starkt underbyggd men inte självständigt verifierad som samma signerade binärversion**.

Den centrala policyregeln är tydlig: `untrusted_app`, `platform_app`, `priv_app`, `system_app`, `sony_platform_app`, `surfaceflinger`, `system_server` och `shell` har alla en explicit `allow`-regel mot `gpu_device:chr_file` med `open`, `read`, `write`, `ioctl` och `getattr` (samt `append` och `lock`). Det finns inga `allowxperm`-regler för `gpu_device:chr_file`. Den här SELinux-policyversionen definierar ingen separat `map`-permission, så en fristående SELinux-dom för `map` kan inte utläsas ur policyn.

Tillsammans med boot-image-konfigurationen `/dev/mali0 0666 root root` visar detta att vanliga appar som kör i `untrusted_app`-domänen har policy- och DAC-tillåtelse att öppna Mali-enheten och göra ioctl-anrop direkt. Det innebär att grafikstacken inte är en SELinux-barriär som begränsar Mali-åtkomst enbart till privilegierade grafikprocesser. Ingen ioctl eller annan operation utfördes mot TV eller Mali-enheten.

## 1. Filkontroll före extraktion

| Kontroll | Resultat |
|---|---|
| Sökväg | `work/sony_tvupdate_2015_5433_eub_auth.zip` |
| Finns | Ja |
| Faktisk storlek | 1 644 753 328 byte |
| Förväntad storlek | 1 644 753 328 byte |
| SHA256 | `fee0a3edebcd18da5ba429ca95869812c78d7c26eb6cf09096442482592aeb73` |
| Förväntad SHA256 | Samma värde |
| ZIP-innehåll | 1 post: `sony_dtv0FA50A09A0A9_00014100_153900b7.pkg` |
| ZIP CRC-test | Godkänt; `zipfile.testzip()` rapporterade inget fel |
| Extraktion påbörjad | Ja, först efter godkända kontroller |

Den extraherade `.pkg`-filen har storleken 1 644 473 468 byte och SHA256 `5bad529cbc7ef2e66d074467bc2f5d3cff09c260f3f77aeaaedcf08496d2cdd2`.

**Proveniensbegränsning:** ZIP-filen kommer från den TLD-hostade spegel som redovisades i föregående rapport. Ingen Sony-publicerad SHA256/signaturkontroll eller oberoende andra spegelhash finns i detta underlag. Firmwareinnehållets buildmetadata stämmer med TV:n, men detta ersätter inte en kryptografisk proveniensverifiering från Sony.

## 2. Paketformat och extraherade komponenter

Verktyget [Unixtract](https://github.com/theubusu/unixtract), källrevision `aadba866ea47ed9506ffe130a8558bb9147ee968`, rapporterade `mtk_pkg_new detected`, `Using key Sony 2015 Android`, produktnamn `Sakura_EU` och paketversionsträngen `t-apollo-mp-1501-1435-0-001-134-001-22`. De 32 komponenterna sparades under `work/sony-5433-extracted/parts/`.

| Komponent | Extraherad storlek (byte) | Filtyp/kommentar |
|---|---:|---|
| `ver.bin` | 1 048 576 | Ogenomskinlig firmwarekomponent |
| `b002.bin` | 1 048 576 | Ogenomskinlig firmwarekomponent |
| `b001.bin` | 1 048 576 | Ogenomskinlig firmwarekomponent |
| `minf.bin` | 304 | Paket-/komponentmetadata |
| `spst.bin` | 1 171 024 | Ogenomskinlig firmwarekomponent |
| `bost.bin` | 286 736 | Ogenomskinlig firmwarekomponent |
| `udt.bin` | 15 567 232 | Ogenomskinlig firmwarekomponent |
| `ddri.bin` | 8 336 | Ogenomskinlig firmwarekomponent |
| `ldat.bin` | 201 792 | Ogenomskinlig firmwarekomponent |
| `pdat.bin` | 1 972 896 | Ogenomskinlig firmwarekomponent |
| `ndat.bin` | 3 617 120 | Ogenomskinlig firmwarekomponent |
| `mlfw.bin` | 80 352 | Ogenomskinlig firmwarekomponent |
| `mafw.bin` | 2 893 344 | Ogenomskinlig firmwarekomponent |
| `dfw2.bin` | 9 648 | Ogenomskinlig firmwarekomponent |
| `dfw1.bin` | 24 736 | Ogenomskinlig firmwarekomponent |
| `bdix.bin` | 42 144 | Ogenomskinlig firmwarekomponent |
| `bdat.bin` | 2 820 864 | Ogenomskinlig firmwarekomponent |
| `rcip.bin` | 532 113 | Uppgraderings-/skriptdata, ej vidare exekverat |
| `cfig.bin` | 1 498 | Konfigurations-/partitionmetadata; innehåller identifieraren `..._15390183` |
| `adsp.bin` | 2 628 336 | Ogenomskinlig firmwarekomponent |
| `pnel.bin` | 33 105 | Ogenomskinlig firmwarekomponent |
| `edid.bin` | 11 280 | Ogenomskinlig firmwarekomponent |
| `aqda.bin` | 11 573 008 | Ogenomskinlig firmwarekomponent |
| `pqda.bin` | 69 795 840 | Ogenomskinlig firmwarekomponent |
| `3rdp.bin` | 80 572 416 | SquashFS 4.0, LZO |
| `tzbp.bin` | 1 097 664 | Ogenomskinlig firmwarekomponent |
| `rfsA.bin` | 79 609 856 | SquashFS 4.0, LZO; Sony Linux-rootfs-komponent |
| `recv.bin` | 14 939 436 | Android recovery boot image |
| `logo.bin` | 26 984 | JPEG-baserad logoresurs |
| `uenv.bin` | 5 364 | ELF-baserad miljö-/bootkomponent |
| `sysm.bin` | 1 339 318 272 | SquashFS 4.0, LZ4; innehåller Android systemfiler |
| `boot.bin` | 12 481 832 | Android boot image för normal boot |

Inga separata komponenter med namnen `system.img` eller `vendor.img` extraherades. I denna paketlayout ligger Android-materialet i komponenten `sysm`; analysen antar inte att 5.457-paketets struktur gäller.

### Boot images

`boot.bin` och `recv.bin` innehåller båda ett kernelområde på 4 585 152 byte och LZO-komprimerad ramdisk. Ramdiskarna packades upp från boot image-formatet och cpio-arkivet. Den normala boot-ramdisken har bland annat `sepolicy`, `file_contexts.bin`, `seapp_contexts`, `property_contexts`, `service_contexts`, `selinux_version`, `default.prop` och `ueventd.mt5890.rc`. Recovery-ramdisken innehåller motsvarande policy/context-filer; recovery-policyn analyserades separat och har samma `gpu_device:chr_file`-regler för domänerna i tabellen nedan. Den normala runtime-analysen utgår ändå från policyn i `boot.bin`.

### Extraktionsbegränsningar på analysdatorn

SquashFS-träden packades upp fullständigt i WSL:s Linux-filsystem under `work/sony-5433-extracted/`. Detta behövdes eftersom systembilden innehåller skiftlägeskänsliga sökvägar som kolliderar på standardmonterad Windows-filsystemyta. De ursprungliga och fullständiga komponentbilderna finns i den begärda arbetskatalogen. Policyfiler och relevanta build-/Mali-filer kopierades även till `work/sony-5433-extracted/analysis/`.

Extraktionen av `rfsA` kördes utan lokala superuser-rättigheter. Därför materialiserades inte SquashFS-avbildningens specialfiler för `/dev` som riktiga device nodes i Linux-trädet; råbilden behölls oförändrad och dess metadata kunde läsas. Detta påverkar inte extraktionen eller analysen av boot-ramdiskens SELinux-policy.

## 3. Verifiering av målbuild och Mali-version

### Bekräftad buildidentitet

`sysm/build.prop` innehåller:

- `ro.build.display.id=SVP4KDTV15_EU-user 7.0 NRD91N.S34 5.433 release-keys`
- `ro.build.version.incremental=5.433`
- `ro.build.version.release=7.0`
- `ro.product.device=SVP-DTV15`
- `ro.product.name=SVP4KDTV15_EU`
- `ro.build.fingerprint=Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys`
- `ro.build.type=user`, `ro.build.tags=release-keys`

Ramdiskens `selinux_version` innehåller samma fingerprint. Detta är en exakt matchning mot den angivna TV-builden `NRD91N.S34/5.433` och en direkt matchning för modellplattformen SVP-DTV15.

### Subversion `5.433.0183EUB`

- Ytterfilnamnet anger uttryckligen `5433_eub`.
- Paketets MediaTek-header identifierar Sony 2015 Android och produkt `Sakura_EU`.
- `cfig.bin` innehåller strängen `CTRL1V0FA50A09A0A9_00014100_15390183`; den avslutande `0183`-delen överensstämmer numeriskt med TV:ns angivna subversionssuffix.
- Ingen exakt textsträng `PKG5.433.0183EUB` hittades i de extraherade system-/metadatafilerna. ZIP:ens inre `.pkg`-filnamn slutar i stället på `_153900b7`.

**Bedömning:** Release-/buildmatchningen `5.433 / NRD91N.S34 / SVP-DTV15` är bekräftad av imageinnehållet. Internidentifieraren `15390183` gör matchningen med `.0183` starkt underbyggd. Eftersom det saknas dokumenterad Sony-mappning mellan denna interna `cfig`-identifierare och det fullständiga visningsnamnet, samt att ingen Sony-signatur/hash verifierats, markeras exakt `PKG5.433.0183EUB` som **inte formellt verifierad**.

### Kernel och Mali

- Kernelsträng i `boot`-kernel: `Linux-3.10.79`.
- `mali_kbase.ko` är ELF32 ARM EABI5; `.modinfo`/strängar anger `vermagic=3.10.79 SMP preempt mod_unload ARMv7`.
- Mali-modulens sträng anger `K:r14p0-01rel0(GPL)`.
- `sysm/lib/egl/libGLES_mali.so` är ELF32 ARM, dynamiskt länkad för Android API 24.
- `sysm/lib/egl/egl.cfg` innehåller implementationen `mali`.

Detta bekräftar att den extraherade 5.433-builden innehåller den förväntade kernelversionen och Mali-drivergrenen.

## 4. SELinux-policy och källor

Den analyserade policyn är den binära `sepolicy` från **normal boot-ramdisk i `boot.bin`**, inte en policy antagen från en annan firmwareversion. `file` identifierar den som SELinux policy v30; SETools läser den som policy version 30, MLS aktiverat. Policyöversikten anger 828 typer, 8 284 `allow`-regler, 111 `allowxperm`-regler, 0 permissiva typer och `Handle unknown classes: deny`.

Policyfilen är kompilerad. Ingen textkälla `.te`, `.cil` eller `.fc` hittades i de extraherade komponenterna. `file_contexts.bin` är kompilerad data, inte en läsbar källfil. `strings` i den binära filen visar `/dev/mali0` och kontexten `u:object_r:gpu_device:s0`; den visar även `/linux_rootfs/dev/mali0` och `/dev/pvrsrvkm`-relaterade context-strängar. Tillsammans med den tidigare läsande ADB-observationen av `/dev/mali0`-labeln stöder detta kopplingen till `gpu_device`. Den kompilerade context-tabellen dekompilerades inte till exakt SELinux-regexkälltext.

Boot-ramdiskens `seapp_contexts` mappar generiska app-UID:n (`user=_app`) utan plattformssignering/privilegier till `untrusted_app`; den innehåller separata rader för `platform_app`, `priv_app` och Sonys namngivna appar till `sony_platform_app`. Exempel är `com.sony.dtv.*`, `com.android.tv.settings` och Opera-paket. `untrusted_app` är därför den relevanta domänen för vanliga tredjepartsappar enligt den här firmwarekonfigurationen.

Policyn i `recv`-ramdisken har också SELinux policy v30 och en annan SHA256. Dess Mali-regler för domänerna som listas nedan är likadana i den granskade sökningen; normal användarprocessanalys utgår fortfarande från `boot`-policyn.

### Mali-behörigheter i normal boot-policy

Följande regler kommer direkt från `sesearch -A -s <domän> -t gpu_device -c chr_file` på den extraherade kompilerade policyn. “Ja” betyder att explicit `allow` finns.

| Domän | `open` | `read` | `write` | `ioctl` | `getattr` | Separat `map`-permission | `allowxperm` på Mali |
|---|:---:|:---:|:---:|:---:|:---:|---|---|
| `untrusted_app` | Ja | Ja | Ja | Ja | Ja | Definieras inte av policyformatets `file`-common i denna policy | Ingen Mali-regel |
| `platform_app` | Ja | Ja | Ja | Ja | Ja | Definieras inte | Ingen Mali-regel |
| `priv_app` | Ja | Ja | Ja | Ja | Ja | Definieras inte | Ingen Mali-regel |
| `system_app` | Ja | Ja | Ja | Ja | Ja | Definieras inte | Ingen Mali-regel |
| `sony_platform_app` | Ja | Ja | Ja | Ja | Ja | Definieras inte | Ingen Mali-regel |
| `surfaceflinger` | Ja | Ja | Ja | Ja | Ja | Definieras inte | Ingen Mali-regel |
| `system_server` | Ja | Ja | Ja | Ja | Ja | Definieras inte | Ingen Mali-regel |
| `shell` | Ja | Ja | Ja | Ja | Ja | Definieras inte | Ingen Mali-regel |

Var och en av dessa åtta regler ger exakt `{ append getattr ioctl lock open read write }` mot `gpu_device:chr_file`.

I denna policy saknar den gemensamma `file`-permissionlistan en separat permission med namnet `map`; SETools returnerar därför att permissionen inte finns i klassen. Det ska inte misstolkas som en explicit `allow map` eller `deny map`. Denna fråga avgör inte hur Linux 3.10:s SELinux-hooks validerar en konkret `mmap()`-väg; inga sådana anrop eller tester gjordes.

Det finns 111 `allowxperm`-regler i policyn totalt, men sökning mot `gpu_device:chr_file` gav ingen regel, vare sig för `untrusted_app` eller någon annan källa. Den policybaserade Mali-regeln ger vanlig `ioctl`-permission och innehåller ingen Mali-specifik `allowxperm`-lista som begränsar ioctl-kommandon. Detta är en policyobservation, inte ett test av vad driverens ioctl-dispatch accepterar.

### Övriga domäner med Mali-regler

Den fullständiga `sesearch`-listan för `gpu_device:chr_file` innehåller även:

- Full uppsättning `{ append getattr ioctl lock open read write }`: `adbd`, `autoplay_app`, `bluetooth`, `bootanim`, `dumpstate`, `mediacodec`, `mediaserver`, `nfc`, `radio`, `shared_relro`.
- `debuggerd`: `{ getattr ioctl lock open read }`.
- `init`: `{ getattr open read setattr }`.
- `mt_dtv-svc`: endast `getattr`.
- `vold`: `{ getattr ioctl lock open read }`.
- `ueventd`: regler för `dev_type`-devicehantering och `gpu_device`-relabelling, inte vanlig appöppning.

## 5. DAC, SELinux och direkt Mali-åtkomst

Boot-image-konfigurationen `ueventd.mt5890.rc` innehåller exakt:

```text
/dev/mali0                0666   root       root
```

Detta är DAC-konfigurationen: alla UID:n får filrättighetsmässigt öppna noden. DAC-regeln ensam skulle inte bevisa åtkomst i en SELinux-enforcing installation. Här finns dessutom explicita SELinux-regler för de åtta app-/systemdomänerna, inklusive `untrusted_app`, med `open`, `read`, `write` och `ioctl` på `gpu_device:chr_file`. Den tidigare ADB-kartläggningen av TV:n bekräftade live SELinux-labeln `u:object_r:gpu_device:s0` och SELinux Enforcing.

**Slutsats om direktåtkomst:** För vanliga tredjepartsappar som kör i `untrusted_app` tillåter den matchande boot-policy direkt åtkomst till `/dev/mali0` på SELinux- och DAC-nivå. Appen behöver inte först få ett privilegierat system-/SurfaceFlinger-process-ID för att policy ska tillåta device-node-öppning och ioctl. Grafiska appar kan samtidigt använda Androids EGL/GLES-stacken: firmware har `egl.cfg` med Mali-implementation och `libGLES_mali.so`. SurfaceFlinger är även tillåten till samma nod, men fungerar inte som den enda tillåtna Mali-klienten enligt policyn.

Denna slutsats gäller policy och node-konfiguration i firmwarebilden samt tidigare observerad node-label på TV:n. Den är inte en ny runtime-processobservation och innefattar ingen öppning av device-noden.

## 6. Betydelse för CVE-2023-4211

Den extraherade modulen anger Mali driver `r14p0-01rel0`, och `untrusted_app` har explicit `open` och vanlig `ioctl`-permission mot Mali-enheten. CVE-2023-4211 beskrivs som felaktig GPU-minneshantering/use-after-free som kan nås av en lokal oprivilegierad användare ([CVE-post](https://www.cve.org/CVERecord?id=CVE-2023-4211)). Policyresultatet innebär därför att SELinux inte isolerar den berörda drivrutinen från vanliga appdomäner i denna build; det höjer den praktiska exponeringen jämfört med en konfiguration där endast privilegierad grafiksystemkod har enheten.

Detta bevisar **inte** att CVE-2023-4211 kan utnyttjas på TV:n, att en viss sårbar ioctl-kodväg fungerar i denna driverbuild, eller att minneskorruption kan kontrolleras. Ingen CVE-testning, ioctl, PoC, exploit eller crash-test genomfördes. Driverbuildens sårbarhetskoppling baseras på versionsidentifieraren och den tidigare kodvägsanalysen; exploitbarheten för just denna firmware återstår utanför denna statiska policyanalys.

## 7. Bekräftade fakta, slutsatser och okända uppgifter

### Bekräftade fakta

- Firmwarefilens storlek, SHA256 och ZIP CRC matchar de angivna förväntningarna.
- Unixtract extraherade 32 namngivna komponenter från Sony-paketet med profilen “Sony 2015 Android”.
- `sysm/build.prop` och ramdiskens `selinux_version` anger exakt fingerprint `NRD91N.S34/5.433` för SVP-DTV15.
- `boot`-kernelsträng och Mali-modulens vermagic anger kernel 3.10.79; modulen anger `r14p0-01rel0`.
- Normal boot-ramdisken har kompilerad SELinux-policy v30 och konfigurationsfilerna redovisade ovan.
- Den kompilerade policyn tillåter `untrusted_app` och samtliga sju andra efterfrågade domäner att öppna/läsa/skriva/ioctl:a `gpu_device:chr_file`.
- Ingen Mali-specifik `allowxperm`-regel hittades. Separat `map`-permission finns inte definierad i denna policy.
- Boot-ramdiskens ueventd-konfiguration anger `/dev/mali0` med mode 0666 och root:root.

### Tekniska slutsatser

- Firmwarebilden är korrekt för release/build 5.433 EUB/SVP-DTV15 på buildfingerprintnivå.
- `cfig`-identifieraren med suffix `0183` är starkt stöd för den installerade subversionen `.0183`, men identifieraren är inte själv en dokumenterad human-readable versionssträng.
- Vanliga tredjepartsappar i `untrusted_app`-domänen har direkt policy- och DAC-tillåtelse till Mali-device-noden, inklusive vanlig ioctl-permission.
- Grafikstackens Mali EGL/GLES-bibliotek är närvarande; GPU-åtkomst för appgrafik behöver därför inte uteslutande gå via en privilegierad process.

### Inte verifierat / begränsningar

- Ingen officiell Sony-checksumma eller signatur verifierade TLD-spegelns ursprung.
- Det fullständiga produktversionsnamnet `PKG5.433.0183EUB` förekommer inte bokstavligen i de kontrollerade metadatafilerna; intern ID-mappningen till `.0183` är inte dokumenterad.
- Kompilerad SELinux-policy finns, men källfiler `.te`/`.cil` saknas; analysen är binärpolicyfrågor via SETools.
- `file_contexts.bin` analyserades med formatidentifiering/strängar och jämfördes med tidigare live-label, inte omvandlad tillbaka till ursprunglig regextext.
- Den aktiva runtime-policyns hash kunde inte hämtas från TV:n i tidigare ADB-kartläggning; här analyseras firmwarebildens boot-policy som matchar dess buildfingerprint.
- Ingen separat SELinux-`map`-permission finns i policyversionen. Den faktiska `mmap()`-kontrollen i kernel/driver testades inte.
- Inga runtime- eller exploitabilitytester utfördes.

## Analysverktyg och körmiljö

Unixtract byggdes lokalt från den angivna git-revisionen. SquashFS/LZO/cpio-extraktion och binärpolicyfrågor utfördes i lokal WSL Ubuntu-miljö med `unsquashfs`, `lzop`, `cpio`, SETools (`seinfo`, `sesearch`) och `file`. Analysverktygen installerades endast på analysdatorn; lokal WSL-administratörsbehörighet användes endast för verktygsinstallation. Ingen root-eskalering eller policyändring gjordes på TV:n.

## 8. Hashar för viktiga analysfiler

| Fil | SHA256 |
|---|---|
| Extraherad `.pkg` | `5bad529cbc7ef2e66d074467bc2f5d3cff09c260f3f77aeaaedcf08496d2cdd2` |
| `parts/cfig.bin` | `bc92246af58b30d56f48448af490d121a7d58c1a3a18acb5dc33469bb235c1b8` |
| `parts/minf.bin` | `6d4eb1916e0460097f8c9843723515b7d0aca268c61a3d310bbae6137a70f578` |
| `parts/boot.bin` | `03fd2f151e00c876b9501df86cc2a2839cb2f54676af6da06bbd8c45b6bcb040` |
| `parts/sysm.bin` | `85da95b36c659c8c5f50b994868c05093d7fabdae24c614b8105805c8d87498f` |
| `parts/3rdp.bin` | `4bd359b5354da5a44a9fd86dc71381047edaef65506d10f13b0fec6b46fc54c2` |
| `sysm/build.prop` | `f47be96685979a122f157d0296c45276b8f589a8be9ed5075127004b27b4a54b` |
| `sysm/lib/modules/mali_kbase.ko` | `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c` |
| `sysm/lib/egl/libGLES_mali.so` | `a6f463e9cb34591c5818e25437ca346130f113e2f85e5b5e0f90b3e96adcf835` |
| Normal boot `sepolicy` | `13da0c50cf46ebfeb250615d57803044fcd7f1bf391ba40c397fbd7f2d1223ea` |
| Recovery `sepolicy` | `b79aed911a3657d7fb27df230f917f6016cbdc59f20e0a98f62639bef6173474` |
| `file_contexts.bin` | `17ed7576639c202ad6a83f2df15a878b2b95618ab930a89554bb186fae5ee6e9` |
| `seapp_contexts` | `9753e6f26d1776ed33761e16c969a0c0eff813f2928d9c1cb838158705689785` |
| `property_contexts` | `c0ed47ff2a067dafc06086ee6a77f59f39ab14626dfae564708d364b7fd3f81d` |
| `service_contexts` | `0708ef1bb16950d392da0dd30e48b0c9fe9aed8c3b83ba606001111a1fbd90bf` |

Filerna för policy och reproducerbara SETools-frågor finns i `work/sony-5433-extracted/analysis/`. Särskilt: `analysis/boot_ramdisk/sepolicy`, `analysis/boot_ramdisk/file_contexts.bin`, `analysis/boot_ramdisk/seapp_contexts`, `analysis/mali-policy-analysis.txt` och `analysis/policy-sha256.txt`.

## 9. Rekommenderade fortsatta steg

1. Bevara firmware- och policyhasharna som analysens integritetsreferens.
2. Om formell releaseproveniens krävs, få en Sony-publicerad hash/signatur eller en oberoende kopia från en källa med verifierbar kedja.
3. För att låsa `.0183EUB`-mappningen, sök efter Sony-dokumentation för `cfig`-ID `15390183` eller en annan officiell paketmanifest/versionstabell.
4. Fortsätt med statisk analys av `mali_kbase.ko` och dess ioctl-/minneshanteringskodvägar, utan att testa dem på TV:n. Jämför den identifierade drivrutinskällan med CVE-2023-4211:s berörda kodväg.
5. Behandla `untrusted_app`→`gpu_device` som en bekräftad lokal kernelattackyta; eventuella framtida runtime-steg bör fortsätta vara icke-destruktiva och avskilda från exploitvalidering.
