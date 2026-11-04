# ChatGPT handoff — Sony KD-55XD9305

**Läge:** Fas 3D är färdigställd och rapporterad. Nästa planerade steg är Fas 3E; den har inte startats. Git-arbetskopian är `work/sony-git`, branch `main`, `origin` = `https://github.com/holyman1337/sony.git`.

## Genomfört och undersökt

- Fas 1 läsande Sony-TV-inventering; runtime uppgav Android 7/API 24, Linux 3.10.79, Mali-T624, SELinux Enforcing och `/dev/mali0` DAC 0666.
- Firmware/policy: lokal 5.433 EUB-kopia extraherades; kompilerad normal boot SELinux-policy och file contexts analyserades.
- Mali fixjämförelse: hashad ARM-modul jämfördes statiskt med Googles CVE-2023-4211-fix.
- Praktisk Sony runtime-validering gjordes läsande i tidigare fas; inga Mali-öppningar/ioctl.
- VM-TV Fas 2–3B gällde en generisk x86 Android 7/API 24 AVD, inte Sony/MT5890/Mali.
- Fas 3C sammanställde lokal statisk kernel-, Mali-, policy-, device-node- och Sony-konfigurationsanalys.
- Fas 3D gjorde explicit ueventd-inventering, kontroll av skriptade node-ändringar, SETools-frågor per analyserad device-typ, partiell Mali ioctl-dispatch, ION/WebAPI-underlagskontroll och riktad manifest/DEX-granskning av Sony-/MediaTek-appar. Ingen TV, emulator eller device-node kontaktades.

## Viktigaste aktuella fynd

1. **F-001 Mali-åtkomst:** policy tillåter `untrusted_app` `open/read/write/ioctl` på `gpu_device:chr_file`; ueventd anger `/dev/mali0 0666 root:root`. Detta bekräftar konfigurerad direktåtkomst på DAC/SELinux-lagren, inte körd appoperation.
2. **F-003 CVE-2023-4211:** Mali r14p0-01rel0 ligger inom det angivna Midgard-intervallet; `kbase_mmap` @ `0x13338` och `kbase_special_vm_close` @ `0x113d4` behåller den gamla tracking-VMA-vägen. Den fixspecifika process-mm-livscykeln och en-sidas-kontrollen saknas i jämförd väg. Stark statisk CVE-matchning, exploit/runtime-verkning ej verifierad.
3. **F-002 ION:** `untrusted_app` tillåts `ioctl/read/write` mot `ion_device:chr_file` och `/dev/ion` DAC 0666. Ingen ION CVE har fastställts.
4. **F-005 device nodes:** `/dev/sw_sync` och `/dev/block/mmcblk0p18` har 0666 i ueventd; SELinux-appåtkomst och funktion är okända. `dtv_svc.sh:27-28` avser att sätta `/dev/video*` till 0660 root:1006 (kommentaren säger root.camera), men runtime-ordning är ej verifierad. RFS `mkpts.sh:77-79` skapar `/dev/ptmx` 0666 i separat RFS-kontext; appåtkomst från Android okänd.
5. **F-004 WebAPI:** `WebApi.xml` deklarerar `/sony/broadcastLink` och servlet. Implementation, autentisering och nätverksreachability ej verifierade.
6. **F-006 kernel:** 3.10.79 bekräftad; config/source/vendor patchstatus okända. Ingen annan kernel-CVE tillskriven enbart på version.
7. **F-007 TNF provider:** exporterad `TnfContentProvider` saknar provider/read/write permission och innehåller SQLite CRUD för TNF/Samba-tabeller. Bekräftat statiskt; ingen runtime-användning eller följdeffekt testad.
8. **F-008 DLNA provider:** exporterad `DlnaUriProvider` har readPermission men ingen writePermission; DEX utför SQLite write-operationer på `datasource`. Bekräftat statiskt; ingen runtime-användning eller påverkan testad.
9. **F-009 Binder:** flera Sony services exporterade utan manifestpermission returnerar Binder-stubbar. Tillgänglig IPC-yta är bekräftad; transaktionskontroller/sårbarhet ej fastställda.

Device-node-fördjupning: `gpu_device`/`ion_device`-regler ger `untrusted_app` open/read/write/ioctl; `allowxperm` saknas för dessa typer. `/dev/mali0` label `gpu_device` har tidigare verifierats på TV; andra path→label-mappningar är delvis osäkra eftersom host-libselinux inte kunde avkoda Android 7 `file_contexts.bin`.

## Underlag och bevisvägar

