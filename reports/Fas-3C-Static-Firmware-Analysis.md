# Sony KD-55XD9305 — Fas 3C: Statisk firmwareanalys

**Analysdatum:** 2026-11-04\
**Mål:** lokala firmwareartefakter för Sony SVP-DTV15 / KD-55XD9305, release 5.433 EUB\
**Metod:** endast statisk och läsande analys av lokala kopior. Ingen ADB, fysisk TV, emulator, firmwareinstallation, enhetsöppning, ioctl, exploit eller fuzzning användes.

## 1. Sammanfattning

I den extraherade Sony-firmwarekopian finns en tydlig och betydelsefull Mali-exponering: boot-ramdiskens DAC-konfiguration sätter `/dev/mali0` till `0666 root:root`, och den kompilerade SELinux-policyn ger `untrusted_app` `open`, `read`, `write` och `ioctl` på `gpu_device:chr_file`. Detta innebär att vanliga tredjepartsappar som placeras i `untrusted_app`-domänen tillåts nå Mali-device-noden direkt enligt firmwarekonfigurationen. Det är en statisk policy-/DAC-slutsats; ingen app öppnade noden i denna fas.

Den extraherade `mali_kbase.ko` identifieras som ARMv7 Mali Midgard r14p0-01rel0 för Linux 3.10.79. Binäranalys i den föregående CVE-rapporten fann den gamla tracking-VMA-livscykeln och frånvaro av Googles fixändringar i `process_mm`-vägen. Arm anger Midgard r12p0–r32p0 som berörda versioner för CVE-2023-4211. Sammantaget är detta ett **starkt statiskt CVE-matchande fynd** i firmwarekopian. Exploitabilitet, faktisk runtime-lastning och konsekvens på TV:n testades inte.

Ytterligare bekräftad policyobservation: `untrusted_app` har också `ioctl` samt read/write på `ion_device:chr_file`, och ueventd-konfigurationen anger `/dev/ion` som DAC 0666. Detta är en separat exponerad kärndrivrutinsyta; ingen sårbarhet har fastställts för den här enheten i denna analys. `/dev/sw_sync` och `/dev/block/mmcblk0p18` anges också som 0666 i ueventd-konfigurationen, men deras SELinux-åtkomst från appdomänen och den praktiska betydelsen kunde inte fastställas tillräckligt här.

Kärnan är ett ARM Linux 3.10.79-image, byggt 2019-03-08. Ingen konfigurationsfil, `vmlinux`, `System.map`, källkod eller komplett patchhistorik hittades i inventerat material. Därför går det inte att verifiera dess kernelconfig, vilka mitigationer/optioner som är aktiva eller göra tillförlitlig CVE-korrelation för den generiska kärnan enbart från versionssträngen.

## 2. Undersökt firmwarematerial

### 2.1 Identitet och proveniens

Lokalt underlag finns i `work/sony-5433-extracted/`. Arkiv-/extraktionsrapporten anger ZIP-storlek 1 644 753 328 byte och SHA256 `fee0a3edebcd18da5ba429ca95869812c78d7c26eb6cf09096442482592aeb73`; ZIP CRC-kontrollen passerade vid tidigare extraktion. Den här fasen hashade inte om den ursprungliga ZIP-filen. ZIP:ens inre PKG anges som 1 644 473 468 byte, SHA256 `5bad529cbc7ef2e66d074467bc2f5d3cff09c260f3f77aeaaedcf08496d2cdd2` enligt föregående extraktionsrapport.

Innehållet i `sysm/build.prop` bekräftar följande metadata:

- `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys`
- Android 7.0, API 24, ARMv7 (`armeabi-v7a`), plattform `mt5890`
- säkerhetspatchnivå `2019-02-01`
- produktnamn `SVP4KDTV15_EU`, enhet `SVP-DTV15`, buildtyp `user`

Paketets `cfig.bin`-identifierare `15390183` stöder `.0183`-delversionen, men ingen auktoritativ Sony-mappning av denna identifierare eller Sony-publicerad kontrollsumma/signaturkedja finns i underlaget. Firmwarekopian är därför verifierad som build `NRD91N.S34/5.433` för plattformen, men exakt identitet `PKG5.433.0183EUB` och Sony-proveniens är **inte formellt kryptografiskt verifierade**. Se även [föregående extraktionsrapport](sony-5433-firmware-extraction-and-policy.md).

