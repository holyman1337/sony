> Historisk Fas 1-rapport. Senare policy- och firmwarefynd kompletterar/korrigerar slutsatser där de uttryckligen anges i efterföljande rapporter.

# Säkerhetskartläggning: Sony KD-55XD9305

**Datum:** 2026-11-04\
**Enhet:** Sony BRAVIA 4K 2015, KD-55XD9305, `SVP-DTV15`\
**ADB:** `<private-TV-IP>:5555`\
**Metod:** Läsande ADB-kommandon. Ingen root, filändring, inställningsändring, exploit, fuzzning, ioctl-anrop eller kraschtest.

## Sammanfattning

ADB-anslutningen till `<private-TV-IP>:5555` fungerar. `<private-TV-IP>:5555` visas offline; den svarande enheten på den nya adressen identifierar sig via systemegenskaper som KD-55XD9305.

Bekräftade huvudfynd:

- TV:n kör Android 7.0/API 24, build `NRD91N.S34/5.433`, säkerhetspatch `2019-02-01`.
- SELinux är `Enforcing`.
- `/dev/mali0` finns med DAC-rättigheterna `crw-rw-rw-` (0666), ägare/grupp `root:root`, SELinux-etikett `gpu_device`. `/dev/mali` finns inte.
- Samma 0666-läge gäller de observerade noderna `/dev/binder`, `/dev/ashmem` och `/dev/ion`. SELinux kan fortfarande begränsa åtkomst.
- ADB-daemon körs och TCP-port 5555 är konfigurerad. `ro.adb.secure=1` anger att ADB-autentisering är aktiverad; denna session är ansluten.
- Den aktiva SELinux-policyfilen och andra processers filbeskrivare kunde inte läsas som ADB-shell. Därför går det inte att bekräfta vilka appdomäner eller appar som faktiskt får öppna/använder Mali-enheten.

## 1. Användare, grupper och processer

Android visar en enda startad användare, `UserInfo 0`; systemet stöder upp till två användare. `/system/etc/passwd` och `/system/etc/group` finns inte i denna build, så en fullständig katalog över alla systemkonton kunde inte hämtas. `ps` visar konton som faktiskt kör processer.

ADB-shell kör som UID/GID `2000(shell)`, SELinux-domän `u:r:shell:s0`. Dess kompletterande grupper är `input` (1004), `log` (1007), `adb` (1011), `sdcard_rw` (1015), `sdcard_r` (1028), `net_bt_admin` (3001), `net_bt` (3002), `inet` (3003), `net_bw_stats` (3006) och `readproc` (3009). Konton som `root` (0), `system` (1000), `graphics` (1003) och `media` (1013) kunde upplösas av `id`.

`ps` gav en ögonblicksbild av processerna. Exempel per klass:

- **Root:** `/init`, `ueventd`, `vold`, `lmkd`, `netd`, `debuggerd` och leverantörsprocesser. Kärntrådar kör också som `root`; Mali-relaterade namn inkluderar `kbase_job_fault` och flera `kbase_event`.
- **System:** `system_server`, `surfaceflinger`, `servicemanager`, `audioserver`, `cameraserver` samt Sony-/MediaTek-tjänster.
- **Shell:** `adbd`-processen kör med användarnamnet `shell`, liksom ADB-skal och diagnostikkommandon.
- **App-UID:er:** exempelvis `u0_a146` för `com.netflix.ninja`, `u0_a57` för `com.android.systemui`, samt många Sony- och Google-appar med separata `u0_aNN`-UID:er. Processlistan visar även `system`-privilegierade appar.

Exempel på SELinux-domäner som `ps -Z` visade är `init`, `kernel`, `shell`, `adbd`, `surfaceflinger`, `system_server`, `platform_app`, `priv_app`, `untrusted_app`, `system_app` och `sony_platform_app`. Detta bekräftar att processerna är separerade i domäner, men säger inte ensamt vilka domäner som får använda GPU:n.

## 2. Mali-enheten och appåtkomst

| Sökväg | Resultat |
|---|---|
| `/dev/mali0` | Finns; `crw-rw-rw-`, `root:root`, etikett `u:object_r:gpu_device:s0`, major/minor `10:49` |
| `/dev/mali` | Finns inte |

