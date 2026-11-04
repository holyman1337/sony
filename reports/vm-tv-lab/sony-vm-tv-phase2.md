# Sony VM-TV – Fas 2: fördjupad emulering och kompatibilitetsanalys

Datum: 2026-11-04

## Sammanfattning

Den nuvarande VM är en fungerande, generisk Android TV 7.0/API 24 x86 AVD. Den ger en stabil miljö för generisk Android-analys men kör inte Sonys userspace, kernel, HAL eller GPU-driver.

Den verifierade lokala PKG5.433-firmwarekopian innehåller en ARMv7 Android boot image, en U-Boot `uImage` med Linux 3.10.79, Sony/MediaTek systemfiler, Mali-T624 userspace-bibliotek och Mali kernelmodul. Partitionen `sysm.bin` är en SquashFS-image. Den lokala firmwarekopian kan analyseras statiskt; dess komponenter kan inte säkert läggas in i den x86 AVD:n.

En ARMv7 Linux-VM med QEMU är tekniskt möjlig som **generisk ARM-plattform**, men detta är inte Sony/MT5890-emulering. QEMU kräver en matchande boardmodell/device tree. Den lokala miljön saknar fristående `qemu-system-arm` och `qemu-system-aarch64`; Android Emulators äldre ARM-start misslyckades med `Missing emulator engine program for 'arm' CPU`. QEMU:s dokumentation beskriver `virt` som en generisk virtuell plattform, inte en riktig board, och varnar för att en kernel för en board normalt inte startar på en annan. Inga MT5890-boardmodell hittades i den tillgängliga miljön.

**Inga ändringar gjordes i den körande AVD:n, dess disk, properties, snapshot eller nätverkskonfiguration.** Den befintliga loopback-bindningen och Windows-brandväggsreglerna lämnades orörda. Ingen ADB-server eller anslutning till fysisk TV användes i denna fas.

## 1. Nuvarande arkitektur och vad som körs

| Del | Bekräftat tillstånd |
|---|---|
| AVD | `SonyVM_API24_TV_x86`, Android TV API 24 system image, x86, 2 GiB RAM, 1920×1080 konfiguration |
| Android | 7.0, API 24, generisk Google fingerprint `generic_x86/sdk_google_atv_x86/generic_x86:7.0/NYC/6696031:userdebug/test-keys` |
| Kernel | Emulatorns generiska Linux 3.10.0+, inte Sony 3.10.79 |
| Grafik | Emulatorns programvaru-rendering/SwiftShader-konfiguration, inte Mali-T624 eller Sonys Mali userspace/kernel-driver |
| SELinux | Enforcing i den generiska gästen; detta bekräftar inte Sony-policy i runtime |
| Nätverk | ADB/emulatorportar är loopback-bundna; emulator-/QEMU-programmens aktiva utgående blockregler är kvar |
| Gäststatus | Den tidigare verifieringen visade `sys.boot_completed=1`; denna fas gjorde ingen ny ADB-kontakt |

