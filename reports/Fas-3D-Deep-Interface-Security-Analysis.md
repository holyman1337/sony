# Sony KD-55XD9305 — Fas 3D: Deep Interface Security Analysis

**Analysdatum:** 2026-11-04
**Omfattning:** lokal, statisk granskning av extraherad firmware 5.433 EUB-kopia. Ingen kontakt med TV:n eller emulatorn, ingen APK-installation, ingen device-node-öppning, inget ioctl/mmap-test, ingen exploit eller fuzzning.

## 1. Sammanfattning

Fas 3D stärker tre tidigare resultat och tillför två bekräftade statiska Android-åtkomstproblem samt en indikerad Binder-granskningsyta:

1. **Mali:** `untrusted_app` har en explicit SELinux-allow-regel med `open/read/write/ioctl` på `gpu_device:chr_file`; ueventd anger `/dev/mali0` som `0666 root:root`. Detta är direkt konfigurerad app-till-kernel-åtkomst. Den tidigare starka statiska matchningen mot CVE-2023-4211 kvarstår. `kbase_ioctl` har flera minnes-, job-, sync- och profiling-operationer, men saknade UAPI-header och källkod hindrar en fullständig namn-till-kommando-mappning. Inga nya Mali-CVE:er fastställdes.
2. **ION:** `/dev/ion` är `0666 system:system`, och policyn tillåter `untrusted_app` `open/read/write/ioctl` på `ion_device:chr_file`. Den faktiska kernel-implementationen och dess ioctl-dispatch kunde inte granskas: ingen ION-modul eller kärnkällkod finns i underlaget, och den enda `libion.so`-artefakten är tom (0 byte). Ingen separat ION-sårbarhet/CVE fastställdes.
3. **Noder:** ueventd-konfigurationen anger ytterligare bred DAC-konfiguration, särskilt `/dev/sw_sync` och `/dev/block/mmcblk0p18` som `0666`. För de flesta andra noder kunde den gamla binära `file_contexts.bin` inte mappas exakt till sökväg med tillgänglig modern host-libselinux. Deras SELinux-typ och därmed effektiva appåtkomst redovisas därför med tydlig osäkerhet.
4. **Ny F-007:** den exporterade `TnfContentProvider` saknar läs- och skrivpermission. DEX-metoderna `query`, `insert`, `update` och `delete` utför SQLite-operationer mot `TNF_DATABASE.db`, med URI:er för bland annat Samba-inställningar och nyckeltabell. Android 7:s provider-ramverk tillåter standardåtkomst när en provider är exporterad och saknar motsvarande permission. Statisk åtkomstexponering är bekräftad; ingen app anropade providern och innehåll/effekt testades inte.
5. **Ny F-008:** exporterade `SemcDlna`-providern `DlnaUriProvider` har läspermission men ingen skrivpermission. Dess DEX `insert/delete/update` öppnar den skrivbara SQLite-databasen och utför CRUD mot tabellen `datasource`. Statisk oreglerad skrivyta är bekräftad enligt manifest och Android 7-ramverk; faktisk data-/systempåverkan är inte testad.
6. **Ny F-009:** flera Sony Binder-tjänster är uttryckligen exporterade utan manifestpermission och returnerar Binder-stubbar från `onBind`. Detta är bekräftad tillgänglig IPC-yta men en sårbarhet är inte fastställd; fullständig transaktionssemantik och alla indirekta caller-checks återstår.
7. **Sony WebAPI:** `WebApi.xml` deklarerar `/sony/broadcastLink`, men den angivna JAR-filen saknas i extraherad kopia. Listener, transport, autentisering och nätverksreachability är okända.

Inga resultat från generisk emulator används för att dra slutsatser om Sony-hårdvaran. Firmwarekopian matchar build `NRD91N.S34/5.433` enligt tidigare metadata, men dess exakta kryptografiska koppling till subrevision `.0183EUB` är fortfarande inte verifierad.

## 2. Metod, omfattning och identitetsbegränsning

Analysunderlaget är den lokala extraktionen `work/sony-5433-extracted/`. Den befintliga rapporten anger systembilden `sysm.bin`, bootbilden `boot.bin`, kernelimage `parts/boot.kernel`, normal boot-policy och Mali-modul med följande hashar:

| Artefakt | SHA256 |
|---|---|
| `analysis/boot_ramdisk/sepolicy` | `13da0c50cf46ebfeb250615d57803044fcd7f1bf391ba40c397fbd7f2d1223ea` |
| `analysis/boot_ramdisk/file_contexts.bin` | `17ed7576639c202ad6a83f2df15a878b2b95618ab930a89554bb186fae5ee6e9` |
| `analysis/boot_ramdisk/ueventd.mt5890.rc` | `3636ba7bec62d1ccbacd166ebba253b9facca2bba2f5f2805b4e993e2bc01e53` |
| `analysis/system/mali_kbase.ko` | `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c` |
| `sysm/etc/webapi/WebApi.xml` | `045ea24778845850d4f874f8c314cad46127face61946991b62b29c932405fb4` |
| `sysm/app/InteractiveTvPlatform/InteractiveTvPlatform.apk` | `d6af6e3cb67c891b032be800c22c396445e7ab523a052af180117d99129ff96a` |
| `sysm/app/InteractiveTvUtil/InteractiveTvUtil.apk` | `b615668ab1838d5fd290f0ebe8fc6146f99b59fa5b931b667bd9f18cc57931cc` |
| `sysm/app/SemcDlna/SemcDlna.apk` | `d7cc3633c5149a583360537afd032f5cdb32040a7b6af90da2d8a41fbd5e3450` |

Metoder/verktyg: SETools `seinfo`/`sesearch` mot kompilerad normal boot-policy, binär DEX-manifest-/metodgranskning med lokalt installerade Androguard 4.1.4, `arm-linux-gnueabi-objdump` på Mali-modulen samt text-/filinventering. Verktygspaketet placerades utanför Git-kopian under lokal `work/analysis-tools/`; firmwarefilerna ändrades inte.

Policystatistik från `analysis/mali-policy-analysis.txt`: policyversion 30, 828 typer, 8 284 allow-regler, 111 allowxperm-regler, 0 permissiva typer och okända klasser nekas. Textkällor `.te`/`.cil` saknas.

## 3. Device-node-inventering

### 3.1 DAC-konfiguration

Normal boot-konfiguration `analysis/boot_ramdisk/ueventd.mt5890.rc` har explicita regler på rader 1–14. Recovery-konfigurationen har samma innehåll. Detta är init/ueventd-konfiguration, inte runtime-bevis för att varje nod existerar vid varje boot.

| Nod/sökvägsmönster | DAC mode | Ägare:grupp | Funktion/underlag | SELinux-typstatus |
|---|---:|---|---|---|
| `/dev/hidraw*` | `0660` | `system:audio` | HID rågränssnitt | `input_device` är indikativt från kompilerade kontextsträngar; exakt sökvägsmappning ej avkodad |
| `/dev/video*` | ueventd `0660`; `dtv_svc.sh` anger senare `0660` | ueventd `system:camera`; skript `root:1006` (kommenterad `root:camera`) | V4L/video-gränssnitt | `video_device` indikativt; exakt sökvägsmappning ej avkodad |
| `/dev/mali0` | `0666` | `root:root` | Mali GPU-kärndrivrutin | `gpu_device` bekräftad av tidigare läsande runtime-label och finns i firmwarepolicy |
| `/dev/sw_sync` | `0666` | `root:root` | Sync-fence-gränssnitt | `gpu_device` är kandidat från kontextartefaktens strängar; inte verifierat som exakt path-context |
| `/dev/ion` | `0666` | `system:system` | Android ION-minnesgränssnitt | `ion_device` indicerat av policyregler och path/type-strängar; exakt context-lookup ej möjlig |
| `/dev/trusty-ipc-dev0` | `0660` | `system:keystore` | Trusty/TEE IPC | `tee_device` kandidat; path-context ej exakt avkodad |
| `/dev/stpbt` | `0660` | `bluetooth:net_bt_stack` | MediaTek Bluetooth-kontrollgränssnitt | Exakt typ okänd |
| `/dev/block/mmcblk0rpmb` | `0660` | `media:mediadrm` | RPMB/blockgränssnitt | `block_device` kandidat; path-context ej exakt avkodad |
| `/dev/block/mmcblk0p3` | `0660` | `root:system` | Partition; funktion ej fastställd här | `block_device` kandidat; path-context ej exakt avkodad |
| `/dev/block/mmcblk0p18` | `0666` | `root:root` | Partition; innehåll/funktion ej fastställd | `block_device` kandidat; path-context ej exakt avkodad |
| `/dev/ttyUSB0` | `0660` | `system:irblaster` | IR-blaster seriellt gränssnitt | `tty_device` kandidat; path-context ej exakt avkodad |
| `/dev/ttyUSB1` | `0660` | `system:irblaster` | IR-blaster seriellt gränssnitt | `tty_device` kandidat; path-context ej exakt avkodad |
| `/dev/ttyUSB2` | `0660` | `system:irblaster` | IR-blaster seriellt gränssnitt | `tty_device` kandidat; path-context ej exakt avkodad |