### 2.2 Komponenter och hashar

| Artefakt | Storlek | SHA256 | Betydelse |
|---|---:|---|---|
| `parts/boot.bin` | 12 481 832 | `03fd2f151e00c876b9501df86cc2a2839cb2f54676af6da06bbd8c45b6bcb040` | Android boot image; källa för normal boot-ramdisk och policy |
| `parts/boot.kernel` | 4 585 152 | `8f0d4a3b0520e642c666f027afce04a2a8e3be36a52be01d0d8f3387e565ddfa` | U-Boot uImage, Linux/ARM 3.10.79 |
| `parts/sysm.bin` | 1 339 318 272 | `85da95b36c659c8c5f50b994868c05093d7fabdae24c614b8105805c8d87498f` | SquashFS 4.0 LZ4, Android-/Sony-systemfiler |
| `parts/rfsA.bin` | 79 609 856 | `c69b370cc19ce1d3a3ce08885a5cb2dc399b2afd857a145fdc271efdd5190b85` | SquashFS 4.0 LZO, MediaTek/Sony TV Linux-del |
| `parts/3rdp.bin` | 80 572 416 | `4bd359b5354da5a44a9fd86dc71381047edaef65506d10f13b0fec6b46fc54c2` | SquashFS 4.0 LZO, tredjepartskomponenter |
| `analysis/system/mali_kbase.ko` | 400 067 | `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c` | Mali kernelmodul; viktigaste analyserade binär |
| `analysis/system/libGLES_mali.so` | 4 800 460 | `a6f463e9cb34591c5818e25437ca346130f113e2f85e5b5e0f90b3e96adcf835` | ARM EGL/GLES userspace-bibliotek |
| `analysis/system/build.prop` | 3 954 | `f47be96685979a122f157d0296c45276b8f589a8be9ed5075127004b27b4a54b` | Build-/produktmetadata |
| `analysis/boot_ramdisk/sepolicy` | 224 113 | `13da0c50cf46ebfeb250615d57803044fcd7f1bf391ba40c397fbd7f2d1223ea` | Kompilerad normal SELinux-policy v30 |
| `analysis/boot_ramdisk/file_contexts.bin` | 164 559 | `17ed7576639c202ad6a83f2df15a878b2b95618ab930a89554bb186fae5ee6e9` | Kompilerade filkontexter |

`sysm.bin`, `rfsA.bin` och `3rdp.bin` är SquashFS-bilder. `boot.bin` innehåller kernel och ramdisk. Modulfilen har ELF32 ARM EABI5-metadata, `vermagic=3.10.79 SMP preempt mod_unload ARMv7`; modulens versionssträng är `K:r14p0-01rel0`. Modulmetadata visar Sony signerarinformation, men själva signaturens kryptografiska kedja verifierades inte mot en oberoende Sony-nyckel.

## 3. Linux-kärnan

### Bekräftat

- Boot-kärnan identifieras av uImage-headern som **Linux 3.10.79, Linux/ARM**, payload cirka 4,58 MB, datum **2019-03-08 13:32:58 UTC** enligt headern.
- Firmwareplattformens metadata anger MediaTek `mt5890`, ARMv7.
- Kernelmoduler i `sysm.bin/lib/modules` omfattar `android.ko`, `fuse.ko`, `gator.ko`, `libcomposite.ko`, `mali_kbase.ko`, `mtk_mmap.ko`, `musb_hdrc.ko`, `ntfs.ko`, `uvcvideo.ko`, V4L2-moduler och `wlan.ko`.
- `rfsA.bin` innehåller bland annat `dtv_driver.ko`, `fusion.ko`, `sony_bectrl.ko`, `sony_pqctrl.ko`, `snd-mtk.ko`, ytterligare V4L2/WLAN/USB/filystemsmoduler och MediaTek-moduler. `3rdp.bin` innehåller `cifs.ko`.
- RFS-komponentens init-/shellmaterial innehåller skript med `insmod` för en del MediaTek-/TV-moduler. Förekomsten visar att modulinläsning ingår i firmwaredesignen, men inte exakt vilka skript som körs i varje runtime-läge eller vilken ordning de körs i.

### Inte fastställt

