# Projektstatus — Sony KD-55XD9305

Senast uppdaterad: 2026-11-04

## Projektmål och hårdvara

Defensiv säkerhetsanalys av Sony KD-55XD9305, plattform SVP-DTV15, utan exploatering, fuzzning, modifiering eller riskfylld testning av den fysiska TV:n.

- Chassi/board: SVP-DTV15; SoC MediaTek MT5890; ARMv7.
- Android 7.0 / API 24; Linux 3.10.79.
- GPU: Mali-T624; driveridentifierare r14p0-01rel0.
- Installerad version enligt tidigare läsande enhetsdiagnostik: `PKG5.433.0183EUB`.
- Fingerprint: `Sony/SVP4KDTV15_EU/SVP-DTV15:7.0/NRD91N.S34/5.433:user/release-keys`.
- Lokal extraherad firmwarekopia: release/build 5.433 EUB/SVP-DTV15 stöds av image-metadata. `cfig`-ID stöder `.0183`, men exakt subrevision och Sony-publicerad kryptografisk proveniens är inte formellt verifierade.
- ZIP SHA256 enligt tidigare extraktionsrapport: `fee0a3edebcd18da5ba429ca95869812c78d7c26eb6cf09096442482592aeb73`.

## Genomförda faser

| Fas/arbete | Status | Resultat och rapport |
|---|---|---|
| Fas 1: TV-inventering | Klar | Läsande ADB-inventering; Android/kernel/GPU, SELinux Enforcing och Mali-nod dokumenterades. [Fas 1](reports/Fas-1-TV-inventering.md) |
| Firmwareidentifiering och SELinux | Klar | 5.433 EUB buildmetadata och kompilerad boot-policy analyserades. [Extraktion och policy](reports/sony-5433-firmware-extraction-and-policy.md) |
| Mali/CVE-fixjämförelse | Klar | ARM-binär statiskt jämförd med publicerad CVE-2023-4211-fix. [Jämförelserapport](reports/sony-5433-cve-2023-4211-google-fix-comparison.md) |
| Praktisk Sony-validering | Klar | Läsande runtime-identitet/property-/filjämförelse; inga Mali-enhetsöppningar eller ioctl-anrop. [Valideringsrapport](reports/sony-tv-practical-security-validation.md) |
| VM-TV Fas 2–3B | Klar | Generisk Android 7 x86-emulator analyserad. Resultat gäller inte Sony/MT5890/Mali. [VM-rapporter](reports/vm-tv-lab/) |
| Fas 3C: statisk firmwareanalys | Klar | Kärna, Mali, policy, node-konfiguration och Sony-komponenter sammanställda. [Fas 3C](reports/Fas-3C-Static-Firmware-Analysis.md) |

Historiska rapporter före extraktionen av rätt 5.433-policy ligger kvar i `reports/` med en tydlig notering om att senare evidens har ersatt vissa preliminära slutsatser.

## Aktuell fas

Fas 3C är senast färdigställd. Fas 3D – fördjupad statisk analys av device nodes, ION, Mali-dispatch och Sony WebAPI/APK/Binder – är nästa begärda analysarbete men är inte färdigställt. Ingen analys startas automatiskt av denna Git-konfiguration.

## Aktuella huvudfynd

- **F-001/F-003 Mali:** vanlig `untrusted_app` har i den extraherade SELinux-policyn `open/read/write/ioctl` mot `gpu_device:chr_file`; ueventd anger `/dev/mali0 0666`. Den hashade r14p0-modulen har relevant gammal tracking-VMA-väg och saknar den jämförda Google-fixen för CVE-2023-4211. Stark statisk CVE-matchning, inte verifierad exploitbarhet.
- **F-002 ION:** `untrusted_app` tillåts ioctl/read/write mot `ion_device:chr_file`; `/dev/ion` är konfigurerad DAC 0666. Ingen separat ION-sårbarhet/CVE fastställd.
- **F-005:** `/dev/sw_sync` och `/dev/block/mmcblk0p18` konfigureras DAC 0666; SELinux-/appåtkomst och nodernas praktiska innebörd är inte fastställda.
- **F-004:** Sony `WebApi.xml` deklarerar `/sony/broadcastLink`; autentisering, implementation och nätverksreachability är inte verifierade.
- **F-006:** Linux 3.10.79 är identifierad, men kernelconfig, källkod och fullständig vendorpatchserie saknas; ingen ytterligare CVE tillskrivs på versionsnummer ensamt.

Se [FINDINGS.md](FINDINGS.md) för klassificering, status och evidenshistorik.

## Öppna frågor

1. Kan lokalkopian knytas kryptografiskt till exakt installerad subversion `.0183EUB`?
2. Vilka SELinux labels och appdomänregler gäller för samtliga device nodes, särskilt ION, sw_sync och blocknoder?
3. Vilken ION-implementation och ioctl-dispatch ingår i MT5890-kärnan, och vilka relevanta patchar kan beläggas?
4. Kan fullständig Mali ioctl-dispatch och övriga relevanta funktioner jämföras statiskt med publicerade fixar?
5. Finns broadcastLink-servleten i publicerbar firmwaredel, och vilka auth/caller/input checks kan beläggas statiskt?
6. Vilka Sony APK/Binder-komponenter är exporterade eller privileged och hur skyddas deras metoder?
7. Vilken kernelconfig, DTB och Sony/MediaTek patchserie gäller för denna board/release?

## Nästa steg

Efter separat fasstart: genomför Fas 3D som lokal statisk granskning. Spara full rapport under `reports/`, uppdatera fynd/status/överlämning, kontrollera diff och innehåll, och pusha endast textbaserade, rensade rapporter/metadata till `origin/main`. Ingen firmware, APK, kernelmodul, image, dump med personuppgifter eller hemlighet får publiceras.