### 3.1.1 Senare skriptmässiga ändringar och separat RFS-miljö

Inventeringen av nodkonfigurationen kompletterades med två explicita skriptfynd:

- `sysm/bin/dtv_svc.sh:27-28` kör `chmod 660 /dev/video*` och `chown 0:1006 /dev/video*`. Skriptkommentarerna vid raderna 24–25 anger `0:1006` som `root.camera`. Detta är en avsedd ändring efter ueventd:s `system:camera 0660`; faktisk körordning och live-ägarskap har inte verifierats. Den effektiva statiska slutkonfigurationen som skriptet begär är således `root:camera 0660`, om kommandona körs och matchar noder.
- `rfsA/etc/scripts/mkpts.sh:77-79` skapar `/dev/pts`, skapar `/dev/ptmx` som character device `5:2` med mode `0666` och monterar därefter `devpts`. Detta är en RFS-scriptad miljö och ska inte blandas ihop med Androids ueventd-konfiguration. `rfsA/etc/udev/udev.rules:56` anger också mode `0666` för `ptmx`.

Den kompilerade `file_contexts.bin` innehåller en närliggande post med `u:object_r:ptmx_device:s0` och `/dev/ptmx` (rå strängpost). Vi kunde inte använda host-libselinux för att göra en fullständig lookup av Android 7-formatet. `ptmx_device`-typen har därför inte lagts till i domänmatrisen och ingen appdomäns åtkomst till denna RFS-nod påstås. Det behövs en formatkompatibel parser samt klarhet i hur RFS-noden exponeras (om alls) till Android-processer.

I `sysm/etc/permissions/platform.xml` mappas `com.sony.dtv.permission.IRBLASTER` till gruppen `irblaster` (rad 137–139), och `MANAGE_VOICE_KEYPHRASES` mappas till gruppen `audio` (rad 105–107). Det visar gruppkopplingar, inte att en godtycklig app beviljas dessa permissions. Ueventd:s `0666`-noder kringgår gruppvillkoret på DAC-nivån, men inte SELinux.

### 3.2 SELinux-matris per device-typ

Tabellen visar tillämpliga explicita allow-regler i den kompilerade boot-policyn. För en processdomän som saknar allow för den faktiska nodens typ innebär SELinux Enforcing normalt nekad åtkomst. För alla noder vars exakta label inte har verifierats gäller slutsatsen endast den angivna kandidattypen.

| Device-typ / nodkandidater | `untrusted_app` | `platform_app` | `priv_app` | `system_app` | `sony_platform_app` | `surfaceflinger` | `system_server` | `shell` | Andra relevanta allow-domäner |
|---|---|---|---|---|---|---|---|---|---|
| `gpu_device:chr_file` — `/dev/mali0`; ev. `/dev/sw_sync` | `open read write ioctl getattr` | samma | samma | samma | samma | samma | samma | samma | `adbd`, `autoplay_app`, `bluetooth`, `bootanim`, `dumpstate`, `mediacodec`, `mediaserver`,
fc`, `radio`, `shared_relro`, `vold` (vissa med smalare rättigheter); `debuggerd` har inte write; `mt_dtv-svc` bara getattr |
| `ion_device:chr_file` — `/dev/ion` | `open read write ioctl getattr` | samma | samma | samma | samma | Ingen allow | Ingen allow | `open read write ioctl getattr` | `adbd`, `bluetooth`, `bootanim`, `cameraserver`, `mediacodec`, `mediadrmserver`, `mt_dtv-svc`,
fc`, `radio`, `shared_relro`, m.fl.; vissa har smalare rättigheter |
| `video_device:chr_file` — `/dev/video*` | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | `open read write ioctl getattr` | `open read write ioctl getattr` | Ingen allow | `cameraserver`, `mediacodec`, `mediaserver`, `camera_recognition`, `mt_dtv-svc` |
| `input_device:chr_file` — `/dev/hidraw*` kandidat | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | `open read write ioctl getattr` | `open read write ioctl getattr` | `inputflinger`, `sony_input_manager`, `sony_mainunitkey_manager`, `healthd`, `kyil`, `slideshow` |
| `tee_device:chr_file` — Trusty/STPBT kandidater | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | `keystore`, `gatekeeperd`, `drmserver`, `mediadrmserver`, `tee`, `vold`, `surfaceflinger` |
| `tty_device:chr_file` — `/dev/ttyUSB*` kandidat | Ingen allow | Ingen allow | Ingen allow | Ingen allow | `open read write ioctl getattr` | Ingen allow | `open read write ioctl getattr` | `open read write ioctl getattr` | `rild`, `mt_dtv-svc`, `healthd`, `slideshow` |
| `ptmx_device:chr_file` — `/dev/ptmx` i RFS | Ej fastställt | Ej fastställt | Ej fastställt | Ej fastställt | Ej fastställt | Ej fastställt | Ej fastställt | Ej fastställt | Ingen domänquery utförd för denna typ; dessutom RFS-nodens relation till Android-processer är okänd |
| `block_device:blk_file` — `/dev/block/*` kandidat | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | Ingen allow | `uncrypt` har open/write, `kernel` ioctl/open/read, `vold` relabelfrom, `dumpstate` getattr, `mt_dtv-svc` getattr |

