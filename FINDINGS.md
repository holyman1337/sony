# Fyndregister — Sony KD-55XD9305

Detta register behåller de ursprungliga fynd-ID:na F-001–F-004 från repositoryts första version. Ursprungliga preliminära formuleringar är sammanfattade i varje fynds historik; senare statisk evidens uppdaterar den aktuella statusen utan att göra tidigare hypoteser till historiska fakta. F-005 och F-006 tillkom vid Fas 3C.

## Statusdefinitioner

- **Obekräftat:** saknar direkt evidens.
- **Indikerat:** statiskt stöd finns men åtkomst, påverkan eller versionsträff är ofullständig.
- **Bekräftat statiskt:** firmwarekonfiguration/binärkod och analysunderlag stöder det specifika påståendet; ingen runtime/exploit följer automatiskt.
- **Verifierat på enheten:** observerat på fysisk Sony-enhet med säker metod.
- **Avfärdat:** evidens motsäger hypotesen.

## F-001 — Mali-enhetsnodens åtkomst

- **Komponent:** `/dev/mali0`, `gpu_device:chr_file`, SELinux-policy ur normal boot-ramdisk.
- **Beskrivning:** DAC-konfigurationen är `0666 root:root`; policy innehåller `allow untrusted_app gpu_device:chr_file { append getattr ioctl lock open read write };`. Det gör åtkomst tillåten i policy/DAC-lagren för processer i denna domän. Det bevisar inte att ett appanrop faktiskt utfördes.
- **CVE:** Ingen separat från F-003.
- **Klassificering:** Bekräftad statisk direktåtkomstkonfiguration; säkerhetsrelevant attackyta.
- **Aktuell bevisstatus:** Bekräftad i lokala firmwareartefakter, med tidigare runtime-observation av nodens DAC/label och SELinux Enforcing. Ingen Mali-öppning eller ioctl utfördes.
- **Evidens:** `reports/sony-5433-firmware-extraction-and-policy.md`, `reports/Fas-3C-Static-Firmware-Analysis.md`; lokalt `work/sony-5433-extracted/analysis/mali-policy-analysis.txt`, `analysis/boot_ramdisk/sepolicy` SHA256 `13da0c50cf46ebfeb250615d57803044fcd7f1bf391ba40c397fbd7f2d1223ea`, och `analysis/boot_ramdisk/ueventd.mt5890.rc`.
- **Historik:** Ursprunglig repo-post F-001 var *Obekräftat* eftersom kod-/policykällan då saknades. Fas 3C uppdaterade den till bekräftad statisk allow-regel. Tidiga rapporter om okänd policy är arkiverade med historisk/superseded-notering.
- **Senast uppdaterad:** 2026-11-04.

### Fas 3D-tillägg

SETools-fråga bekräftar samma allow-regel i kompilerad normal boot-policy. Full device-node-inventering återfinns i `reports/Fas-3D-Deep-Interface-Security-Analysis.md`. Den binära `file_contexts.bin` kunde inte path-dekodas av tillgänglig host-libselinux; `/dev/mali0` behåller exakt labelbevis från den tidigare runtime-observationen. Ingen Mali-operation testades.

## F-002 — ION-enheter och minnesgränssnitt

- **Komponent:** `/dev/ion`, `ion_device:chr_file`.
- **Beskrivning:** ueventd anger `/dev/ion 0666 system system`; SELinux innehåller `allow untrusted_app ion_device:chr_file { append getattr ioctl lock open read write };`.
- **CVE:** Ingen matchad.
- **Klassificering:** Bekräftad statisk exponering av kernelgränssnitt; sårbarhet ej fastställd.
- **Aktuell bevisstatus:** Policy- och DAC-konfiguration bekräftade i lokal kopia. ION-implementation, ioctl-tabell och runtime-resultat är inte granskade.
- **Evidens:** `reports/Fas-3C-Static-Firmware-Analysis.md`; lokalt `work/sony-5433-extracted/analysis/mali-policy-analysis.txt`, `analysis/boot_ramdisk/ueventd.mt5890.rc` och kompilerad `analysis/boot_ramdisk/sepolicy`.
- **Historik:** Ursprunglig repo-post F-002 var preliminär och obekräftad. Fas 3C gav konfigurations- och policybevis men ingen sårbarhets/CVE-träff.
- **Senast uppdaterad:** 2026-11-04.

### Fas 3D-tillägg