Androids dokumentation beskriver AVD som ett komplett virtuellt systemimage och medger specialbyggda AVD-systemimages, men ett generiskt AVD-image blir inte en OEM-board eller OEM-kernel genom att byta egenskaper ([AOSP: Android Virtual Devices](https://source.android.com/docs/setup/test/avd)).

## 2. Firmwareinventering

### 2.1 Partition- och bootmaterial

Lokala filer under `work/sony-5433-extracted/parts/` inkluderar:

| Fil | Storlek | Identifiering / användning |
|---|---:|---|
| `sysm.bin` | 1,339,318,272 byte | SquashFS v4, LZ4; Android systempartition, statiskt läsbar |
| `3rdp.bin` | 80,572,416 byte | Separat tredjeparts-/vendorinnehåll |
| `rfsA.bin` | 79,609,856 byte | SquashFS v4, LZO; firmware-/servicefilsystem |
| `boot.bin` | 12,481,832 byte | Android boot image med kernel och ramdisk |
| `boot.kernel` | 4,585,152 byte | U-Boot legacy uImage; Linux 3.10.79, ARM, utan komprimering i uImage-headern |
| `boot.ramdisk.raw` | 7,892,608 byte | LZOP-komprimerad ramdiskkälla |
| `recv.kernel` | 4,585,152 byte | Samma storlek som boot-kärnan; återställningskärna, inte bevisat separat board-kompatibel |
| `recv.ramdisk.raw` | 10,349,744 byte | Recovery-ramdiskmaterial |

Ingen separat DTB-fil hittades i `parts/`. Detta är en inventeringsobservation, inte bevis för att bootkedjan aldrig tillhandahåller en DTB från annan firmwarekomponent eller bootloader.

`boot.kernel` identifierades med `file` som ARM Linux 3.10.79 uImage. Kärnans strängar innehåller `MT5890`/`mt5890`, vilket stöder SoC-kopplingen, men bevisar inte ensamt vilka enheter, bootargument eller tidsförlopp kärnan förutsätter. Sony build.prop anger också `ro.board.platform=mt5890`.

### 2.2 Järjestelmäominaisuudet ja Sony-profil

Separat arkivprofil skapad från `work/sony-5433-extracted/sysm/build.prop`:

- [Sony build.prop-profil](sony-kd-55xd9305-5.433.profile.properties)

Källfilens SHA256: `F47BE96685979A122F157D0296C45276B8F589A8BE9ED5075127004B27B4A54B`\
Profilfilens SHA256 (inklusive arkivkommentar): `AE0A91DA9C8E697A37A7A54C13241298E835C211E040DFB7EB1834A8D4C2E998`

Centrala autentiska build-egenskaper i firmwarekällan:

| Property | Extraherat värde | Proveniens |
|---|---|---|
| `ro.build.id` | `NRD91N.S34` | `sysm/build.prop` |
| `ro.build.version.incremental` | `5.433` | `sysm/build.prop` |
| `ro.build.version.release` / `.sdk` | `7.0` / `24` | `sysm/build.prop` |
| `ro.build.version.security_patch` | `2019-02-01` | `sysm/build.prop` |
| `ro.build.fingerprint` | `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys` | `sysm/build.prop`; stämmer med uppgiften i projektunderlaget |
| `ro.product.model` | `BRAVIA 4K 2015` | `sysm/build.prop` |
| `ro.product.brand` / `.device` | `Sony` / `SVP-DTV15` | `sysm/build.prop` |
| `ro.product.board` / `ro.board.platform` | `SVPDTV15` / `mt5890` | `sysm/build.prop` |
| CPU ABI | `armeabi-v7a,armeabi` | `sysm/build.prop` |
| `ro.build.characteristics` | `nosdcard,tv` | `sysm/build.prop` |
| `ro.opengles.version` | `196609` | `sysm/build.prop`; propertyvärde, inte bevis på vilken GPU som finns i gästen |
| `ro.adb.secure` | `1` | `sysm/build.prop`; firmwarekonfiguration, inte runtime-verifiering här |

Värdena ovan är autentiska **för den lokala 5.433 build.prop-filen**. Fingerprint-värdet matchar uppgiften för TV:n men i denna fas lästes inte TV:n och matchningen verifierades inte på enheten. Profilfilen är endast en referensfil. Inga properties har injicerats i AVD:n. Att sätta Sony-fingerprint, `mt5890` eller TV-egenskaper på x86 skulle bara ändra rapporterad identitet och kan få appar att välja fel kodvägar; det skulle inte emulera SoC, TV-funktioner, HAL eller signeringskedja.

### 2.3 Appar, init, permission XML, SELinux och bibliotek

I den extraherade `sysm/app/` finns bland annat Sony/TV-komponenter `SonyShelf`, `SonySelect`, `Tvx`, `TvDotAction`, `TVRemoteserviceagent`, `SemcDlna`, `InteractiveTvPlatform`, `hbbtvlauncher`, `HueyModule`, `ssp_*` och MediaTek `mtktvapi_agent`/`MtkWfdService`. Dessa är **filer i firmware**, inte processer som körs i AVD:n.

Firmwarekopian innehåller Android init-fragment under `sysm/etc/init/`, bland annat standardtjänster för `surfaceflinger`, `mediaserver`, `audioserver`, `vold`, `netd` och andra Android-daemoner. Det finns även Sony- och MTK-specifika permission XML under `sysm/etc/permissions/`, till exempel `sony_features.xml`, `tv_core_hardware.xml`, `com.sony.device.xml`, `com.sony.dtv.*.xml`, `com.mediatek.*.xml` och `platform.xml`. Dessa deklarerar funktioner/permissions och ersätter inte körbara framework-tjänster eller signerade privileged-app-placeringar.

Kompilerad Sony SELinux-policy ja `file_contexts.bin` finns separat i `analysis/boot_ramdisk/`; tidigare policyanalys och hash dokumenteras i [firmware- och policyrapporten](../sony-5433-firmware-extraction-and-policy.md). Policyfiler kan analyseras statiskt men får inte kopieras till x86-gästen som om de vore kompatibla. Sony `init*.rc`, contexts och domain-transition-regler är tätt kopplade till OEM-processer och enhetsnoder.

SquashFS-listning bekräftar att själva `sysm.bin` innehåller riktiga filer som `lib/egl/libGLES_mali.so` (4,800,460 byte), `lib/hw/tv_input.mt5890.so` (46,824 byte), `lib/modules/mali_kbase.ko` (400,067 byte) och Sony APK:er. Den separata arbetskopian under `sysm/lib/` visar däremot flera 0-byte placeholder-filer. Använd därför inte dessa 0-byte poster som bibliotek/moduler; partitioninnehållet och de tidigare verifierade analysfilerna är källunderlaget.

**Kompatibilitet:** Sony-filer är ARMv7 och plattformsspecifika. `libGLES_mali.so` behöver den matchande Mali kernel-driver, EGL/GLES/HAL-kontrakt och hardware/device nodes; `tv_input.mt5890.so`, codec/media, DVB/DTV, HDMI, vendor Binder services och MediaTek JNI-bibliotek förväntar sig MT5890-enheter. Sony APK:er kan kräva Sony frameworks, privapp permissions, certifikat, services och system UID-gränser. De ska inte installeras eller kopieras in i x86 AVD:n. APK:er, manifest, resources, strings och ELF metadata kan i stället analyseras separat utan att köra dem.

## 3. ARM-emulering och den faktiska 3.10.79-kärnan

### Möjliga nivåer

1. **ARMv7 userspace under QEMU user-mode:** kan vara användbart för statisk/isolated test av vissa ARM-binärer, men emulerar inte en hel Android-TV eller kernelens GPU/driver.
2. **Generisk ARMv7 Linux-gäst på QEMU `virt`:** realistisk separat laboratorieuppgift med QEMU TCG och en kernel byggd för QEMU `virt` (ARMv7 Cortex-A15/A7). Ger Linux/ARM eller eventuellt en specialbyggd AOSP-userspace, men har QEMU:s virtio/UART/GIC-device tree, inte MT5890.
3. **Sony-kernel på MT5890-emulering:** inte realistisk med komponenter som nu finns. Kräver QEMU boardmodell för MT5890 eller annan emulator som reproducerar CPU, boot protocol, RAM-layout, interrupt controller, timers, clocks, pinmux, storage, display, DTV/media interfaces, power och vendor drivers, samt kompatibel device tree/ATAG/bootloaderdata. Inget sådant boardstöd hittades i den lokala emulatorn eller QEMU-dokumentationen.
4. **Riktig motsvarande hårdvara:** krävs för att verifiera Sony-kernelns faktiska boot- och device behavior. Denna uppgift rör inte den användarens TV; framtida hardware-labb skulle behöva en separat reserv-TV/MT5890-board med återställningsväg och separat nätverk.

QEMU upstream säger att ARM-systememulering kräver uttryckligt `-M` boardval; OS-images byggda för en maskin startar normalt inte på en annan, och om maskinen saknas i `-machine help` kommer imagen sannolikt inte att boota. QEMU `virt` är uttryckligen en virtuell generisk plattform som kräver en kernel byggd med lämpligt stöd ([QEMU ARM system emulator](https://www.qemu.org/docs/master/system/target-arm), [QEMU `virt` board](https://www.qemu.org/docs/master/system/arm/virt)).

Sony-kärnan identifierades som en U-Boot uImage med MT5890-strängar och ett Android boot-image-omslag. Det bevisar inte att den kan startas på QEMU `virt`. Avsaknad av en separat DTB i det inventerade `parts/`-trädet och avsaknad av MT5890 board/peripheral implementation är centrala hinder. Att konvertera uImage-formatet skulle inte lägga till de saknade SoC-enheterna eller device drivers.

Androids äldre emulator hade en separat “classic” ARM engine, men den installerade versionen gav `PANIC: Missing emulator engine program for 'arm' CPU`; modern QEMU2 stöder inte denna 32-bitars ARM AVD-körväg enligt Androids emulatorarkitekturdokumentation ([Android Emulator QEMU codebase notes](https://android.googlesource.com/platform/external/qemu/%2B/emu-master-dev/android/docs/ANDROID-EMULATION-LIBRARY.TXT)). QEMU upstream ARM system emulation är däremot ett separat verktyg/produkt och finns inte installerat i den lokala WSL-miljön just nu.

## 4. Testbarhetsmatris

“QEMU ARM `virt`” nedan är en möjlig framtida installation, inte en redan byggd eller verifierad gäst. “Hårdvarulabb” avser separat, matchande reservhårdvara. Inga tester som triggar CVE/kernelminnesfel är rekommenderade i någon miljö.

| Testkategori | Nuvarande Android x86 AVD | Framtida ARM QEMU `virt` | Separat matchande hårdvarulabb |
|---|---|---|---|
| Generiska Android 7/API 24 tjänster, PackageManager, permissions | **Ja**, generisk x86-policy och framework | **Möjligen**, först efter att en kompatibel ARM Android-userspace byggts; inte bekräftat | **Ja**, men kräver läsande diagnostik |
| Sony APK/manifest/resursinventering | **Statisk källa ja**, inte körning | **Statisk källa ja**; körning inte garanterad | **Ja** i motsvarande firmware |
| Sony init-, property- och permission-XML | **Statisk analys ja**, inte aktivera i AVD | **Statisk analys ja** | **Ja** runtime endast om läsbarhet finns |
| Sony SELinux policy-regler | **Offline statisk analys ja**; runtime AVD har annan policy | **Offline statisk analys ja**; QEMU policy är en annan | **Runtime-policy kan avläsas** med godkänd read-only metod |
| Generisk SELinux i Android AVD | **Ja** | **Ja, separat QEMU-policy**, om Android byggs | **Ja**, Sony-policy skiljer sig |
| Sony property-profil | **Referensfil ja**; runtime-spoofing ändrar inte hårdvara och rekommenderas inte | **Referensfil ja** | **Ja**, firmwarevärden kan jämföras |
| ARMv7 app/userland-ABI | **Nej**, x86 image | **Möjligen** med separat ARM guest/user-mode; Android API parity inte garanterad | **Ja** |
| Sony-kernelns buildidentitet (read-only) | **Statisk analys ja** | **Statisk analys ja** | **Kan verifieras med read-only metadata** |
| Sony-kernelns körbeteende / Mali-trigger | **Nej** | **Nej**; en `virt`-kernel är en annan kernel/platform | **Nej inom denna uppgift**; kerneltriggning och CVE-trigger ingår inte |
| MT5890 device nodes, HAL, DTV, codecs, boot chain | **Nej** | **Nej** | **Ja** på matchande board |
| Mali-T624 driver/GPU-beteende | **Nej**, SwiftShader/generisk GPU | **Nej**, ingen Mali-T624-modell | **Endast metadata-/icke-triggande kontroller**; kerneltriggning av CVE ligger utanför ramarna |
| CVE-2023-4211 kodvägsjämförelse | **Offline statisk analys ja** | **Offline statisk analys ja** | **Offline statisk analys ja**; praktisk trigger förbjuden i detta projekt |
| Isolerade nätverks-/auth-kontroller för generiska komponenter | **Ja**, så länge loopback/firewallregler bevaras | **Ja**, eget isolerat QEMU-nät krävs | **Ja**, bara på separat testnät |

## 5. Autentiskt respektive simulerat

**Autentiskt i lokalt underlag:** build.prop-nycklar och fingerprint för release 5.433; `sysm.bin`/`3rdp.bin`/`rfsA.bin`; boot image, ARM uImage identifierad som Linux 3.10.79; ramdisk med kompilerad SELinux-policy; Sony/MTK APK/permissions/init-konfigurationer; Mali kernelmodul och userspace-bibliotek enligt firmware-/analysunderlaget.

**Simulerat endast i AVD:** Android 7.0/API 24 generation, Android TV användargränssnitt/systemimage, 1080p display, emulatorns generiska nätverk/sensorer och x86 ABI. AVD fingerprint är `generic_x86...`; det är inte Sony.

**Inte närvarande i gästen:** Sony build.prop, Sony framework, Sony privileged apps/services, MT5890 kernel, MT5890 board, Mali-T624 GPU, `mali_kbase` runtime-driver, TV tuner/media hardware och Sony SELinux policy.

## 6. Rekommenderad nästa tekniska åtgärd

1. **Behåll befintlig x86 AVD orörd.** Använd den för generiska Android 7/API 24 och ofarliga, isolerade analyser.
2. **Fortsätt statisk analys i separat firmware-arbetskatalog.** Prioritera APK manifest/permissions, init service definitions, Sony feature XML, native ELF dependencies/ABI och SELinux-policy. Läs från `sysm.bin`/`3rdp.bin`; hantera aldrig noll-byte arbetskopior som komponenter.
3. **Om ARM runtime behövs, bygg separat QEMU `virt` prototyp från scratch** i ny katalog och med ny virtuell disk/snapshot. Använd en kernel som uttryckligen byggts för QEMU `virt`, ett eget Android/Linux userspace och separat isolerat nät. Märk den “generic ARM virt”; prova inte Sonys kernelmodul, Mali-blobbar, boot image eller Sony policy som systempolicy.
4. **Parkera MT5890/Mali runtime-emulering.** Det kräver omfattande board- och enhetsemulering; realistiskt alternativ är separat, återställningsbar matchande hårdvara. Ingen sådan hårdvara ska anslutas i denna fas.

Inga stora QEMU-/Android-ombyggnader eller installationer gjordes i denna fas. Före framtida gueständringar ska en ren snapshot/clone av respektive AVD skapas; den nuvarande fungerande VM har inte ändrats, så befintliga snapshots lämnades som de var.

## 7. Begränsningar

- Firmware build.prop är lokalt autentisk men har inte återlästs från TV under denna fas.
- QEMU-compatibilitetsbedömningen bygger på lokala binärer och upstream QEMU/AOSP-dokumentation; inget standalone QEMU ARM-system startades.
- Vi har inte byggt AOSP 7 ARM userspace för QEMU `virt`; det är en separat stor builduppgift och kräver en `virt`-kapabel kernel/system image.
- Ingen firmwarepartition, kernel, system property, AVD-disk eller nätverksregel ändrades.
- Ingen ADB-kommunikation, anslutningskontroll eller annan aktivitet riktades mot den fysiska TV:n.

## Källor

- [QEMU ARM system emulator and board selection](https://www.qemu.org/docs/master/system/target-arm)
- [QEMU generic `virt` board and its device tree](https://www.qemu.org/docs/master/system/arm/virt)
- [Android Open Source Project: Android Virtual Devices and custom system images](https://source.android.com/docs/setup/test/avd)
- [Android Emulator architecture notes: classic vs QEMU2 system-image support](https://android.googlesource.com/platform/external/qemu/%2B/emu-master-dev/android/docs/ANDROID-EMULATION-LIBRARY.TXT)