På `gpu_device`-regeln är den fulla allow-mängden för de åtta domänerna ovan `{ append getattr ioctl lock open read write }`. `ion_device` har samma fulla mängd för `untrusted_app`, `platform_app`, `priv_app`, `system_app`, `sony_platform_app` och `shell`. För dessa två typer returnerade `sesearch --allowxperm` ingen ioctl-nummerbegränsning. Det betyder att policyn inte snävar in tillåtna ioctl-nummer med `allowxperm`; det bevisar inte att varje kommando accepteras av drivrutinen.

Denna policyversion har ingen separat `map`-permission i de relevanta filklasserna. Det går därför inte att beskriva `mmap()` som uttryckligen tillåten eller nekad utifrån en permission med namnet `map`. `open`/`ioctl`-allow är däremot tydliga. Driver-/kernelbeteende för mmap har inte testats.

### 3.3 Begränsning i path-context-analys

`file_contexts.bin` innehåller strängar för Mali, ION, sw_sync, block, video, hidraw, Trusty, STPBT och tty-noder, men den medföljande textkällan saknas. Hostens libselinux `selabel_open` avvisade Android 7-filen med `Old compiled fcontext format, skipping`; därför kunde sökväg→typ inte lösas säkert för alla noder. Förutom `/dev/mali0` (även tidigare observerad direkt på TV) ska labels i tabellerna betraktas som kandidater/indikationer, inte verifierade exakt-mappningar.

**DAC/SELinux-skillnad:** `0666` på Mali, ION, sw_sync och partition p18 tar bort DAC-ägare/gruppsbarriären. En app måste fortfarande passera SELinux. För Mali och ION är det inte en barriär för `untrusted_app` i den analyserade policyn; för kandidattypen `block_device` saknar `untrusted_app` allow-regel, så p18:s DAC-läge ensamt bevisar inte appåtkomst.

## 4. ION-gränssnitt

### Bekräftat

- `analysis/boot_ramdisk/ueventd.mt5890.rc:5` konfigurerar `/dev/ion` `0666 system system`.
- Policyn innehåller `allow untrusted_app ion_device:chr_file { append getattr ioctl lock open read write };` och motsvarande allow för `platform_app`, `priv_app`, `system_app`, `sony_platform_app` och `shell`.
- `sysm/lib/modules/` innehåller ingen `ion.ko`; den uppräknade modullistan innehåller bland annat `mali_kbase.ko`, men ingen ION-modul.
- `sysm/lib/libion.so` finns som nollbytesfil (SHA256 `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`) och kan inte stödja API-analys.

### Ej fastställt

Ingen ION-kärnkällkod, `vmlinux`, `System.map`, kernelconfig eller separat ION-modul finns i materialet. `parts/boot.kernel` är ett kernelimage utan användbara symboler i denna analys; en rå sökning efter textsträngar är inte tillräckligt för att utesluta inbyggd ION-kod i komprimerad kernelpayload. Därför kunde följande inte fastställas: ioctl-ID:n/dispatch, heap-ID:n och flaggor, allokerings-/import-/exportkontroller, storleksgränser, user-pointer-hantering, handle-/fd-livscykel, referensräkning, cleanup eller CVE-/patchmatchning.