- Ingen `defconfig`, `/proc/config.gz`-kopia, `IKCONFIG`-payload, `vmlinux`, `System.map`, DTS/DTB eller Linux-källkod hittades i den inventerade arbetskopian. Strängsökning efter `IKCFG_ST`/`IKCFG_ED` och relevanta `CONFIG_`-nycklar i `boot.kernel` gav ingen konfiguration.
- Ingen säker slutsats kan därför dras om exempelvis `CONFIG_SECCOMP`, `CONFIG_HARDENED_USERCOPY`, `CONFIG_KASLR`, `CONFIG_DEBUG_RODATA`, kernel lockdown, syscall-filter, stackprotectorinställningar eller vilka valfria subsystems som är byggda.
- System call-ytan kan inte fullständigt bestämmas från ett kompilerat kernelimage utan motsvarande symboler/config/källkod. Den generella Linux 3.10.79-versionen räcker inte för att fastställa att en specifik kernel-CVE är närvarande: Sony kan ha bakportat patchar, och MTK/Sony kan ha egna ändringar.
- Kernelns patchmängd utöver de analyserade Mali-ändringarna är inte kartlagd. Ingen CVE-lista baserad enbart på upstream-versionen presenteras som träff.

**Bedömning:** Kernelns ålder och brist på tillgänglig konfiguration innebär ett stort verifieringsgap, inte i sig bevis för en viss sårbarhet. Linux 3.10.79 släpptes 2015; firmwareimage är daterad 2019. Versionsskillnaden kan inte ersätta analys av vendorpatchar.

## 4. Mali-T624 och grafikstack

### 4.1 Driver och användarrymd

- `mali_kbase.ko`: ARMv7-modul, `r14p0-01rel0`, vermagic Linux 3.10.79.
- `libGLES_mali.so`: ELF32 ARM userspace EGL/GLES-bibliotek.
- Firmwarekonfigurationen använder `/dev/mali0`; tidigare enhetsdiagnostik rapporterade noden `0666 root:root`, label `u:object_r:gpu_device:s0`, SELinux Enforcing. De runtime-observationerna skedde i en tidigare fas och upprepades inte nu.
- Boot-ramdiskens `ueventd.mt5890.rc` deklarerar `/dev/mali0 0666 root root`.

### 4.2 SELinux-policy

Den analyserade policyn är den kompilerade **normal boot**-policyn ur `boot.bin`, policyversion 30. SETools-resultat i det tidigare analysunderlaget rapporterar 828 typer, 8 284 `allow`-regler, 111 `allowxperm`-regler, noll permissiva typer och okända klasser nekas. Ingen textkälla `.te`/`.cil` följde med.

De explicita reglerna för följande processdomäner är alla:

```selinux
allow <domain> gpu_device:chr_file { append getattr ioctl lock open read write };
```

| SELinux-domän | open | read | write | ioctl | getattr | map | allowxperm på Mali |
|---|:---:|:---:|:---:|:---:|:---:|---|---|
| `untrusted_app` | Ja | Ja | Ja | Ja | Ja | Ingen separat `map`-permission i policyklassens gemensamma filrättigheter | Ingen träff |
| `platform_app` | Ja | Ja | Ja | Ja | Ja | Samma begränsning | Ingen träff |
| `priv_app` | Ja | Ja | Ja | Ja | Ja | Samma begränsning | Ingen träff |
| `system_app` | Ja | Ja | Ja | Ja | Ja | Samma begränsning | Ingen träff |
| `sony_platform_app` | Ja | Ja | Ja | Ja | Ja | Samma begränsning | Ingen träff |
| `surfaceflinger` | Ja | Ja | Ja | Ja | Ja | Samma begränsning | Ingen träff |
| `system_server` | Ja | Ja | Ja | Ja | Ja | Samma begränsning | Ingen träff |
| `shell` | Ja | Ja | Ja | Ja | Ja | Samma begränsning | Ingen träff |

I SELinux-policyformatet här definieras ingen separat `map`-permission för `chr_file`; detta ska inte tolkas som en uttrycklig deny eller allow för en abstrakt `map`-rättighet. Inga `allowxperm`-regler hittades för `gpu_device:chr_file`; därför finns ingen policybaserad ioctl-nummerbegränsning av den typen. Detta säger inte vilka ioctl-opcodes själva drivrutinen accepterar.

