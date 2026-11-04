# Sony VM-TV – Fas 3B: Runtime Security Audit

Datum: 2026-11-04\
Mål: `emulator-5554` via projektets ADB-server `-P 5038`\
Resultat: **Runtime-konfigurationen samlades in. Inga bekräftade exploaterbara sårbarheter identifierades. Två debug-/diagnostikinställningar är hårdningspunkter, men väntade i en generisk userdebug-emulator. Inget av resultaten beskriver Sony-firmware.**

## 1. Målverifiering

Kommando:

```text
& '.\work\vm-tv\sdk\platform-tools\adb.exe' -P 5038 devices -l
```

Utdata:

```text
List of devices attached
emulator-5554          device product:sdk_google_atv_x86 model:sdk_google_atv_x86 device:generic_x86 transport_id:1
```

Exakt en enhet visades: `emulator-5554 device`. Alla följande ADB-kommandon använde projektets `work/vm-tv/sdk/platform-tools/adb.exe`, `-P 5038` och `-s emulator-5554`. Global ADB på 5037 och den fysiska TV:n användes inte. Ingen inställning, fil eller VM-konfiguration ändrades.

## 2. Android och säkerhetskonfiguration

| Kommando | Resultat | Säkerhetsmässig betydelse | Status / relevans |
|---|---|---|---|
| `getprop ro.build.version.release` | `7.0` | Android-version | Bekräftat, generisk emulator |
| `getprop ro.build.version.sdk` | `24` | API-nivå | Bekräftat, generisk emulator |
| `getprop ro.build.fingerprint` | `generic_x86/sdk_google_atv_x86/generic_x86:7.0/NYC/6696031:userdebug/test-keys` | Visar uttryckligen generisk x86 `userdebug`-build med test keys | Bekräftat; inte Sony |
| `getprop ro.debuggable` | `1` | Debug-build; ökar tillgängligheten för diagnostik jämfört med production `user`-build | Bekräftad debugkonfiguration, väntad i emulatorn; inte i sig en sårbarhet |
| `getprop ro.secure` | `1` | Säkerhetsproperty för Android userspace/adbd-konfiguration | Bekräftat; innebär inte att gästen är production-härdad |
| `getprop ro.adb.secure` | tom sträng | Propertyn saknas/är tom i den här imagen; går inte ensam att översätta till ett säkert besked om autentiseringsläge | Bekräftat tomt, tolkning okänd; generisk emulator |
| `getenforce` | `Enforcing` | SELinux-policy upprätthålls i gästen | Bekräftat; det är emulatorns policy, inte Sony-policy |
| `id` | `uid=2000(shell) gid=2000(shell)`; groups `input`, `log`, `adb`, `sdcard_rw`, `sdcard_r`, nätverksgrupper, `readproc` (`3009`); context `u:r:shell:s0` | ADB-sessionen är shell, inte root; kompletterande grupper ger diagnostik-/systemfunktioner | Bekräftat, generisk emulator |
| `uname -a` | `Linux localhost 3.10.0+ ... i686` | Gästkärnan är generisk x86 3.10.0+, inte Sony 3.10.79/ARM | Bekräftat; ingen Sony-kerneltestning |
| `ps` | 106 processer i stickprovet: 69 `root`, 6 `system`, 3 `shell`, 17 app-UID-processer (`u0_a*`) och övriga separata daemon-UID:er | UID-separation syns mellan systemdaemoner och app-processer; root-räkningen inkluderar många kerneltrådar | Bekräftat, generisk emulator |

Exempel från processlistan: `/init`, `vold`, `netd`, `zygote` och kerneltrådar kör som root; `servicemanager`, `surfaceflinger` och `system_server` kör som `system`; `adbd` kör som `shell`; appar såsom `com.android.systemui` kör under app-UID `u0_a23`. Detta är observerade UID:er, inte en full SELinux-domänanalys.

## 3. Applikationer, UID:er och behörigheter

### Paketinventering