**Slutsats:** exponeringen till kärngränssnittet är bekräftad statiskt; en ION-implementationssårbarhet är inte fastställd. Nästa nödvändiga underlag är exakt MTK/Sony-kärnkällkod eller motsvarande obfuskeringsfri symboliserad kernel med config och vendorpatchar.

## 5. Mali-T624 kompletterande analys

### 5.1 `kbase_ioctl`-dispatch

ARM ELF-modulen `analysis/system/mali_kbase.ko` har SHA256 `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c`, driveridentifierare `r14p0-01rel0` och vermagic `3.10.79 SMP preempt mod_unload ARMv7`.

`kbase_ioctl` finns vid ELF-adress `0x177b8`, storlek `0xbb8`. Disassembly (`arm-linux-gnueabi-objdump -dr --disassemble=kbase_ioctl`) visar en separat specialförfrågan `0x480b`, hantering av ioctl-nummer i familjen `0x200`–`0x228`, kopiering från användarrymd via `__copy_from_user` och kopiering ut via `__copy_to_user`. Dispatchen anropar bland annat följande funktioner (call-site-adresser i `.text`):

| Call site | Mål | Funktionell kategori utifrån symbolnamn |
|---:|---|---|
| `0x17ad0` | `kbase_region_tracker_init_jit` | JIT/region tracker initiering |
| `0x17b20` | `kbase_mem_alloc` | GPU-minnesallokering |
| `0x17b60` | `kbase_mem_import` | import av minnesobjekt |
| `0x17c44` | `kbase_mem_alias` | aliasering av minnesregion |
| `0x17cc8` | `kbase_mem_commit` | commit/reservering av minne |
| `0x17d4c` | `kbase_mem_query` | minnesfråga |
| `0x17db0` | `kbase_mem_flags_change` | ändring av minnesflaggor |
| `0x17e04` | `kbase_mem_free` | frigöring |
| `0x17e38` | `kbase_jd_submit` | job submission |
| `0x17e7c` | `kbase_sync_now` | sync |
| `0x17ed0`/`0x17ef4`/`0x17f14` | `kbase_vinstr_*` | hardware-counter/profiling |
| `0x17f6c` | `kbase_gpuprops_uk_get_props` | GPU-egenskaper |
| `0x18074` | `kbase_stream_create` | stream creation |
| `0x18088` | `kbase_fence_validate` | fencekontroll |
| `0x180f8` | `_mali_profiling_control` | profiling control |
| `0x18198` | `kbasep_mem_profile_debugfs_insert` | memory profiling |
| `0x181cc`/`0x181fc`/`0x18228` | `kbase_tlstream_*` | timeline-stream-gränssnitt |
| `0x18274` | `kbase_soft_event_update` | soft-event uppdatering |

Dispatchen kontrollerar argument-/payloadstorlek och använder user-copy-funktioner, men dessa generella kontroller utesluter inte fel i enskilda handlers. Eftersom matchande UAPI-header saknas har ioctl-numeriska index inte kopplats till namngivna publika `KBASE_IOCTL_*`-kommandon. Inga specifika handlerfel eller nya CVE:er påstås.

### 5.2 VMA, referensräkning och CVE-2023-4211

Tidigare disassembly-evidens kvarstår: `kbase_mmap` `0x13338`, `kbase_special_vm_close` `0x113d4` och `kbasep_os_process_page_usage_update` `0x14310` följer den gamla tracking-VMA/process-mm-vägen. Den matchade Google-fixens `process_mm`-referenslivscykel och exakt-en-sida-kontroll saknas där. En separat `__mmdrop`-plats i en annan cleanup-väg räknas inte som fix.