Policyn mappar vanliga app-UID:n utan plattformsprivilegier till `untrusted_app` enligt `seapp_contexts`. Det är därför en vanlig tredjepartsappdomän, och inte enbart SurfaceFlinger eller en privilegierad Sony-tjänst, som uttryckligen har Mali-access i policy.

### 4.3 Statisk jämförelse med CVE-2023-4211

Föregående binäranalys av samma hashade modul identifierade:

- `kbase_mmap` vid ELF-adress `0x13338` och `kbase_special_vm_close` vid `0x113d4` finns kvar.
- Tracking-VMA-vägen sparar `current->mm` i `process_mm` utan att ta en motsvarande livstidsreferens i den vägen.
- Close-vägen tar bort/clears pekaren, väntar på RCU och använder därefter den tidigare `mm_struct`-pekaren för sidräkningsbokföring.
- Tracking-vägen saknar den fixens explicita begränsning att VMA ska vara exakt en sida.
- Googles fixspecifika `mmgrab()`/`mmdrop()`-ägarskap för `process_mm` saknas i denna livscykel. En separat `__mmdrop`-anropsplats i en annan minnesobjekt-cleanup finns, men är inte fixen för `process_mm`.

Google Project Zero beskriver CVE-2023-4211 som en use-after-free i Mali tracking-page/VMA-hanteringen och anger Midgard r12p0–r32p0 som berört intervall. r14p0 ligger inom intervallet. Den officiella Android-bulletinen listar CVE:n under Mali och patchnivåerna 2023-10; firmwaremetadata anger 2019-02-01. Se [Project Zeros RCA](https://googleprojectzero.github.io/0days-in-the-wild/0day-RCAs/2023/CVE-2023-4211.html), [Googles patchcommit](https://android.googlesource.com/kernel/google-modules/gpu/+/35feb9b795cf1cd0d9a0a2edb6ade3c83040f48b%5E%21/), [Android Security Bulletin October 2023](https://source.android.com/docs/security/bulletin/2023-10-01) ja [CVE-posten](https://www.cve.org/CVERecord?id=CVE-2023-4211).

**Klassificering:** Starkt bekräftat statiskt fynd att den analyserade firmwarens Mali-driver är i berört versionsintervall och saknar den jämförda fixen i relevanta funktioner. Med SELinux/DAC-accessen ovan är app-till-driver-gränsen också statiskt öppen för `untrusted_app`. Detta bevisar **inte** att exploitkedja fungerar på TV:n, att den analyserade kopian har identisk kryptografisk proveniens med installerad image, eller att en specifik app faktiskt utförde operationen.

## 5. Android- och Sony-komponenter

### 5.1 Privilegier, appdomäner och systemfiler

- `seapp_contexts` skiljer mellan `untrusted_app`, `platform_app`, `priv_app` och Sony-domänen `sony_platform_app`; vissa `com.sony.dtv.*`-/Sonypaket mappas separat. Exakt runtime-UID/processlista samlades inte in i denna fas.
- Systempartitionen innehåller Sony-/MediaTek-TV-appar och konfigurationer, bland annat TV-/broadcast-/remote-relaterade appar, Sony feature- och permissions-XML samt MediaTek-tjänster. Förekomst på systempartitionen visar inte automatiskt att komponenten är privilegierad, aktiv eller exponerad.
- `sysm/etc/service/` innehåller Sony-servicehjälpskript. RFS-bilden innehåller MTK init- och modulskript. Dessa analyserades inte fullständigt med avseende på varje startvillkor, argumentvalidering och privilegiumövergång.
- `sysm/etc/webapi/WebApi.xml` deklarerar en Sony broadcast-link-servlet (`/sony/broadcastLink`) med implementation i `com.sony.dtv.webapi.service.broadcastlink.jar`. Detta är ett statiskt konfigurationsfynd; lyssnande socket, autentisering, nätverksbindning och runtime-åtkomst verifierades inte.
- Ingen fullständig dekompilering av Sony APK:er, Binder-API-granskning eller metod-för-metod behörighetsanalys gjordes i Fas 3C. Därför pekas ingen Sony-tjänst ut som sårbar här.

### 5.2 Hårdningsindikatorer i metadata

Boot-ramdiskens `default.prop` anger `ro.secure=1`, `ro.debuggable=0`, `ro.adb.secure=1`, `ro.allow.mock.location=0`, `security.perf_harden=1` och `persist.sys.usb.config=none`. `build.prop` anger `ro.build.type=user`, `release-keys` och `ro.boot.flash.locked=1`; äldre diagnostik har rapporterat SELinux Enforcing.

Detta är konfigurations-/propertyvärden från firmware, inte bevis för aktuell runtime-konfiguration. `ro.oem_unlock_supported=1` betyder att firmware rapporterar upplåsningsstöd och är **inte** bevis på att bootloadern är upplåst. ADB TCP-listener, autentiseringsbeteende och exponerade nätverkstjänster undersöktes inte i denna fas. Värdet `ro.adb.secure=1` är en positiv säkerhetsindikator, men kan inte ensamt bevisa vilka ADB-transportvägar som är aktiva.

Androids `2019-02-01` security patch property motsvarar den deklarerade patchnivån i Google-bulletinens schema; Androids kompatibilitetskrav säger att patchnivån ska indikera att bulletiner upp till datumet är åtgärdade. Detta är en builddeklaration, inte en oberoende verifiering av alla Sony-/MediaTek-/kärnpatchar. [AOSP:s 2019-bulletinöversikt](https://source.android.com/docs/security/bulletin/2019?hl=en) listar februari 2019 som `2019-02-01`; [Android 7.0 CDD](https://source.android.com/static/docs/compatibility/7.0/android-7.0-cdd.pdf?hl=en) anger semantiken för `SECURITY_PATCH`.

## 6. Exponerade enheter och gränssnitt

Boot-ramdiskens `ueventd.mt5890.rc` innehåller följande DAC-regler:

| Device node / mönster | DAC i boot-konfiguration | SELinux-status för vanliga appar i denna analys |
|---|---|---|
| `/dev/mali0` | `0666 root:root` | Bekräftad `untrusted_app`-allow till `gpu_device:chr_file` för open/read/write/ioctl/getattr |
| `/dev/ion` | `0666 system:system` | Bekräftad `untrusted_app`-allow till `ion_device:chr_file` för open/read/write/ioctl/getattr |
| `/dev/sw_sync` | `0666 root:root` | Den specifika appdomänens policyregel och runtime-label har inte verifierats |
| `/dev/block/mmcblk0p18` | `0666 root:root` | Partitionsändamål och SELinux-/runtime-åtkomst har inte verifierats; dra ingen slutsats om läs-/skrivbarhet utifrån DAC ensam |
| `/dev/video*` | `0660 system:camera` | Konfigurationen visar gruppbegränsning; appdomänens SELinux-regler/runtime-medling ej verifierade |
| `/dev/hidraw*` | `0660 system:audio` | Appdomänens SELinux-regler/runtime-medling ej verifierade |
| `/dev/trusty-ipc-dev0` | `0660 system:keystore` | DAC begränsar; konkret SELinux- och tjänstegräns ej analyserad |
| `/dev/block/mmcblk0rpmb` | `0660 media:mediadrm` | DAC begränsar; SELinux-/runtime-åtkomst ej analyserad |
| `/dev/stpbt` | `0660 bluetooth:net_bt_stack` | DAC begränsar; SELinux-/runtime-åtkomst ej analyserad |
| `/dev/ttyUSB0..2` | `0660 system:irblaster` | DAC begränsar; SELinux-/runtime-åtkomst ej analyserad |

`ion_device`-regeln verifierades direkt med SETools mot kompilerad policy. Övriga noder bedöms inte som åtkomliga eller oåtkomliga för appar utan måltyp, policyregel och korrekt runtime-kontext. Filkontexttabellen innehåller path-regex för flera av noderna, men ingen fullständig tabell över varje subjekt/objekt-behörighet togs fram här.

Androids grafikstack innehåller Mali EGL/GLES-bibliotek och direkt nodbehörighet för bland annat `surfaceflinger`; detta indikerar att både appprocesser och systemets grafikprocesser kan ha separata vägar mot drivrutinen. Någon körbar GPU-test, `mmap` eller ioctl gjordes inte. Androids grafikstack kan alltså inte i denna konfiguration antas vara en barriär som exklusivt brokerar all Mali-åtkomst.

## 7. Identifierade säkerhetsproblem och CVE-korrelationer

| ID | Fynd | Klass | Säkerhetsmässig bedömning | Status |
|---|---|---|---|---|
| F-01 | Mali r14p0-01rel0 i berört intervall för CVE-2023-4211; relevant tracking-VMA kod saknar jämförd fix | Sårbar kod / CVE-matchning | Local app-to-kernel attack surface är betydande eftersom `untrusted_app` har explicit Mali open/ioctl och DAC är 0666. Möjlig påverkan omfattar UAF-relaterad kernelminnessäkerhet; faktisk exploitation/påverkan ej testad. | Starkt statiskt bekräftad för analyserad firmwarekopia; installerad Sony-releaseproveniens ej kryptografiskt bevisad |
| F-02 | `untrusted_app` har full vanlig ioctl/read/write-allow till `ion_device`; `/dev/ion` DAC 0666 | Exponerad kärnyta / potentiell risk | En vanlig appdomän når även ION-drivrutinens ioctl-yta enligt policy. Ingen specifik defekt/CVE har identifierats, och funktionernas runtime-konfiguration är okänd. | Policy och ueventd-konfiguration bekräftade; sårbarhet ej fastställd |
| F-03 | `/dev/sw_sync` och `/dev/block/mmcblk0p18` anges som DAC 0666 | Misstänkt bred DAC-konfiguration | Potentiellt bred device access; utan SELinux-regler/labels, node-existens och partitionsfunktion är det inte möjligt att avgöra faktisk oprivilegierad åtkomst eller påverkan. | DAC-konfigurationsrader bekräftade; faktisk exponering ej verifierad |
| F-04 | Äldre Linux 3.10.79, utan konfigurations- eller patchkällor i materialet | Kunskapslucka / förhöjd underhållsrisk | Stor yta för potentiella historiska kärnproblem, men ingen specifik CVE kan tillskrivas endast från versionen. | Version bekräftad; CVE-/patchstatus ej fastställd |
| F-05 | Sony `broadcastLink` WebAPI-deklaration i konfigurationsfil | Gränssnitt att granska | Kan vara lokal, autentiserad, inaktiv eller inte nätverksbunden. Inte en sårbarhetsrapport i sig. | Statiskt endpoint-/servletfynd; runtime-ejenskaper ej verifierade |

CVE-korrelationen är medvetet begränsad till CVE-2023-4211, eftersom den kombinerar exakt komponent/version, berört intervall och binär jämförelse med den publicerade fixen. Inga andra CVE presenteras som träff utan matchad patch-, config- eller binärevidens. Linux 3.10.79 eller en gammal patch property är inte ensamt tillräckligt underlag för att deklarera en ytterligare sårbarhet.

## 8. Kunskapsluckor och begränsningar

1. Firmwarekällan var en tredjepartsspegel; Sony-publicerad checksumma/signaturkedja saknas. Build/fingerprint matchar, men `.0183EUB` är inte formellt verifierad.
2. Ingen runtime-insamling gjordes under Fas 3C. Modullastning, processer, listeners, nätverk, ADB-state och faktiska device-node labels/rättigheter bekräftades inte på TV:n i denna fas.
3. Kernelconfig, `vmlinux`, `System.map`, DTB/DTS, källkod och vendor patchseries saknas. Kernel-CVE-täckning och mitigationer kan därför inte verifieras helt.
4. SELinux-policyn är kompilerad och analyserad, men dess byggkällor saknas. `map` är inte en uttrycklig file-class-permission i denna policy; specifik Linux SELinux `mmap`-hookbedömning gjordes inte.
5. Mali driverens ioctl-dispatch, GPU-minnesfunktioner utöver CVE-kodvägen, samt ELF-/firmwarematchning mellan userspace och kernel på faktisk enhet har inte dynamiskt testats.
6. Sony APK-/Binder-/WebAPI-granskning är inventeringsnivå. Deklarerad komponent betyder inte att den är exporterad eller åtkomlig; manifest, permissions, caller checks, listener och nätverksbindning behöver granskas komponent för komponent.
7. DAC-regel i `ueventd` visar init-konfiguration, inte nödvändigtvis den aktuella nodens live-mode efter senare init-script eller SELinux-etikettering.
8. Ingen sårbarhet har testats, reproducerats eller exploaterats. Fas 3B:s generiska x86-emulator ingår inte som evidens för Sony-firmware.

## 9. Rekommenderade nästa steg för Fas 3D

1. Bevara proveniens: skaffa om möjligt Sony-publicerad hash/signatur eller en oberoende legitim firmwarekopia med jämförbar hash; verifiera den interna `.0183`-identifierarens officiella betydelse.
2. Skaffa exakt kernelconfig och vendor kernel source/patches för `SVP-DTV15`/MT5890 från Sony/MediaTek eller dokumenterat SDK. Gör offline CVE-/patch-diff mot just denna källkod.
3. Fortsätt passiv Mali-granskning i isolerat analysläge: kontrollera hela `mali_kbase.ko`-ioctl-tabellen och andra versioner/patchar med reproducerbara symbol-/disassemblyreferenser; inga device calls behövs.
4. Extrahera samtliga `ueventd` device labels och kör en fullständig statisk SETools-korsning mellan de oprivilegierade appdomänerna och alla `chr_file`/`blk_file`-typer. Lägg till DAC, init-regel, label, SELinux allow och syfte per nod; markera okända partitioners syfte.
5. Gör offline manifest-/permission-/Binder-granskning av prioriterade Sony APK:er och `WebApi.xml`; fastställ `exported`, permission-protection level, caller checks och avsedd listener från statiska källor. Redovisa dynamisk nåbarhet som okänd tills en separat godkänd, icke-destruktiv metod finns.
6. Håll fysisk TV utanför testning. Om praktisk runtime-verifiering senare behövs, begränsa den till redan tillåtna läsdiagnostik; inga Mali-öppningar/ioctl eller kerneltriggerförsök.

## 10. Spårbarhet och källor

Lokala bevisfiler:

- `work/sony-5433-extracted/analysis/system/build.prop`
- `work/sony-5433-extracted/analysis/boot_ramdisk/default.prop`
- `work/sony-5433-extracted/analysis/boot_ramdisk/ueventd.mt5890.rc`
- `work/sony-5433-extracted/analysis/boot_ramdisk/sepolicy`
- `work/sony-5433-extracted/analysis/boot_ramdisk/file_contexts.bin`
- `work/sony-5433-extracted/analysis/mali-policy-analysis.txt`
- `work/sony-5433-extracted/analysis/cve-fix-comparison-evidence.txt`
- `work/sony-5433-extracted/analysis/system/mali_kbase.ko`
- `work/sony-5433-extracted/analysis/system/libGLES_mali.so`
- `sony-5433-firmware-extraction-and-policy.md`
- `sony-5433-cve-2023-4211-google-fix-comparison.md`

Externa primärkällor:

- [Google Project Zero: CVE-2023-4211 root-cause analysis](https://googleprojectzero.github.io/0days-in-the-wild/0day-RCAs/2023/CVE-2023-4211.html)
- [Android/Google GPU patch commit `35feb9b795cf1cd0d9a0a2edb6ade3c83040f48b`](https://android.googlesource.com/kernel/google-modules/gpu/+/35feb9b795cf1cd0d9a0a2edb6ade3c83040f48b%5E%21/)
- [Android Security Bulletin, October 2023](https://source.android.com/docs/security/bulletin/2023-10-01)
- [CVE.org record: CVE-2023-4211](https://www.cve.org/CVERecord?id=CVE-2023-4211)
- [Android Security Bulletins 2019](https://source.android.com/docs/security/bulletin/2019?hl=en)
- [Android 7.0 Compatibility Definition Document](https://source.android.com/static/docs/compatibility/7.0/android-7.0-cdd.pdf?hl=en)
- [Linux kernel.org archive: Linux 3.10.79 release material](https://cdn.kernel.org/pub/linux/kernel/v3.x/)

## Samlad slutsats

Den starkaste konkreta säkerhetsobservationen i den analyserade Sony-firmwarekopian är Mali-r14p0-drivrutinens CVE-2023-4211-relaterade kodväg, i kombination med uttrycklig SELinux- och DAC-åtkomst från vanliga `untrusted_app`-processer. ION är en andra appåtkomlig kärnyta, men ingen specifik ION-sårbarhet har fastställts. Firmwarebilden ger dessutom indikationer på flera breda device-node DAC-lägen och en Sony WebAPI-komponent som förtjänar fortsatt statisk granskning.

Detta är inte ett bevis på en fungerande exploit eller på att någon annan konkret sårbarhet finns i Sony TV:n. Kernelns övriga CVE-status, runtime-tjänster och firmwarekopians formella releaseproveniens är ännu inte fastställda.
