# Sony VM-TV lab

## Status

En körbar, generisk Android TV API 24-miljö finns. Den är användbar som återställningsbar Android-miljö och för ofarliga, isolerade analyser av tjänster, paket, filrättigheter och gränssnitt. Den emulerar **inte** Sony KD-55XD9305, MT5890 eller Mali-T624.

Den verifierade Sony-firmwaren och dess extraherade komponenter ligger separat i `work/sony-5433-extracted/` och är endast analysunderlag. Proprietära Sony-HAL:ar och firmwarepartitioner är inte inkopplade i gästen; emulatorns generiska systemimage kan inte starta Sonys boot-/systemkomponenter direkt.

## Miljö

- Android Emulator 37.2.13 (QEMU2), Android TV system image `android-24;android-tv;x86` revision 22.
- AVD: `SonyVM_API24_TV_x86`, Android 7.0, API 24, ABI x86.
- Lokal ADB server: port 5038; emulatorportar 5554/5555. Startskriptet kräver loopback-only listeners och lokal ADB-enhet.
- Windows Firewall har aktiva utgående blockregler för emulator- och QEMU-programmen mot alla destinationer utom loopback. Gästen har ett virtuellt NIC och NAT-konfiguration, så isoleringen bygger på Windows Firewall-reglerna och ADB:s loopback-bindning; `-qemu -net none` tog inte bort gästens virtuella nätverkskort vid tidigare kontroll. Ta därför inte bort isoleringsreglerna medan VM körs.
- ARMv7 API 24 image finns installerad. Android Emulator 32.1.15 saknar ARM-engine (`PANIC: Missing emulator engine program for 'arm' CPU`), och aktuell emulator 37 stöder inte ARM. ARM-gästen startade alltså inte.

## Start, stopp och återställning

Kör PowerShell i projektkatalogen:

```powershell
.\outputs\sony-vm-tv-lab\Start-VM-TV.ps1
.\outputs\sony-vm-tv-lab\Stop-VM-TV.ps1
.\outputs\sony-vm-tv-lab\Rollback-VM-TV.ps1
```

Lägg till `-Headless` på startkommandot för att dölja emulatorfönstret. Start kräver att brandväggsreglerna är aktiva; om de saknas, kör `Enable-NetworkIsolation.ps1` som administratör. Ta bort dem endast efter att VM har stoppats: `Remove-NetworkIsolation.ps1` kräver administratör. Rollback ersätter gästernas användardata med rena API 24-bilder och tar bort snapshots/cache; den rör inte Sonys källfirmware.

Skript använder projektets SDK och ADB på port 5038. De rör inte global ADB-port 5037 eller den fysiska TV:n. En redan existerande värdprocess (PID 11344, ADB-port 5037) hade en TCP-anslutning till TV:ns ADB-port under inventeringen; den användes inte av VM-skripten och inga kommandon skickades till den. Kör inte `adb devices` utan `-P 5038` när fysisk-TV-anslutningen kan finnas i en annan ADB-server.

## Validerade egenskaper för den startade gästen

- `sys.boot_completed=1`
- Android 7.0, API 24, ABI x86
- Generisk fingerprint: `generic_x86/sdk_google_atv_x86/generic_x86:7.0/NYC/6696031:userdebug/test-keys`
- Kernel `3.10.0+`, inte Sonys 3.10.79
- SELinux Enforcing
- Ethernet `10.0.2.15`; ingen default route visades av gästens `ip route`-kontroll. Windows-reglerna för respektive emulatorprogram blockerar ändå utgående trafik utanför loopback.
- ADB, emulator-5554 local-only; hostens emulator-/ADB-lyssnare var endast `127.0.0.1` eller `::1`.

## Säker användning och begränsningar

Tillåtna laboratorieaktiviteter: statisk analys av originalfirmware, läsande inspektion av gästens Android-tjänster/paket/konfiguration, inventering av filrättigheter och SELinux i gästen, samt ofarliga åtkomstkontroll- och autentiseringstester mot isolerade lokala komponenter. Kontrollera att brandväggsreglerna fortfarande är aktiva före varje start.

Miljön kan inte verifiera Mali-SELinux-åtkomst, Sony-grafikstack eller CVE-2023-4211 vid körning eftersom gästen saknar Mali-T624, Sony-kernel och Sony-HAL. Kör inte exploit-, fuzz-, ioctl-, mmap- eller race-tester. Nätverksstudier beskriver bara den generiska gästen, inte TV:n.

## Spårbarhet

Nedladdade verktygsarkiv SHA256:

- Google command-line tools: `90AE805D20434428BFFCB699C290860F19BB5F66A67E6B330067E3DE801FB04A`
- Microsoft JDK 17: `49154292866386B2F4F09A3ECF3BE07AB80A419149C84EC1D4419B9EEC9E4B64`
- Android Emulator archive 32.1.15: `8B16F7E802031C45285339FCFD3B2CEBC4BF9D935CF46C6B6C5DB8E7B8FE22E8`
- API 24 x86 Android TV base `userdata.img`: `1C55446E1F85DE01A67EB2446FA7B5413324529045619B73971890BF3F2AA705`

Hashes för Sony-firmware och extraherade Sony-komponenter finns kvar i den tidigare rapporten `../sony-5433-firmware-extraction-and-policy.md`.




## Fas 2

- [Fördjupad emulering och kompatibilitetsanalys](sony-vm-tv-phase2.md)
- [Statisk Sony 5.433 build.prop-profil](sony-kd-55xd9305-5.433.profile.properties) — endast referens, inte avsedd att aktiveras i gästen.

- [Fas 3 praktiskt säkerhetstest](sony-vm-tv-phase3-security-test.md) — avbrutet före ADB/guest-kommandon eftersom projektservern på port 5038 inte redan körde.