Läget 0666 innebär att vanliga Unix/DAC-rättigheter inte begränsar läs-/skrivöppning till en särskild användare eller grupp. SELinux kan ändå neka öppning, läsning, skrivning eller ioctl efter processdomän. Att en nod är öppningsbar enligt DAC bevisar därför inte att varje app kan använda drivrutinen.

Android-appar får normalt inte GPU-åtkomst genom en särskild användarsynlig runtime-behörighet med namnet Mali. Åtkomsten avgörs i praktiken av device-node-rättigheter, SELinux-policy och grafikstacken. Vi kunde inte läsa `/sys/fs/selinux/policy` (permission denied) och kunde inte läsa FD-tabeller för exempelprocesser i `system_server`, `surfaceflinger` eller app-processer (permission denied). En sökning efter Mali-FD:er gav ingen träff, men på grund av dessa nekanden går det **inte** att dra slutsatsen att inga processer använder enheten.

**Slutsats:** ingen viss Android-app kunde bekräftas ha eller sakna faktisk Mali-åtkomst. Den breda DAC-rättigheten är bekräftad; den effektiva SELinux-auktoriseringen är inte verifierad.

## 3. SELinux

Bekräftat:

- `getenforce` returnerar `Enforcing`.
- `/dev/mali0` etiketteras `u:object_r:gpu_device:s0`.
- ADB-skalets processkontext är `u:r:shell:s0`; `ps -Z` visar skilda domäner för plattformstjänster och appar.

Ej verifierat:

- Exakta `allow`-/`neverallow`-regler för `gpu_device` och `chr_file`.
- Om `untrusted_app`, `platform_app`, `priv_app` eller Sonys domäner får `open`, `read`, `write` eller `ioctl` på Mali-enheten.

Aktiv policy under `/sys/fs/selinux/policy` nekades vid läsning från shell. Någon läsbar policykällfil hittades inte på de kontrollerade standardplatserna. Ingen root-åtkomst försöktes.

## 4. Andra observerade enheter och drivrutiner

| Enhet | DAC-ägare/grupp och läge | SELinux-etikett |
|---|---|---|
| `/dev/binder` | `root:root`, 0666 | `binder_device` |
| `/dev/ashmem` | `root:root`, 0666 | `ashmem_device` |
| `/dev/ion` | `system:system`, 0666 | `ion_device` |
| `/dev/random`, `/dev/urandom` | `root:root`, 0666 | `random_device`, `urandom_device` |
| `/dev/tty` | `root:root`, 0666 | `owntty_device` |
| `/dev/alarm` | `system:radio`, 0664 | `alarm_device` |
| `/dev/input/event*`, `js0`, `mice`, `mouse*` | `root:input`, 0660 | `input_device` |

`/dev/dri`, `/dev/kgsl` och `/dev/video0` saknas. Försök att lista hela `/dev`, `/dev/block`, `/dev/graphics` och flera interna sockets nekades, så detta är en riktad inventering, inte en komplett lista. ADB-shell ingår i gruppen `input`; vanliga appar behöver inte göra det. SELinux gäller även för noder med öppna DAC-rättigheter.

Andra intressanta observerade gränssnitt/processer är `adbd` på TCP 5555, `mdnsd`, Sonys nätverks-/webb-API-processer, browser-/Opera-processer och Samba `cidclient`. Tjänsten `reverseproxy` hade en `nginx`-process i processlistan samtidigt som `init.svc.reverseproxy` var `stopped`; ingen tjänst- eller portskanning utfördes och dess externa exponering är inte fastställd.

## 5. Android- och kernelbegränsningar

Bekräftade systemegenskaper och mount-flaggor:

- Android 7.0, SDK 24; build `user`, `release-keys`, säkerhetspatch `2019-02-01`.
- `ro.secure=1`, `ro.debuggable=0`, `security.perf_harden=1`.
- SELinux enforcing.
- `/proc` är monterat med `hidepid=2`; processmetadata kunde listas, men öppna FD-tabeller för andra UID:er nekades.
- ADB-shells `/proc/self/status` visade `Seccomp: 0`. Detta gäller endast den kontrollerade shell-processen; seccomp-läge för appar och systemtjänster kunde inte verifieras.
- `/system`, `/linux_rootfs` och `/3rd` är skrivskyddade squashfs-mounts. `/data`, `/cache`, `/3rd_rw` och `/upgrade` är skrivbara för processer med tillräckliga behörigheter; inga skrivförsök gjordes.
- `ro.crypto.state=unsupported`; faktisk kryptering/dataskydd av lagrade användardata har inte bedömts.
- Bootloader-egenskapen `ro.boot.flash.locked=1`; `ro.oem_unlock_supported=1`. Ingen bootloader- eller upplåsningsåtgärd utfördes.

## 6. ADB-åtkomst

Bekräftat via ADB och systemegenskaper:

- `init.svc.adbd=running`.
- `service.adb.tcp.port=5555`, `sys.usb.config=adb`.
- `ro.adb.secure=1`: ADB-authentication är aktiverad. Den aktuella ADB-klienten är ansluten; autentiseringens historik/nyckelhantering kontrollerades inte separat.
- Enheten svarar på `<private-TV-IP>:5555`. Adressen `<private-TV-IP>:5555` var offline vid kontrollen.

Detta innebär att en ADB-tjänst är tillgänglig över nätverket på port 5555 i den aktuella konfigurationen. Autentisering är ett skydd men eliminerar inte risk från betrodda/komprometterade hostnycklar eller nätverksexponering. Ingen annan värd eller port än den angivna enheten undersöktes.

## 7. Intressanta fynd

1. **Mali-enheten är DAC-världsskrivbar.** `/dev/mali0` är 0666. SELinux kan minska effektiv åtkomst, men regeln kunde inte läsas. Detta är det viktigaste bekräftade angreppsytfyndet.
2. **Brett åtkomliga systemgränssnitt.** Binder, ashmem och ION är också 0666 enligt DAC. De används av Androids IPC-/grafikstack och ska bedömas tillsammans med SELinux-policy.
3. **Gammal patchnivå.** Android 7.0/API 24 med patchnivå från februari 2019 är en tydlig underhålls-/exponeringsrisk. Denna kartläggning fastställer inte vilka enskilda CVE:er som är exploaterbara.
4. **Nätverks-ADB är aktivt.** Port 5555 och `adbd` är aktiva samtidigt som ADB-authentication är påslagen.
5. **Mali-drivrutin syns i kärnan.** `kbase_job_fault` och `kbase_event`-trådar finns. Drivrutinsversion, patchstatus och CVE-2023-4211:s tillämplighet verifierades inte i denna avläsning; CVE:n är tidigare uppgift från beställaren.

## 8. Rekommendationer

- Behandla 0666 på `/dev/mali0` som prioriterad granskningspunkt. Läs SELinux-policy från en auktoriserad firmware-/policykopia och fastställ åtkomst per domän utan att testa ioctl eller försöka krascha drivrutinen.
- Använd en firmware-/källkodskopia för att kontrollera Mali-kärndrivrutinens exakta version och CVE-2023-4211-fixstatus. Bekräfta patchnivå mot tillverkarens build-information.
- Begränsa nätverksåtkomst till ADB-port 5555 till ett isolerat, betrott administrationsnät; stäng av nätverks-ADB när den inte behövs, genom ordinarie TV-inställning/administration.
- Inventera öppna Mali-FD:er med en betrodd diagnostikmetod som inte kräver att ändra enheten; nuvarande ADB-shell hindras av procfs-/SELinux-åtkomst.
- Fortsätt endast med passiv inventering av fler enhetsnoder och tjänster. Undvik att öppna specialenheter eller anropa drivrutinsgränssnitt eftersom sådana operationer kan ha sidoeffekter.

## Begränsningar och säkerhetsgrad

ADB-resultat ovan är direkt insamlade från enheten och bekräftade vid tidpunkten för körningen. Flera slutsatser är medvetet begränsade: faktisk appåtkomst till Mali, exakta SELinux-regler, fullständig systemkontolista, samtliga enhetsnoder, process-FD:er, seccomp för andra domäner och CVE-2023-4211:s tillämplighet kunde inte verifieras utan privilegier eller otillgängliga policy-/procfs-data. Root-krävande kommandon hoppades över.
