# ThinkPad L390 – OpenCore EFI (branch `sequoia`: macOS Sequoia 15.x)

> **Status: BELUM DIUJI BOOT.** Branch ini diturunkan dari EFI Ventura 13.7.8 yang terbukti jalan (branch `main`). Tidak ada rilis Sequoia yang diinstall di mesin saat penyusunan, jadi anggap ini *kandidat*, bukan EFI yang sudah terbukti.

## Beda dari `main` (Ventura)

| Item | Ventura (`main`) | Sequoia (branch ini) |
|---|---|---|
| `AirportItlwm.kext` | aktif (build Ventura 2.3.0, dibatasi kernel 22.x) | **dinonaktifkan** |
| Lainnya | | identik |

### Kenapa Wi-Fi dimatikan
`AirportItlwm` yang ada adalah build **Ventura saja**. Rilis stabil resmi OpenIntelWireless/itlwm hanya menyediakan build sampai Sonoma 14.4; dukungan macOS 15 masih berupa permintaan terbuka ([itlwm#1009](https://github.com/OpenIntelWireless/itlwm/issues/1009)). Di Sequoia kext ini tidak akan load. Jalur komunitas yang beredar (tidak resmi, belum dipakai di repo ini):
- fork pihak ketiga `laobamac/itlwm` (build Sequoia, status alpha), atau
- `AirportItlwm 2.3.0` Ventura + OCLP Modern Wi-Fi root patch + spoof IOName Broadcom (butuh root patch, bukan EFI-only).

Selama Wi-Fi belum beres, internet lewat **Ethernet** (IntelMausiEthernet, I219-V) atau adapter USB.

### Cek kompatibilitas yang sudah dilakukan (statis, bukan boot)
- OpenCore **1.0.8** adalah rilis stabil terbaru dan mendukung macOS 15.
- Semua kext selain Wi-Fi sudah di versi sama atau lebih baru dari rilis terbaru upstream (Lilu 1.7.3, VirtualSMC/WhateverGreen/AppleALC terbaru, NVMeFix 1.1.4, VoodooPS2 2.3.8, BlueToolFixup 2.7.3, IntelBluetoothFirmware/BTPatcher 2.5.0).
- Tidak ada kext aktif yang dibatasi `MaxKernel` di bawah 24 (Sequoia = Darwin 24.x).
- Semua kext, SSDT, dan driver yang diaktifkan di `config.plist` ada di disk.
- `MacBookPro15,2` masih didukung Sequoia.

### Belum diverifikasi
`ocvalidate`, boot nyata, sleep, audio, touchpad (VoodooI2C/PS2), Bluetooth (BlueToolFixup), card reader (Sinetek-rtsx v9.0 dari 2021), dan YogaSMC (1.5.3 dari 2022) di Sequoia. Dua yang terakhir adalah kandidat paling mungkin bermasalah karena lama tidak diperbarui.

**Sebelum install:** simpan EFI Ventura yang jalan dan siapkan USB installer. Isi SMBIOS sendiri.

---

EFI OpenCore untuk Hackintosh **Lenovo ThinkPad L390**, berjalan di **macOS Ventura 13.7.8 (22H730)**.

> ⚠️ **Wajib** ganti SMBIOS (`SystemSerialNumber`, `MLB`, `SystemUUID`, `ROM`) di `EFI/OC/config.plist` dengan milikmu sendiri (GenSMBIOS / macserial). Nilai di repo ini hanya placeholder. Jangan pakai serial orang lain.

## Hardware

| Komponen | Detail |
|---|---|
| CPU | Intel Core i5-8265U (Whiskey Lake, 4C/8T, 1.6 GHz) |
| RAM | 16 GB |
| iGPU | Intel UHD Graphics 620, layar 1920x1080 |
| Storage | Micron NVMe `MTFDKBA512TFH` 512 GB |
| Ethernet | Intel I219-V (I219V6) |
| Wi-Fi | Intel (pakai `AirportItlwm`, firmware itlwm 46.4e1ceb39) |
| Bluetooth | Intel (pakai `IntelBluetoothFirmware` + `IntelBTPatcher` + `BlueToolFixup`) |
| Card reader | Realtek (`Sinetek-rtsx`) |
| Input | Keyboard + TrackPoint (PS/2), touchpad (PS/2 + I2C) |

## Versi

| Item | Nilai |
|---|---|
| OpenCore | **1.0.8** (`REL-108-2026-09-23`) |
| macOS | Ventura 13.7.8 |
| SMBIOS | `MacBookPro15,2` |
| SIP | Custom, `csr-active-config = 030a0000` (0x0A03: untrusted kexts, unrestricted FS, unapproved kexts, unauthenticated root) |

## Struktur

```
EFI/
├── BOOT/BOOTx64.efi
└── OC/
    ├── ACPI/        # 11 SSDT
    ├── Drivers/     # HfsPlus, OpenRuntime, ResetNvramEntry
    ├── Kexts/       # 25 kext
    ├── Resources/
    ├── OpenCore.efi
    ├── config.plist
    └── oldConfig.plist   # config lama sebelum upgrade OC (referensi)
```

Catatan layout: di mesin asli, `BOOT` dan `OC` ada di **root** partisi EFI (`/BOOT`, `/OC`), bukan `/EFI/OC`. Di repo ini dipindah ke layout standar `EFI/BOOT` + `EFI/OC`. Isinya sama persis (kecuali SMBIOS yang di-redact). Bootloader Windows (`EFI/Microsoft`, `EFI/Boot`) sengaja tidak disertakan.

## ACPI

| SSDT | Fungsi |
|---|---|
| `SSDT-PLUG` | CPU power management (XCPM) |
| `SSDT-EC` | Embedded controller palsu (syarat macOS) |
| `SSDT-USBX` | USB power properties |
| `SSDT-MCHC` | Perbaikan device SMBus/MCHC |
| `SSDT-SBUS` | SMBus |
| `SSDT-PMC` | NVRAM native (PMC) |
| `SSDT-RTCAWAC` | Perbaikan RTC / ACPI AWAC clock |
| `SSDT-PNLF` | Kontrol brightness backlight |
| `SSDT-ALS0` | Ambient light sensor |
| `SSDT-GPI0` | Mengaktifkan GPIO untuk VoodooI2C |
| `SSDT-XOSI` | Spoof `_OSI` (butuh patch rename `_OSI → XOSI`) |

### ACPI Patch (semua aktif)

1. `GPI0 _STA to XSTA` rename (dipasangkan dengan `SSDT-GPI0`)
2. `NBCF 0x00 to 0x01`
3. `PNLF to XNLF` rename (dipasangkan dengan `SSDT-PNLF`)
4. `_OSI to XOSI` rename (dipasangkan dengan `SSDT-XOSI`)

## Kext (urutan load di config)

| Kext | Versi | Fungsi |
|---|---|---|
| Lilu | 1.7.3 | Platform patching |
| VirtualSMC | 1.3.8 | Emulasi SMC |
| SMCProcessor / SMCSuperIO / SMCLightSensor / SMCBatteryManager | 1.3.8 | Sensor CPU, sensor lain, ALS, baterai |
| WhateverGreen | 1.7.1 | Patch GPU Intel |
| AppleALC | 1.9.8 | Audio (layout-id 100 di DeviceProperties) |
| NVMeFix | 1.1.4 | Power management NVMe |
| ECEnabler | 1.0.6 | Baca baterai 16-bit |
| BrightnessKeys | | Tombol brightness |
| IntelMausiEthernet | 3.0.3 | Ethernet I219 |
| AirportItlwm | 2.3.0 | Wi-Fi Intel |
| IntelBluetoothFirmware / IntelBTPatcher | 2.5.0 | Bluetooth Intel |
| BlueToolFixup | 2.7.3 | Bluetooth di Monterey+ |
| Sinetek-rtsx | 9.0.0 | Card reader |
| VoodooPS2Controller (+ Keyboard, Mouse, Trackpad) | 2.3.8 | Keyboard, TrackPoint, touchpad PS/2 |
| VoodooI2C + VoodooGPIO + VoodooI2CServices + VoodooInput | 2.9.1 | Bus I2C |
| VoodooI2CHID | 1.0 | Touchpad I2C |
| YogaSMC | 1.5.3 | Fitur Lenovo/ThinkPad (tombol Fn, dll) |
| USBToolBox + UTBMap + UTBDefault | UTBMap 1.1 | Pemetaan USB |
| XHCI-unsupported | 0.9.2 | USB xHCI |

(`VoodooInput` di dalam VoodooPS2 dinonaktifkan; yang dipakai dari VoodooI2C.)

## DeviceProperties

| Path | Properti |
|---|---|
| `PciRoot(0x0)/Pci(0x2,0x0)` (iGPU) | `AAPL,ig-platform-id = 00009B3E`, `device-id = 9B3E0000`, `framebuffer-patch-enable = 1`, `framebuffer-fbmem = 00009000`, `framebuffer-stolenmem = 00003001` |
| `PciRoot(0x0)/Pci(0x1f,0x3)` (audio) | `layout-id = 100` |

## boot-args dan NVRAM

```
-v debug=0x100 keepsyms=1 -igfxblt -vi2c-force-polling
```

- `-igfxblt`: perbaikan blitter iGPU
- `-vi2c-force-polling`: VoodooI2C pakai polling (touchpad tanpa interrupt GPIO)
- `-v`: verbose boot; `debug=0x100` + `keepsyms=1`: tidak auto-reboot saat panic, simpan simbol

`csr-active-config = 030a0000`, `run-efi-updater = No`.

## Booter Quirks (yang aktif)

`AvoidRuntimeDefrag`, `EnableSafeModeSlide`, `EnableWriteUnprotector`, `ProvideCustomSlide`, `SetupVirtualMap`.
Yang off: `DevirtualiseMmio`, `RebuildAppleMemoryMap`, `ProtectUefiServices`, `SyncRuntimePermissions`.

## Kernel Quirks (yang aktif)

`AppleXcpmCfgLock`, `CustomSMBIOSGuid`, `DisableIoMapper`, `DisableLinkeditJettison`, `PanicNoKextDump`, `PowerTimeoutKernelPanic`.
`XhciPortLimit = false` (port dipetakan lewat UTBMap, jadi limit tidak dibutuhkan).

## UEFI

- Driver: `HfsPlus.efi`, `OpenRuntime.efi`, `ResetNvramEntry.efi`
- Quirks aktif: `EnableVectorAcceleration`, `ReleaseUsbOwnership`, `RequestBootVarRouting`
- ProtocolOverrides: `FirmwareVolume`
- APFS: `EnableJumpstart`, `HideVerbose`, `MinDate/MinVersion = -1`
- Output: `ProvideConsoleGop`, `Resolution = Max`, `TextRenderer = BuiltinGraphics`

## Misc / Security

- `PickerMode = Builtin`, `ShowPicker = true`, `Timeout = 5`, `HideAuxiliary = true`
- `LauncherOption = Disabled`
- `SecureBootModel = Default`, `Vault = Optional`, `ScanPolicy = 0`, `DmgLoading = Signed`
- `Debug.Target = 3`
- Tools: tidak ada

## PlatformInfo

`UpdateSMBIOS`, `UpdateDataHub`, `UpdateNVRAM` aktif, mode `Custom`, `SystemProductName = MacBookPro15,2`. Serial, MLB, UUID, ROM di-redact.

## Status

Berjalan dengan baik di macOS Ventura 13.7.8. Yang terlihat dari sistem saat dokumentasi ini dibuat: iGPU UHD 620 dikenali 1080p, Wi-Fi (itlwm) terhubung, NVMe terbaca, Ethernet I219V terdeteksi, Lilu termuat.
Sleep, audio, brightness, touchpad gesture, dan Bluetooth belum diuji ulang satu per satu saat membuat repo ini.

## Cara pakai

1. Generate SMBIOS sendiri untuk `MacBookPro15,2` lalu isi `PlatformInfo/Generic` di `config.plist`.
2. Salin `EFI/` ke partisi EFI (`disk0s1`) dan jangan menimpa folder Windows kalau dual boot.
3. Reset NVRAM setelah ganti EFI.
4. Kalau pindah ke OpenCore versi lain, jalankan `ocvalidate` dan sesuaikan kext.

## Kredit

Terima kasih untuk: [OpenCore](https://github.com/acidanthera/OpenCorePkg), [Dortania guide](https://dortania.github.io/OpenCore-Install-Guide/), Acidanthera kexts, VoodooI2C, VoodooPS2, itlwm / AirportItlwm, IntelBluetoothFirmware, Sinetek-rtsx, YogaSMC, USBToolBox.

Pakai dengan risiko sendiri.