Den statiska kodmatchningen och versionen är förenliga med Project Zeros RCA för CVE-2023-4211 och Googles publicerade patch. Det är ingen exploitability- eller runtime-verifiering. Läs [Project Zero RCA](https://googleprojectzero.github.io/0days-in-the-wild/0day-RCAs/2023/CVE-2023-4211.html) och [AOSP Mali-patchen](https://android.googlesource.com/kernel/google-modules/gpu/+/35feb9b795cf1cd0d9a0a2edb6ade3c83040f48b%5E%21/). Andra Mali-säkerhetsfixar matchades inte fullständigt mot denna binär; versionsnummer ensamt används inte som CVE-bevis.

## 6. Sony WebAPI / broadcastLink

`sysm/etc/webapi/WebApi.xml` rader 8–10 deklarerar:

- route/service `/sony/broadcastLink`
- classpath `/system/framework/com.sony.dtv.webapi.service.broadcastlink.jar`
- servlet `com.sony.dtv.webapi.service.broadcastlink.BroadcastLinkServlet`

Den deklarerade JAR-filen saknas i den extraherade `sysm/framework/`-katalogen, och ingen fil med namnet `*broadcastlink*` hittades i extraktionsmaterialet. I den lokala `sysm`, `3rdp` och konfigurationssökningen återfanns inget servletimplementationsunderlag. `WebApi.xml` har ingen deklarerad shared library eller transport-/bindningsuppgift. Därför kan följande inte verifieras: att endpointen laddas, faktisk HTTP/socket-bindning, lokale kontra nätverksåtkomst, autentisering, caller-checks, inputvalidering eller endpointens påverkan. F-004 förblir en bekräftad deklaration och en obekräftad exponerings-/sårbarhetshypotes.

## 7. APK-, export- och Binder-granskning

Granskningen var riktad, inte en genomgång av alla 108 kataloger under `sysm/app`. APK-manifesten och valda DEX-klasser lästes statiskt; inga APK:er ändrades eller kördes. `exported=true` utan permission är en komponentgräns som andra appar kan nå. För providers kontrollerades även om deras metoder faktiskt utför CRUD.

### 7.1 F-007 — TNF content provider

**APK:** `sysm/app/InteractiveTvPlatform/InteractiveTvPlatform.apk`, SHA256 ovan; package `com.sony.dtv.interactivetvplatform`, target SDK 24.

Manifestdeklarationen för `com.sony.dtv.interactivetvplatform.discovery.provider.TnfContentProvider` har `android:exported="true"`, authority `com.sony.dtv.interactivetvplatform.discovery.provider` och inga `android:permission`, `android:readPermission` eller `android:writePermission`-attribut. Ingen path-permission deklaration finns i manifestet.

DEX-evidens (`classes.dex`-code offsets):

| Metod | Offset | Statiskt beteende |
|---|---:|---|
| `<clinit>` | `0x9eec0` | URI matcher kopplar auktoriteten till Samba/TNF-tabeller |
| `getEntry` | `0x9efa8` | URI:er mappar till tabellerna `samba_supported_countries`, `cdn_version_number`, `samba_enabled`, `samba_port_number`, `samba_keys`, `samba_notify`, `current_iso_country` |
| `onCreate` | `0x9f270` | öppnar `TNF_DATABASE.db` via `TnfDatabaseHelper` |
| `delete` | `0x9f090` | anropar `SQLiteDatabase.delete` |
| `insert` | `0x9f1cc` | anropar `SQLiteDatabase.insert` |
| `query` | `0x9f2b4` | bygger fråga med `SQLiteQueryBuilder` och `SQLiteDatabase` |
| `update` | `0x9f370` | anropar `SQLiteDatabase.update` |

De granskade CRUD-metoderna saknar direkta DEX-anrop till `checkCallingPermission`, `enforceCallingPermission`, `getCallingUid` eller motsvarande caller-check API. Android 7.0:s `ContentProvider.java` (tag `android-7.0.0_r33`) tillåter default read/write när providern är exporterad och ingen provider- eller path-permission är satt: read-vägen vid rader 553–591 och write-vägen vid 630–668. Se [AOSP Android 7.0 ContentProvider](https://android.googlesource.com/platform/frameworks/base/+/android-7.0.0_r33/core/java/android/content/ContentProvider.java) och [Android provider-manifestdokumentation](https://developer.android.com/guide/topics/manifest/provider-element).

**Bedömning:** bekräftat statiskt att vanliga appar når ett oreglerat read/write-providergränssnitt enligt manifest och Android 7-ramverk. CRUD-metoderna begränsar URI till kända tabeller, men providerinnehåll, inverkan av modifierade värden och om andra tjänster använder dessa data granskades inte. Ingen app gjorde ett anrop. F-007 är ingen CVE och är inte en verifierad runtime-konsekvens.

### 7.2 F-008 — DLNA `DlnaUriProvider` oreglerad skrivväg

**APK:** `sysm/app/SemcDlna/SemcDlna.apk`, SHA256 ovan; package `com.sonyericsson.dlna`, target SDK 19, `sharedUserId="com.sonyericsson.dlna"`.

Providern `com.sonyericsson.dlna.datasource.provider.DlnaUriProvider` är exporterad, authority `com.sonyericsson.dlna.datasource.provider`, med `android:readPermission="com.sonymobile.dlna.permission.READ_DATA"`, men utan `android:writePermission`, övergripande `android:permission` eller path-permission. AOSP Android 7 write-check använder separat write-permission; när den saknas och providern är exporterad tillåter standardvägen skrivning.

DEX-evidens: `delete` `0x8c7fc` öppnar `DlnaUriDatabaseHelper.getWritableDatabase()` och anropar `SQLiteDatabase.delete` på `datasource`; `insert` `0x8c9a0` anropar SQLite insert; `update` `0x8cca8` anropar SQLite update. `query` finns vid `0x8cacc`. Ingen av dessa provider-metoder innehåller direkt caller-identity/permission-check enligt sökningen. Granskningen bevisar inte att data används av privilegierade tjänster eller att modifiering ger säkerhetskonsekvens; den oreglerade DB-skrivytan är dock statiskt bekräftad.

`DlnaDmrProvider` är också exporterad men har läspermission `com.sonymobile.dlna.permission.READ_DATA`; dess `insert/delete/update`-metoder returnerar direkt
ull`/`0` i granskad DEX, så ingen skrivexponering tillskrivs den. `DlnaContentProvider` har en generell permission. Detaljerna ändrar inte F-008.

### 7.3 F-009 — ytterligare exporterade Binder-tjänster

Följande manifestkomponenter i `InteractiveTvPlatform.apk` är uttryckligen exporterade utan manifestpermission:

| Komponent | Intent action / DEX-evidens | Notering |
|---|---|---|
| `...plugins.modules.videoshmserv.VideoShmServ` | `com.sony.dtv.interactivetvplatform.discovery.plugin.video.connect`; `onBind` DEX `0x9e738` | Binder-gränssnitt returneras; metodernas fulla caller-auth/effekt behöver följas |
| `...plugins.modules.audioshmserv.AudioShmServ` | `...plugin.audio.connect`; `onBind` `0x9b554` | returnerar Binder-stub för matchande action |
| `...plugins.modules.mtkvideonative.NativeSystemsManager` | `...filesystem.connect`; `onBind` `0x9dc08` | returnerar `IFileSystemManager`-stub; även en native Binder används internt |
| `...tvaction.Kuyun`, `...Kantar`, `...ShopTv` | `com.sony.dtv.action.extension` | exporterade tjänster; effekt ej analyserad vidare |

I `InteractiveTvUtil.apk` är `ApplicationLogCollectorService`, `ActionLogCollectorService` och `PrivacyPolicyService` också exporterade utan manifestpermission; deras `onBind`-metoder (DEX `0x1dc288`, `0x1dbe0c`, `0x15ae80`) returnerar Binder-stubbar. Inga `checkCallingPermission`/`enforceCallingPermission`/`getCallingUid`-anrop hittades i de granskade service-entrypoint-metoderna. Detta är ett riktat sökresultat, inte bevis för att alla Binder-transaktioner saknar kontroll; gränssnittsstubbar, delegerade metoder och kontextkontroller behöver följas vidare.

I `SemcDlna.apk` har `NetworkReceiver` explicit `exported=true` utan permission och flera nätverksstatus-actions; faktisk möjlighet att påverka systemtillstånd via dessa broadcasts är inte bedömd. `RemoteControlSettings.apk` använder `sharedUserId="android.uid.system"` och har en activity med intent filters utan manifestpermission; aktivitetens intentvalidering och signaturförutsättningar behöver granskas separat. Dessa två observationer är granskningsprioriteringar, inte nya sårbarhetspåståenden.

**Bedömning:** exporterad Binder-yta bekräftad statiskt; exploaterbar brist/caller-check saknas som verifierat påstående. F-009 klassas indikerat och bör hållas separat från F-007/F-008:s bekräftade provideråtkomst.

## 8. Fyndstatus och CVE-korrelation

| ID | Fas 3D-status | Evidensklass |
|---|---|---|
| F-001 Mali node/policy | Oförändrat; direct `untrusted_app` access i DAC + policy bekräftad statiskt | Bekräftat statiskt; tidigare runtime label för Mali |
| F-002 ION | Oförändrat; node/policy access bekräftad, driver implementation okänd | Bekräftad exponering, ingen ION-sårbarhet fastställd |
| F-003 CVE-2023-4211 | `kbase_ioctl` dispatch/handlers kartlagda delvis; tidigare fixgap kvarstår | Stark statisk CVE-matchning, ej exploitverifierad |
| F-004 broadcastLink | Deklaration bekräftad; angiven JAR saknas | Obekräftad faktisk reachability/auth |
| F-005 broad node DAC | ueventd-listan fullständig för explicit regler; labels/app access övriga än Mali delvis okända | Konfigurationsrader bekräftade; effektiv appåtkomst delvis okänd |
| F-006 kernel patch/config | Ingen ny kernelkälla/config hittad | Kunskapslucka; ingen CVE på versionsnummer ensamt |
| **F-007** TNF provider | exporterad, utan read/write permission, CRUD mot TNF/Samba DB | Bekräftad statisk access-control-brist; runtime-effekt ej testad |
| **F-008** DLNA URI provider | exporterad med readPermission men utan writePermission; CRUD skriver datasource-DB | Bekräftad statisk write-exponering; runtime-effekt ej testad |
| **F-009** Sony Binder services | exporterade utan manifestpermission, Binder-stubbar returneras | Indikerad IPC-granskningsyta; sårbarhet ej fastställd |

Ingen ny CVE tilldelas F-007–F-009. Ingen ytterligare Mali- eller ION-CVE matchades med patch-/binärevidens i denna fas.

## 9. Begränsningar och kunskapsluckor

- Firmwarekopian matchar build `NRD91N.S34/5.433`; exakt kryptografisk Sony-proveniens och `.0183EUB`-subrevision är fortsatt ej verifierad.
- `file_contexts.bin` är Android 7-format som host-libselinux i analysmiljön inte kunde avkoda. Noder utöver `/dev/mali0` har därför inte alla verifierade exakta labels.
- Ingen ION-kärnkällkod, symboliserad kernel, kernelconfig eller ION-modul finns. Ingen ioctl-dispatch/CVE-patchjämförelse för ION är möjlig med detta underlag.
- `kbase_ioctl` kan disassembleras men UAPI-header saknas; numeriska kommandon är inte säkert mappade till UAPI-namn/argumentstrukturer. Ingen fullständig review av varje handler gjordes.
- WebAPI-servletens deklarerade JAR saknas. Autentisering, nätverksbindning och faktisk endpoint-reachability är okända.
- APK-granskningen omfattade utvalda Sony/MediaTek-appar, inte alla appar/tjänster, och DEX-anrop spårades inte genom varje transitive hjälpare eller Binder-transaction till slutlig effekt.
- Inga anrop mot provider/service, nätverkstester eller device-node-operationer utfördes. Alla komponent-/providerresultat är statiska.
- Inga emulatorresultat eller tidigare TV-processlistor har blandats in som bevis för firmwareinnehållet.

## 10. Prioriterade rekommendationer för Fas 3E

1. **F-007/F-008:** granska med Sony produktägare om cross-app läs/skriv är avsiktlig. Föreslagen åtgärdsriktning är `exported=false` där möjligt eller separata signature-level read/write-permissions; lägg till path-permissions bara där datamodellen kräver selektiv access. Validera också URI, projection, selection och values statiskt.
2. **F-009:** bygg full statisk Binder-kedja från `onBind` till varje Stub-transaktion och implementation; verifiera `Binder.getCallingUid`/permission checks, parametergränser och om native shared-memory/filsystem API kan påverkas.
3. Hämta matchande Sony/MediaTek Linux 3.10.79-källkod/config/patchserie och kontrollsumma för exakt 5.433 build. Analysera ION och block-device allow-contexts därifrån.
4. Skaffa Android 7-kompatibel SELinux context decoder eller ursprunglig `.fc`-källa så att sökväg→SELinux-type-tabellen kan göras exakt.
5. Återfinn broadcastLink JAR från den verifierade partition/builden eller dokumentera extern leveransmekanism innan nätverksreachability/auth bedöms.
6. Behåll framtida validering statisk eller i en separat återställningsbar labbmiljö; ingen CVE-trigger eller riskfylld TV-testning ingick i denna fas.

## 11. Klassificeringsnyckel

- **Bekräftat statiskt:** direkt konfigurations-, policy-, manifest- eller binärkodsevidens styrker det angivna påståendet.
- **Indikerat:** evidens visar granskningsyta eller sannolik typ/åtkomst, men en länk i kedjan är okänd.
- **Obekräftat:** tillräcklig evidens saknas.
- **Verifierat på fysisk enhet:** kräver separat, säker runtime-observation. Denna fas gjorde inga sådana observationer.