| Kommando | Resultat | Säkerhetsmässig betydelse | Status / relevans |
|---|---|---|---|
| `pm list packages -U` | `Error: Unknown option: -U` | Den begärda `pm`-flaggan stöds inte av denna Android 7-version | Bekräftad verktygsbegränsning |
| `pm list packages -f` | 61 paket totalt; APK-sökvägar i `/system` | Paket från systemimagen | Bekräftat |
| `pm list packages -s -f` | 61 systempaket | Alla inventerade paket markeras som systempaket | Bekräftat |
| `pm list packages -3 -f` | Ingen utdata | Inga tredjepartspaket installerade i denna VM-körning | Bekräftat för denna snapshot |
| `dumpsys package packages` | Paketens `userId`, `sharedUser`, `codePath`, `privateFlags` och installerade permissions kunde läsas | Alternativ källa för UID och privilegieattribut efter att `-U` avvisades | Bekräftat, generisk emulator |
| `pm list permissions -g -d` | Listade dangerous permissions per permission group, bland annat contacts, phone, calendar, camera, location, storage, microphone, SMS och `READ_TV_LISTINGS` | Visar definierade runtime-permission-grupper, inte vilka appar som beviljats varje permission | Bekräftat; inga grants sluts från detta kommando |

`pm list packages -f` rapporterade 36 APK-sökvägar under `/system/priv-app`, 24 under `/system/app` och en framework package-sökväg; package dump markerade 37 poster med `PRIVATE PRIVILEGED`. Skillnaden beror på att path-kategori och `PRIVILEGED`-flagga är olika mått, inte automatiskt en inkonsekvens.

### UID-/sandboxobservationer

`dumpsys package packages` visade separata app-ID:n för vanliga systemimage-appar och avsiktliga shared-user-grupper, bland annat:

- UID 1000 (`android.uid.system`): exempelvis `android`, SettingsProvider, InputDevices och TV Settings.
- UID 1002: Bluetooth och TV Remote Pairing.
- UID 10005: ContactsProvider och UserDictionaryProvider.
- UID 10008: MediaProvider och DownloadProvider.
- UID 10011: flera Google/GMS-komponenter.
- `com.android.shell`: UID 2000.

Dessa delade UID-grupper är synliga i systemimagen och framstår som avsiktliga Android system/shared UID-upplägg; inget separat tredjepartspaket fanns att jämföra med dem. Ingen isoleringsgräns kring en vanlig tredjepartsapp kunde därför praktiskt testas.

Exempel från PackageManager-dumpen: `com.google.android.katniss` låg under `/system/priv-app`, hade `PRIVATE PRIVILEGED` och beviljade privilegier såsom `WRITE_SETTINGS`, `WRITE_SECURE_SETTINGS`, `STATUS_BAR` och `CAPTURE_AUDIO_OUTPUT`. Det är en privilegierad systemapp i den generiska Google TV-imagen; beviljandena innebär inte att en normal app har dessa rättigheter. Det är en komponent med hög tillit och därmed relevant att hålla uppdaterad i en produktionsimage.

**Appdata:** Inga privata appfiler öppnades eller lästes. Ingen testapp installerades. Åtkomst från en normal app till en annan apps data via tillåtna gränssnitt är därför **ej testad**. Shell-åtkomst till katalogmetadata ersätter inte app-till-app-test.

## 4. Android-tjänster / Binder

Kommando: `service list`

Resultat: `Found 109 services`. Exempel i listan var `activity`, `package`, `permission`, `appops`, `connectivity`, `network_management`, `tv_input`, `gpu`, `SurfaceFlinger`, `media.codec`, `media.audio_flinger`, `keystore` och `gatekeeper`.

Kompletterande PackageManager-läsningar:

| Kommando | Resultat | Betydelse / begränsning |
|---|---|---|
| `dumpsys package com.android.systemui` | Service Resolver Table listar `com.android.systemui/.DessertCaseDream` för `android.service.dreams.DreamService`; paketdumpen visar bland annat `STATUS_BAR_SERVICE` och `ACCESS_VOICE_INTERACTION_SERVICE` bland appens permissionposter | Bekräftar systemkomponent/permissionmetadata, men den fångade dumpen binder inte entydigt dessa permissionposter till en specifik Binder-metod |
| `dumpsys package com.google.android.katniss` | Resolvern listar `KatnissRecognitionService` (`android.speech.RecognitionService`) och `KatnissVoiceInteractionService` (`android.service.voice.VoiceInteractionService`) | Bekräftar deklarerade servicekomponenter i en privileged app; ingen anropsåtkomst testades |
| `dumpsys package com.android.tv` | Resolvern listar `UsbTunerTvInputService`; dumpen listar komponenten under `disabledComponents` | Komponenten finns i package metadata men är markerad disabled i denna runtime |

`service list` redovisar registrerade Binder-servicenamn och interface-deskriptorer, inte metodernas permission checks eller om varje metod är åtkomlig för en vanlig app. PackageManager-dumparna ger komponent- och permissionmetadata men bevisar inte att en vanlig app kan anropa en skyddad service. Inga `service call`/Binder-transaktioner eller appanrop gjordes. Metodspecifika åtkomstkrav kunde därför inte fullständigt fastställas; inga obehöriga serviceåtkomster är bekräftade. Allt detta gäller generisk AOSP/Google TV Android, inte Sony.
## 5. Nätverk

| Kommando | Resultat | Säkerhetsmässig betydelse | Status / relevans |
|---|---|---|---|
| `cat /proc/net/tcp` | Endast rubrik; inga TCP-socketposter vid mättillfället | Inga IPv4 TCP-sockets listades i gästens namespace | Bekräftat stickprov, generisk emulator |
| `cat /proc/net/tcp6` | Endast rubrik; inga TCP6-socketposter | Inga IPv6 TCP-sockets listades i gästens namespace | Bekräftat stickprov, generisk emulator |
| `ip addr` | `lo` 127.0.0.1; `eth0` 10.0.2.15/24; `wlan0`, `wlan1`, `hwsim0`, `sit0` DOWN | Gästen har ett virtuellt Ethernetinterface; Wi-Fi-liknande interface är inaktiva | Bekräftat |
| `ip route` | Endast `10.0.2.0/24 dev eth0 ...`; ingen default route visades | Ingen default IPv4-route i detta gästresultat | Bekräftat för tabellen vid mättillfället; inte en extern reachability-test |

Ingen extern portskanning, anslutning eller autentiseringstest utfördes. TCP-tabellerna visar gästnamespace, inte hostens ADB-port 5038. Hostens loopback-bindning och brandväggsregler hanterades inte om i denna fas.

## 6. Filsystem och mountar

| Kommando | Resultat | Säkerhetsmässig betydelse | Status / relevans |
|---|---|---|---|
| `mount` | `/` är `rootfs` read-only; `/system` ext4 `ro`; `/data` ext4 `rw,nosuid,nodev`; `/cache` ext4 `rw,nosuid,nodev`; `debugfs` på `/sys/kernel/debug` är `rw`; `/proc` har `hidepid=2,gid=3009` | Systempartitionen är skrivskyddad; användardata/cache är skrivbara som normalt; debugfs är monterad skrivbart; processinformation skyddas med hidepid men shell har gruppen `readproc` | Bekräftat; debugfs är en diagnostikyta, åtkomst till dess enskilda filer testades inte |
| `df -h` | `/system` 2.4G total, 788M använt, 1.6G ledigt; `/data` 10G total, 89M använt; `/cache` 61M | Kapacitetstillstånd, inget avvikande identifierat | Bekräftat |
| `ls -ldL /system /data /vendor` | `/system` `root:root` 0755; `/data` `system:system` 0771; `/vendor` följer symlink och visar `root:shell` 0755 | `/system` och vendorinnehåll är läsbara men mountade read-only; `/data` har begränsad katalogåtkomst via mode 0771 | Bekräftat |
| `ls -ld /system/app /system/priv-app /system/vendor` | `/system/app` `root:root` 0755; `/system/priv-app` `root:root` 0755; `/system/vendor` `root:shell` 0755 | System-/priv-app-installationskataloger ligger i systempartitionen; `/vendor` är del av `/system`, inte separat mount | Bekräftat |