- Senaste fullrapport: [`reports/Fas-3D-Deep-Interface-Security-Analysis.md`](reports/Fas-3D-Deep-Interface-Security-Analysis.md); tidigare sammanhang i [`reports/Fas-3C-Static-Firmware-Analysis.md`](reports/Fas-3C-Static-Firmware-Analysis.md).
- [Firmware-/SELinux-rapport](reports/sony-5433-firmware-extraction-and-policy.md); [Mali-fixjämförelse](reports/sony-5433-cve-2023-4211-google-fix-comparison.md).
- Lokala, ej publicerade evidensartefakter: `work/sony-5433-extracted/analysis/cve-fix-comparison-evidence.txt`, `analysis/mali-policy-analysis.txt`, `analysis/boot_ramdisk/sepolicy`, `analysis/boot_ramdisk/ueventd.mt5890.rc`, `analysis/system/mali_kbase.ko`.
- Mali-modul SHA256: `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c`.
- SELinux-policy SHA256: `13da0c50cf46ebfeb250615d57803044fcd7f1bf391ba40c397fbd7f2d1223ea`.
- `TnfContentProvider`: `InteractiveTvPlatform.apk`; code offsets `query 0x9f2b4`, `insert 0x9f1cc`, `update 0x9f370`, `delete 0x9f090`.
- `DlnaUriProvider`: `SemcDlna.apk`; code offsets `delete 0x8c7fc`, `insert 0x8c9a0`, `update 0x8cca8`.
- `kbase_ioctl` dispatch: Mali module symbol `0x177b8`, size `0xbb8`; handler summary and callsites in Fas 3D report.
- Allmänna findings och statusar: [`FINDINGS.md`](FINDINGS.md), [`PROJECT-STATUS.md`](PROJECT-STATUS.md).

## Osäkerheter och korrigerade antaganden

- Initial repo-version klassade F-001–F-004 som obekräftade i väntan på evidens. Fas 3C uppdaterade F-001/F-002/F-003/F-004 med olika bevisstyrka; de ursprungliga statusarna finns i Git-historiken.
- Tidiga rapporter som saknade policy ersattes på evidensnivå efter extraktion av 5.433 boot-ramdisk; historiska kopior är märkta som sådana.
- 5.457-firmware får inte användas som ersättning. 5.433 build/fingerprint matchar, men intern `.0183`-mappning/signaturkedja är inte oberoende verifierad.
- F-003 visar kod-/versionsmatchning, inte att exploit fungerar eller att den fysiska TV:n kan komprometteras.
- Fynd från VM-emulatorn gäller bara generisk x86-gäst.
- Android 7 `file_contexts.bin` gav host-libselinux-felet `Old compiled fcontext format, skipping`; labels utöver Mali är därför kandidater om ingen annan källa löser exakta path associations.
- ION-kärnkod/ioctl-tabell saknas: ingen ION `.ko`, symboliserad kernel eller källa hittades; `sysm/lib/libion.so` är 0 byte.
- `WebApi.xml` pekar på broadcastLink-JAR som saknas i extraherad kopia. Endpointens drift, auth och nätverksreachability är okända.
- Inga binder/provider-anrop gjordes. DEX-sökningar efter direkta caller-check API:er är begränsade till granskade metoder och bevisar inte frånvaro av delegerade kontroller.
- Publiceringskopior maskerar privata TV-IP och lokala användarsökvägar; firmwarebinärer, APK:er, kernelmoduler och images har inte lagts till i Git.

## Frågor till ChatGPT-granskning

- Är skillnaden mellan bekräftad statisk device-åtkomst och ej testad runtime-operation tillräckligt tydlig?
- Är F-003 lämpligt klassad som stark CVE-matchning snarare än exploaterbarhetsbekräftelse?
- Är F-007 och F-008 korrekt klassade som bekräftade statiska access-control-brister utan att dra slutsats om runtime-impact?
- Vilken av F-007/F-008 och F-009 bör prioriteras för djupare statisk transaktions-/datakontraktsanalys?

## Nästa fas

Fas 3E bör: (1) följa `TnfContentProvider` och `DlnaUriProvider` downstream och fastställa data-/funktionspåverkan statiskt, (2) granska F-009:s Binder stubs/transaktioner och caller-auth end-to-end, (3) skaffa matchande kernel/ION source/config och avkoda exakta file contexts, (4) söka efter saknad broadcastLink implementation. Ingen runtime-testning mot fysisk TV. Efter slutförd fas: uppdatera rapport, findings, status och handoff; diffgranska och pusha enbart text till `origin/main` enligt `CODEX-INSTRUCTIONS.md`.
