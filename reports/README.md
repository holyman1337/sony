# Publicerade rapporter

Rapporterna här är textbaserade dokumentationskopior från Sony KD-55XD9305-projektet. Äldre firmware-/SELinux-rapporter är märkta som historiska där senare evidens ändrat slutsatsen. VM-TV-rapporter ligger separat och gäller en generisk x86-emulator, inte den fysiska Sony-TV:n.

Lokala firmwarebinärer, APK:er, kernelmoduler, images, extraktionskataloger och råa device dumps har inte publicerats. Privata TV-IP-adresser och lokala användarsökvägar har maskerats i publika rapportkopior. Hashar, firmwareversioner, symboler och statiska analysresultat finns kvar för spårbarhet.

## Sony / firmware

- [Fas 1 – TV-inventering](Fas-1-TV-inventering.md)
- [Historisk initial SELinux-analys](Historisk-initial-SELinux-analys.md)
- [Historisk firmwareidentifiering](Historisk-firmwareidentifiering.md)
- [Firmwareextraktion och SELinux/Mali-policy](sony-5433-firmware-extraction-and-policy.md)
- [CVE-2023-4211 Mali-fixjämförelse](sony-5433-cve-2023-4211-google-fix-comparison.md)
- [Praktisk Sony runtime-validering](sony-tv-practical-security-validation.md)
- [Fas 3C – statisk firmwareanalys](Fas-3C-Static-Firmware-Analysis.md)

## VM-TV (generisk x86-emulator)

- [Fas 2 – fördjupad emulering](vm-tv-lab/sony-vm-tv-phase2.md)
- [Avbrutet emulator-test](vm-tv-lab/sony-vm-tv-phase3-security-test.md)
- [Fas 3A – återställ testanslutning](vm-tv-lab/sony-vm-tv-phase3a.md)
- [Fas 3B – runtime audit](vm-tv-lab/sony-vm-tv-phase3b.md)

VM-materialet är separerat från Sony-firmwarefynd och innehåller inga slutsatser om den fysiska TV:n.