Inga privata appdata söktes igenom. `ls` kontrollerade endast katalogmetadata som efterfrågades.

## 7. Kommandon som inte fungerade eller inte kunde fastställa frågan

| Kommando / fråga | Resultat och hantering |
|---|---|
| `pm list packages -U` | Android 7 PackageManager CLI svarade `Unknown option: -U`. UID-information hämtades i stället ur `dumpsys package packages`. |
| `ps -A` | Androids `ps` svarade `bad pid '-A'`. Vanligt `ps` fungerade och gav process-/UID-tabellen. |
| Exakt tjänstepermission och åtkomst från normal app | `service list` visar service/interface men inte metodernas access checks. Ingen Binder-transaktion eller normal-app-harness kördes. Ej fastställt. |
| App-till-app-åtkomst till privat data | Inga tredjepartsappar fanns installerade; ingen testapp installerades. Ej testat. |
| Extern nätverksreachability/autentisering | Inga anslutningsförsök eller portskanningar gjordes. Ej testat. |
| ADB secure/auth-status på gästegenskap | `ro.adb.secure` var tom. Ingen slutsats om autentiseringspolicy drogs enbart från propertyn. |
| Sony-specifika runtime-egenskaper eller Mali | AVD är generic x86 utan Sony-kernel/HAL/Mali-T624. Ej tillämpligt på denna runtime. |

## 8. Prioriterade säkerhetsproblem och bedömning

1. **Låg till måttlig hårdningspunkt – debug-build (`ro.debuggable=1`, `userdebug/test-keys`).** Bekräftad egenskap i Android TV AVD-image. Det är väntat i en utvecklar-/testemulator och inte en bekräftad exploaterbar sårbarhet. Den ska inte förväxlas med en production-Android TV build.
2. **Måttlig granskningspunkt – `debugfs` monterat `rw`.** Bekräftad mountflagga i gästen. Om den är åtkomlig från lägre privilegier kan debugfs exponera diagnostikgränssnitt, men inga filer eller skrivmöjligheter testades. Detta är en möjlig lokal attackyta i generisk userdebug-kernel, inte bevis på en sårbarhet.
3. **Låg/okänd – `ro.adb.secure` tom.** Propertyn är tom i gästen; autentiseringsbeteendet fastställdes inte. Inga ADB-anslutningar utöver den redan verifierade lokala emulatortransporten testades.
4. **Ingen bekräftad app-sandbox- eller Binder-åtkomstsårbarhet.** Det saknas tredjepartsapp i imagen och inga metodanrop gjordes. Frånvaron av en testapp innebär att isoleringen inte har praktiskt validerats mellan två vanliga appar.
5. **Ingen Sony/Mali-sårbarhet testades.** Alla bekräftade runtime-observationer här gäller generisk Android 7 API 24 x86. De kan inte bekräfta eller motbevisa firmware-/kernel-/Mali-fynden för KD-55XD9305.

### Slutbedömning

Fas 3B bekräftade en generisk Android 7 userdebug-emulator med SELinux Enforcing, read-only systempartition, shell-UID 2000 och PackageManager-separation mellan system-/app-UID:er. Inga bekräftade sårbarheter hittades. Testet identifierade väntade utvecklarinställningar och en skrivbart monterad debugfs-yta som bör ses som hårdnings-/granskningspunkter i emulatorimagen. App-till-app-isolering, Binder-metodbehörigheter och extern nätverksreachability är inte bevisade genom denna körning.

Inga av dessa resultat fastställer säkerheten i Sony-firmwaren. Fortsatt analys av Sony kräver separata firmwarebaserade statiska slutsatser eller en separat matchande laboratorieenhet; den fysiska TV:n användes inte.