ION-rättighetsregeln verifierades om med SETools. Implementationens ioctl-dispatch kunde inte analyseras: ingen `ion.ko`, kernelkälla eller symboliserad kernel hittades; `sysm/lib/libion.so` är 0 byte. Detta förändrar inte den bekräftade konfigurations-/policyexponeringen och ger ingen separat CVE-träff.

## F-003 — Mali-koppling till CVE-2023-4211

- **Komponent:** `mali_kbase.ko`, Mali-T624 Midgard, version `r14p0-01rel0`, Linux vermagic `3.10.79`.
- **Beskrivning:** Den lokala ELF32 ARM-modulen har SHA256 `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c`. Binärjämförelsen finner gamla tracking-VMA-symboler `kbase_mmap` @ `0x13338`, `kbase_special_vm_close` @ `0x113d4` och `kbasep_os_process_page_usage_update` @ `0x14310`. Den fixspecifika `process_mm`-referenslivscykeln och kontrollen att tracking-VMA är en sida saknas i analyserad väg. Separat `__mmdrop`-anrop i annan cleanup-väg finns och räknas inte som fix.
- **CVE:** CVE-2023-4211.
- **Klassificering:** Stark statisk matchning mot känd Mali use-after-free; inte en bekräftad exploit eller verifierad runtime-kompromettering.
- **Aktuell bevisstatus:** Modulversionen ligger inom Project Zeros/Arm angivna Midgard-intervall r12p0–r32p0 och relevanta funktioner jämfördes med Googles publicerade patch. Lokal firmware matchar build 5.433; Sony-signatur/proveniens och exakt `.0183EUB` subrevision har inte kryptografiskt verifierats.
- **Evidens:** `reports/sony-5433-cve-2023-4211-google-fix-comparison.md`, `reports/Fas-3C-Static-Firmware-Analysis.md`; lokal analysfil `work/sony-5433-extracted/analysis/cve-fix-comparison-evidence.txt`.
- **Referenser:** [Project Zero RCA](https://googleprojectzero.github.io/0days-in-the-wild/0day-RCAs/2023/CVE-2023-4211.html), [AOSP patch commit](https://android.googlesource.com/kernel/google-modules/gpu/+/35feb9b795cf1cd0d9a0a2edb6ade3c83040f48b%5E%21/), [CVE.org](https://www.cve.org/CVERecord?id=CVE-2023-4211).
- **Historik:** Ursprunglig repo-post F-003 kallade kopplingen obekräftad och krävde kod/patchbevis. Fas 3C-binärjämförelsen uppfyller statiskt krav; exploitbarhet förblev otestad.
- **Senast uppdaterad:** 2026-11-04.

### Fas 3D-tillägg

`kbase_ioctl` vid `0x177b8` (storlek `0xbb8`) disassemblerades. Den har dispatch till bland annat `kbase_mem_alloc/import/alias/commit/query/flags_change/free`, `kbase_jd_submit`, sync-, fence-, JIT- och profiling-funktioner. UAPI-header saknas, så numeriska ioctl-index kunde inte säkert namnges. Befintlig CVE-2023-4211-status ändras inte; ingen ny Mali-CVE fastställdes.

## F-004 — Sony WebAPI / broadcastLink

- **Komponent:** `sysm/etc/webapi/WebApi.xml`, WebApiCore och broadcastLink servlet.
- **Beskrivning:** XML deklarerar `/sony/broadcastLink`, klasspath `/system/framework/com.sony.dtv.webapi.service.broadcastlink.jar` och `BroadcastLinkServlet`. Deklarationen bevisar inte att implementation/JAR finns i publicerad systembild, att tjänsten startar, att den lyssnar på nätverk eller att autentisering saknas.
- **CVE:** Ingen matchad.
- **Klassificering:** Bekräftad tjänst-/servletdeklaration; potentiell granskningsyta, ej bekräftad sårbarhet.
- **Aktuell bevisstatus:** Deklarationen bekräftad lokalt. Auth/caller/input checks, nätverksbindning och faktisk reachability okända.
- **Evidens:** `reports/Fas-3C-Static-Firmware-Analysis.md`; lokalt `work/sony-5433-extracted/sysm/etc/webapi/WebApi.xml`.
- **Historik:** Ursprunglig repo-post F-004 var obekräftad. Fas 3C bekräftade endast deklarationen, inte dess körning eller exponering.
- **Senast uppdaterad:** 2026-11-04.

### Fas 3D-tillägg

`sysm/etc/webapi/WebApi.xml:8-10` deklarerar broadcastLink/JAR/servlet, men deklarerad JAR saknas i den extraherade kopian. Implementations-, autentiserings- och nätverksreachability-status förblir okänd.

## F-005 — Bred DAC-konfiguration för ytterligare device nodes

- **Komponent:** `/dev/sw_sync` och `/dev/block/mmcblk0p18`.
- **Beskrivning:** ueventd anger 0666 för båda noderna. Den effektiva SELinux-regeln för vanliga appar, live-label/node-existens och partitionens funktion är inte fastställda.
- **CVE:** Ingen matchad.
- **Klassificering:** Bekräftad DAC-konfigurationsrad; möjlig exponering, inte bekräftad appåtkomst/sårbarhet.
- **Aktuell bevisstatus:** Statiskt ueventd-underlag finns; appdöm och funktion ej verifierade.
- **Evidens:** `reports/Fas-3C-Static-Firmware-Analysis.md`; lokalt `work/sony-5433-extracted/analysis/boot_ramdisk/ueventd.mt5890.rc` och `analysis/boot_ramdisk/fstab.mt5890`.
- **Historik:** Tillkom Fas 3C.
- **Senast uppdaterad:** 2026-11-04.

### Fas 3D-tillägg

Explicit ueventd-inventering finns i Fas 3D-rapportens tabell. `/dev/sw_sync` och `/dev/block/mmcblk0p18` är fortsatt `0666`; exakta labels för dessa och flera andra noder kunde inte verifieras eftersom den gamla kompilerade `file_contexts.bin` inte kunde path-avkodas. SELinux-regler per kandidattyp redovisas separat från DAC.

Kompletterande skriptgranskning fann `sysm/bin/dtv_svc.sh:27-28`, som efter ueventd avser att ändra `/dev/video*` till `0660 root:1006` (kommenterad som `root.camera`). Körordning och faktiskt runtime-ägarskap har inte verifierats. `rfsA/etc/scripts/mkpts.sh:77-79` skapar separat RFS `/dev/ptmx` med mode 0666 och monterar `devpts`; detta är inte bevis för att Android-appar når noden. Kontextfilens råa strängar innehåller `ptmx_device` och `/dev/ptmx`, men varken Android-domäners åtkomst till typen eller nodens relation till Android-processer har fastställts.

## F-006 — Kärnans patch- och konfigurationsstatus

- **Komponent:** Linux 3.10.79 ARM/MT5890 kernel image.
- **Beskrivning:** Kernelversion är bekräftad, men kernelconfig, `vmlinux`, `System.map`, källkod och fullständig Sony/MediaTek patchserie saknas i det inventerade underlaget. Versionen ensam styrker ingen ytterligare CVE.
- **CVE:** Ingen ytterligare CVE matchad.
- **Klassificering:** Kunskapslucka/underhållsrisk, inte en bekräftad sårbarhet.
- **Aktuell bevisstatus:** uImage-version bekräftad; mitigationer och patchstatus ofullständiga.
- **Evidens:** `reports/Fas-3C-Static-Firmware-Analysis.md`; lokal `work/sony-5433-extracted/parts/boot.kernel` SHA256 `8f0d4a3b0520e642c666f027afce04a2a8e3be36a52be01d0d8f3387e565ddfa`.
- **Historik:** Tillkom Fas 3C.
- **Senast uppdaterad:** 2026-11-04.

### Fas 3D-tillägg

Ingen ny kernelkälla, config, `vmlinux`, `System.map` eller patchserie tillkom i inventerat underlag. Ingen ytterligare CVE matchas enbart utifrån kernelversion.

## F-007 — Oskyddad exporterad TNF/Samba content provider

- **Komponent:** `InteractiveTvPlatform.apk`, `TnfContentProvider` (`content://com.sony.dtv.interactivetvplatform.discovery.provider`).
- **Beskrivning:** providern är `exported=true` och saknar `android:permission`, `readPermission` och `writePermission`. DEX-metoderna `query`, `insert`, `update`, `delete` utför SQLite CRUD mot `TNF_DATABASE.db`; URI-mappningen omfattar bl.a. `samba_enabled`, `samba_port_number`, `samba_keys`, `samba_notify`, länder och CDN-version.
- **CVE:** Ingen matchad.
- **Klassificering:** Bekräftad statisk access-control-brist / cross-app read-write-yta.
- **Bevisstatus:** Manifest- och DEX-evidens bekräftar export, avsaknad av providerpermission och CRUD-implementation. Android 7 AOSP `ContentProvider` släpper igenom standard read/write för exporterad provider utan permission/path permission. Ingen app anropade komponenten; faktisk data och följdeffekt är inte testade.
- **Evidens:** `reports/Fas-3D-Deep-Interface-Security-Analysis.md`; `work/sony-5433-extracted/sysm/app/InteractiveTvPlatform/InteractiveTvPlatform.apk`, DEX code offsets `<clinit> 0x9eec0`, `getEntry 0x9efa8`, `onCreate 0x9f270`, `query 0x9f2b4`, `insert 0x9f1cc`, `update 0x9f370`, `delete 0x9f090`. AOSP-källa: `ContentProvider.java` tag `android-7.0.0_r33`, read gate rader 553–591 och write gate 630–668.
- **Historik:** Tillkom Fas 3D från riktad manifest-/DEX-granskning. Ingen CVE eller runtime exploatering påstås.
- **Senast uppdaterad:** 2026-11-04.

## F-008 — DLNA provider med oreglerad skrivpermission

- **Komponent:** `SemcDlna.apk`, `com.sonyericsson.dlna.datasource.provider.DlnaUriProvider`.
- **Beskrivning:** providern är exporterad och har `readPermission=com.sonymobile.dlna.permission.READ_DATA`, men saknar `writePermission`, generell `permission` och path-permission. DEX `insert`, `delete` och `update` öppnar `DlnaUriDatabaseHelper.getWritableDatabase()` och utför SQLite-operationer på `datasource`.
- **CVE:** Ingen matchad.
- **Klassificering:** Bekräftad statisk oreglerad cross-app write-yta; efterföljande påverkan okänd.
- **Bevisstatus:** Manifestfält, Android 7 write-permission-beteende samt CRUD-call sites är verifierade statiskt. Ingen provideranvändning eller effekt testades.
- **Evidens:** `reports/Fas-3D-Deep-Interface-Security-Analysis.md`; `work/sony-5433-extracted/sysm/app/SemcDlna/SemcDlna.apk`, DEX `delete 0x8c7fc`, `insert 0x8c9a0`, `update 0x8cca8`, `query 0x8cacc`.
- **Historik:** Tillkom Fas 3D genom statisk manifest-/DEX-granskning.
- **Senast uppdaterad:** 2026-11-04.

## F-009 — Sony exporterade Binder-gränssnitt utan manifestpermission

- **Komponent:** `InteractiveTvPlatform.apk` och `InteractiveTvUtil.apk`; bl.a. VideoShmServ, AudioShmServ, NativeSystemsManager, log-/privacy-/action-log-services.
- **Beskrivning:** flera services är uttryckligen exporterade utan manifestpermission och returnerar Binder-stubbar från `onBind`. Riktad DEX-sökning fann inga direkta caller-permission/UID-check-anrop i granskade service-entrypoints. Fullständig transaktions- och delegeringskedja är inte analyserad.
- **CVE:** Ingen matchad.
- **Klassificering:** Indikerad IPC-attackyta; sårbarhet eller otillbörlig åtkomst ej fastställd.
- **Bevisstatus:** Export/permission och Binder-retur bekräftade statiskt. Transaktionsnivåns kontroller och faktisk påverkan är okända.
- **Evidens:** `reports/Fas-3D-Deep-Interface-Security-Analysis.md`; `InteractiveTvPlatform.apk` DEX `AudioShmServ.onBind 0x9b554`, `NativeSystemsManager.onBind 0x9dc08`, `VideoShmServ.onBind 0x9e738`; `InteractiveTvUtil.apk` `ActionLogCollectorService.onBind 0x1dbe0c`, `ApplicationLogCollectorService.onBind 0x1dc288`, `PrivacyPolicyService.onBind 0x15ae80`.
- **Historik:** Tillkom Fas 3D. Föreslagen fortsättning är full statisk granskning av Binder Stub-transaktioner och caller-checks.
- **Senast uppdaterad:** 2026-11-04.

## Registerhistorik

- Ursprungsversionen av F-001–F-004 var preliminära; de första obekräftade statusarna finns kvar i repositoryts tidigare Git-commit.
- 2026-11-04: Fas 3C-evidens lades till utan att firmwarebinärer eller andra proprietära filer publicerades. Gamla rapporter omöjliga att förena med senare evidens har arkiverats med tydlig historiknotering.
- 2026-11-04: Fas 3D lade till ioctl-dispatchöversikt, full ueventd-inventering med path-context-begränsning och tre nya fynd (F-007–F-009). Endast textdokumentation uppdaterades; ingen runtime-testning genomfördes.
- Nästa fasuppdatering ska läggas till som status-/evidenshistorik; ID:n återanvänds inte för nya fynd.
