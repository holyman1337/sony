# ChatGPT handoff — Sony KD-55XD9305

**Läge:** Fas 3C är senast färdigställd. Fas 3D är begärd som nästa statiska analys, men är inte färdig och den här synkroniseringen startar den inte. Den GitHub-anslutna arbetskopian är `work/sony-git`, branch `main`, `origin` = `https://github.com/holyman1337/sony.git`.

## Genomfört och undersökt

- Fas 1 läsande Sony-TV-inventering; runtime uppgav Android 7/API 24, Linux 3.10.79, Mali-T624, SELinux Enforcing och `/dev/mali0` DAC 0666.
- Firmware/policy: lokal 5.433 EUB-kopia extraherades; kompilerad normal boot SELinux-policy och file contexts analyserades.
- Mali fixjämförelse: hashad ARM-modul jämfördes statiskt med Googles CVE-2023-4211-fix.
- Praktisk Sony runtime-validering gjordes läsande i tidigare fas; inga Mali-öppningar/ioctl.
- VM-TV Fas 2–3B gällde en generisk x86 Android 7/API 24 AVD, inte Sony/MT5890/Mali.
- Fas 3C sammanställde lokal statisk kernel-, Mali-, policy-, device-node- och Sony-konfigurationsanalys.

## Viktigaste aktuella fynd

1. **F-001 Mali-åtkomst:** policy tillåter `untrusted_app` `open/read/write/ioctl` på `gpu_device:chr_file`; ueventd anger `/dev/mali0 0666 root:root`. Detta bekräftar konfigurerad direktåtkomst på DAC/SELinux-lagren, inte körd appoperation.
2. **F-003 CVE-2023-4211:** Mali r14p0-01rel0 ligger inom det angivna Midgard-intervallet; `kbase_mmap` @ `0x13338` och `kbase_special_vm_close` @ `0x113d4` behåller den gamla tracking-VMA-vägen. Den fixspecifika process-mm-livscykeln och en-sidas-kontrollen saknas i jämförd väg. Stark statisk CVE-matchning, exploit/runtime-verkning ej verifierad.
3. **F-002 ION:** `untrusted_app` tillåts `ioctl/read/write` mot `ion_device:chr_file` och `/dev/ion` DAC 0666. Ingen ION CVE har fastställts.
4. **F-005 device nodes:** `/dev/sw_sync` och `/dev/block/mmcblk0p18` har 0666 i ueventd; SELinux-appåtkomst och funktion är okända.
5. **F-004 WebAPI:** `WebApi.xml` deklarerar `/sony/broadcastLink` och servlet. Implementation, autentisering och nätverksreachability ej verifierade.
6. **F-006 kernel:** 3.10.79 bekräftad; config/source/vendor patchstatus okända. Ingen annan kernel-CVE tillskriven enbart på version.

## Underlag och bevisvägar

- Senaste fullrapport: [`reports/Fas-3C-Static-Firmware-Analysis.md`](reports/Fas-3C-Static-Firmware-Analysis.md).
- [Firmware-/SELinux-rapport](reports/sony-5433-firmware-extraction-and-policy.md); [Mali-fixjämförelse](reports/sony-5433-cve-2023-4211-google-fix-comparison.md).
- Lokala, ej publicerade evidensartefakter: `work/sony-5433-extracted/analysis/cve-fix-comparison-evidence.txt`, `analysis/mali-policy-analysis.txt`, `analysis/boot_ramdisk/sepolicy`, `analysis/boot_ramdisk/ueventd.mt5890.rc`, `analysis/system/mali_kbase.ko`.
- Mali-modul SHA256: `8660a08a76729c8e2cfeb6afacd70765bb89615fbc0ea47e2554de1a98fb405c`.
- SELinux-policy SHA256: `13da0c50cf46ebfeb250615d57803044fcd7f1bf391ba40c397fbd7f2d1223ea`.
- Allmänna findings och statusar: [`FINDINGS.md`](FINDINGS.md), [`PROJECT-STATUS.md`](PROJECT-STATUS.md).

## Osäkerheter och korrigerade antaganden

- Initial repo-version klassade F-001–F-004 som obekräftade i väntan på evidens. Fas 3C uppdaterade F-001/F-002/F-003/F-004 med olika bevisstyrka; de ursprungliga statusarna finns i Git-historiken.
- Tidiga rapporter som saknade policy ersattes på evidensnivå efter extraktion av 5.433 boot-ramdisk; historiska kopior är märkta som sådana.
- 5.457-firmware får inte användas som ersättning. 5.433 build/fingerprint matchar, men intern `.0183`-mappning/signaturkedja är inte oberoende verifierad.
- F-003 visar kod-/versionsmatchning, inte att exploit fungerar eller att den fysiska TV:n kan komprometteras.
- Fynd från VM-emulatorn gäller bara generisk x86-gäst.
- Publiceringskopior maskerar privata TV-IP och lokala användarsökvägar; firmwarebinärer, APK:er, kernelmoduler och images har inte lagts till i Git.

## Frågor till ChatGPT-granskning

- Är skillnaden mellan bekräftad statisk device-åtkomst och ej testad runtime-operation tillräckligt tydlig?
- Är F-003 lämpligt klassad som stark CVE-matchning snarare än exploaterbarhetsbekräftelse?
- Bör Fas 3D prioritera full SELinux/device-node-matrisen eller statisk WebAPI/APK/Binder-kodgranskning först?

## Nästa fas

Fas 3D: fullständig device-node/context/allow-matris; statisk identifiering och ioctl-/minneshanteringsanalys av ION; kompletterande Mali dispatch-/patchjämförelse; och WebApiCore/APK manifest/Binder/caller-checks. Följ `CODEX-INSTRUCTIONS.md`: färdig rapport i `reports/`, uppdatera denna fil/status/findings, granska `git status`, `git diff --check`, `git diff --stat`, hela diff och publiceringsinnehåll; committa och pusha endast text till `origin/main`. Vid konflikt eller pushfel får fjärrhistorik inte skrivas över.