## Appendix A. Full package inventory

Källor: `pm list packages -f` för paket/sökväg och `dumpsys package packages` för `userId`. Detta är PackageManager-metadata från denna generiska AVD-snapshot; delade UID:er visas likadant för varje paket.

| Paket | UID | APK-/framework-sökväg |
|---|---:|---|
| `com.android.cts.priv.ctsshim` | 10006 | `/system/priv-app/CtsShimPrivPrebuilt/CtsShimPrivPrebuilt.apk` |
| `com.google.android.ext.services` | 10012 | `/system/priv-app/GoogleExtServices/GoogleExtServices.apk` |
| `com.google.android.katniss` | 10014 | `/system/priv-app/Katniss/Katniss.apk` |
| `com.android.providers.telephony` | 1001 | `/system/priv-app/TelephonyProvider/TelephonyProvider.apk` |
| `com.android.providers.calendar` | 10001 | `/system/priv-app/CalendarProvider/CalendarProvider.apk` |
| `com.android.tv.settings` | 1000 | `/system/priv-app/TvSettings/TvSettings.apk` |
| `com.lego.android.tvleanback` | 10037 | `/system/app/LeanbackSampleApp/LeanbackSampleApp.apk` |
| `com.android.providers.media` | 10008 | `/system/priv-app/MediaProvider/MediaProvider.apk` |
| `com.google.android.ext.shared` | 10033 | `/system/app/GoogleExtShared/GoogleExtShared.apk` |
| `com.android.externalstorage` | 10010 | `/system/priv-app/ExternalStorageProvider/ExternalStorageProvider.apk` |
| `com.android.htmlviewer` | 10035 | `/system/app/HTMLViewer/HTMLViewer.apk` |
| `com.android.providers.downloads` | 10008 | `/system/priv-app/DownloadProvider/DownloadProvider.apk` |
| `com.android.providers.tv` | 10024 | `/system/priv-app/TvProvider/TvProvider.apk` |
| `com.google.android.pano.packageinstaller` | 10002 | `/system/priv-app/CanvasPackageInstaller/CanvasPackageInstaller.apk` |
| `com.google.android.backdrop` | 10026 | `/system/app/Backdrop/Backdrop.apk` |
| `com.google.android.configupdater` | 10004 | `/system/priv-app/ConfigUpdater/ConfigUpdater.apk` |
| `com.android.defcontainer` | 10007 | `/system/priv-app/DefaultContainerService/DefaultContainerService.apk` |
| `com.android.pacprocessor` | 10039 | `/system/app/PacProcessor/PacProcessor.apk` |
| `com.android.certinstaller` | 10029 | `/system/app/CertInstaller/CertInstaller.apk` |
| `com.android.carrierconfig` | 10003 | `/system/priv-app/CarrierConfig/CarrierConfig.apk` |
| `com.google.android.marvin.talkback` | 10043 | `/system/app/talkback/talkback.apk` |
| `com.google.android.leanback.ime` | 10036 | `/system/app/LeanbackIme/LeanbackIme.apk` |
| `android` | 1000 | `/system/framework/framework-res.apk` |
| `com.android.backupconfirm` | 10000 | `/system/priv-app/BackupRestoreConfirmation/BackupRestoreConfirmation.apk` |
| `com.google.android.tv.remotepairing` | 1002 | `/system/priv-app/GamepadPairingService/GamepadPairingService.apk` |
| `com.android.statementservice` | 10022 | `/system/priv-app/StatementService/StatementService.apk` |
| `com.google.android.gsf.notouch` | 10011 | `/system/app/NoTouchAuthDelegate/NoTouchAuthDelegate.apk` |
| `com.android.providers.settings` | 1000 | `/system/priv-app/SettingsProvider/SettingsProvider.apk` |
| `com.android.sharedstoragebackup` | 10021 | `/system/priv-app/SharedStorageBackup/SharedStorageBackup.apk` |
| `com.android.printspooler` | 10040 | `/system/app/PrintSpooler/PrintSpooler.apk` |
| `com.android.dreams.basic` | 10027 | `/system/app/BasicDreams/BasicDreams.apk` |
| `com.android.tv` | 10016 | `/system/priv-app/LiveTv/LiveTv.apk` |
| `com.android.inputdevices` | 1000 | `/system/priv-app/InputDevices/InputDevices.apk` |
| `com.google.android.leanbacklauncher.recommendations` | 10019 | `/system/priv-app/RecommendationsService/RecommendationsService.apk` |
| `com.android.sdksetup` | 10020 | `/system/priv-app/SdkSetup/SdkSetup.apk` |
| `com.google.android.webview` | 10042 | `/system/app/WebViewGoogle/WebViewGoogle.apk` |
| `com.google.android.syncadapters.contacts` | 10032 | `/system/app/GoogleContactsSyncAdapter/GoogleContactsSyncAdapter.apk` |
| `com.android.keychain` | 1000 | `/system/app/KeyChain/KeyChain.apk` |
| `com.google.android.packageinstaller` | 10013 | `/system/priv-app/GooglePackageInstaller/GooglePackageInstaller.apk` |
| `com.android.emulator.smoketests` | 10031 | `/system/app/EmulatorSmokeTests/EmulatorSmokeTests.apk` |
| `com.google.android.gms` | 10011 | `/system/priv-app/PrebuiltGmsCorePano/PrebuiltGmsCorePano.apk` |
| `com.google.android.gsf` | 10011 | `/system/priv-app/GoogleServicesFramework/GoogleServicesFramework.apk` |
| `com.google.android.sss` | 10041 | `/system/app/SecondScreenSetup/SecondScreenSetup.apk` |
| `com.google.android.tts` | 10034 | `/system/app/GoogleTTS/GoogleTTS.apk` |
| `com.android.proxyhandler` | 10018 | `/system/priv-app/ProxyHandler/ProxyHandler.apk` |
| `com.google.android.tungsten.overscan` | 10017 | `/system/priv-app/Overscan/Overscan.apk` |
| `com.android.netspeed` | 10038 | `/system/app/NetSpeed/NetSpeed.apk` |
| `com.google.android.leanbacklauncher` | 10015 | `/system/priv-app/LeanbackLauncher/LeanbackLauncher.apk` |
| `com.google.android.backuptransport` | 10011 | `/system/priv-app/GoogleBackupTransport/GoogleBackupTransport.apk` |
| `com.google.android.sss.authbridge` | 10011 | `/system/app/SecondScreenSetupAuthBridge/SecondScreenSetupAuthBridge.apk` |
| `com.android.cts.ctsshim` | 10030 | `/system/app/CtsShimPrebuilt/CtsShimPrebuilt.apk` |
| `com.android.vpndialogs` | 10025 | `/system/priv-app/VpnDialogs/VpnDialogs.apk` |
| `com.android.shell` | 2000 | `/system/priv-app/Shell/Shell.apk` |
| `com.android.wallpaperbackup` | 1000 | `/system/app/WallpaperBackup/WallpaperBackup.apk` |
| `com.android.providers.userdictionary` | 10005 | `/system/app/UserDictionaryProvider/UserDictionaryProvider.apk` |
| `com.android.emergency` | 10009 | `/system/priv-app/EmergencyInfo/EmergencyInfo.apk` |
| `com.android.location.fused` | 1000 | `/system/priv-app/FusedLocation/FusedLocation.apk` |
| `com.android.systemui` | 10023 | `/system/priv-app/SystemUI/SystemUI.apk` |
| `com.android.bluetooth` | 1002 | `/system/app/Bluetooth/Bluetooth.apk` |
| `com.android.providers.contacts` | 10005 | `/system/priv-app/ContactsProvider/ContactsProvider.apk` |
| `com.android.captiveportallogin` | 10028 | `/system/app/CaptivePortalLogin/CaptivePortalLogin.apk` |
