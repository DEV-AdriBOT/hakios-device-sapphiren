# HakiOS on Xiaomi Redmi Note 13 4G NFC

**Codename:** `sapphiren`
**SoC:** Qualcomm Snapdragon 685 / SM6225 (Khaje)
**Repository:** [DEV-AdriBOT/hakios-device-sapphiren](https://github.com/DEV-AdriBOT/hakios-device-sapphiren)

## Current state

This is an experimental HakiOS port for the Xiaomi Redmi Note 13 4G NFC. The bootloader must be unlocked.

The port boots **Darwin 22.5.0 (XNU 8796.121.2) with iOS 16.5 user space** and reaches `launchd`, `backboardd` and **SpringBoard/the home screen**. The Darwin root and cryptexes can be read from the phone's `darwin` UFS partition, so the complete system does not need to be uploaded from the host on every boot.

The current state is deliberately conservative: hardware marked as experimental or untested below should not be considered production-ready.

| Component | State | Notes |
|---|---|---|
| Darwin / iOS | **Home screen** | Darwin 22.5.0 with iOS 16.5 user space reaches SpringBoard. |
| Display | **Works** | The panel is usable; the framebuffer path still needs performance work. |
| CPU/SMP | **Works** | All eight cores can run under the Khaje-specific XNU setup. |
| Touch | **Partly** | FocalTech FTS at I2C `0x38` works in tests, but cold-boot firmware loading and swipe reliability still need work. |
| GPU / graphics | **Partly** | Experimental Adreno 610 path and Mesa OpenGL ES rendering work. Rendering performance and full graphics integration are still poor. Metal is not complete. |
| UFS / storage | **Partly** | Read-only native UFS access and the `darwin` partition work experimentally. The U-Boot UFS command still hangs. |
| USB | **Partly** | U-Boot fastboot and the loader's USB device mode work; kernel USB is not complete. |
| Wi-Fi | **No** | The Khaje legacy-WLAN/MPSS path is being ported and has not been validated yet. |
| Bluetooth | **No** | Not implemented. |
| Audio | **No** | Not implemented. |
| Cellular | **No** | Modem bring-up is still pending. |
| Battery gauge | **Partly** | A battery level can be passed from Android; there is no native gauge driver yet. |
| Camera, haptics, thermal | **No** | Not implemented. |

## Boot requirements

- Unlocked bootloader.
- Android boot header version 4.
- The kernel Image must use `text_offset = 0` for this device's ABL.
- The Khaje build needs `HAKIOS_ARMV8_0=1`; the SoC is ARMv8.0 and does not provide the assumptions used by newer reference devices.
- The virtual timer is INTID 19 (`soc_vtimer=0x13`).

The device-specific loader and boot image work is in the common HakiOS kernel/UEFI repositories. This repository contains the device description, Khaje layer and device-specific configuration.

## Hardware notes

- Qualcomm Khaje / SM6225, Snapdragon 685.
- Eight ARM cores and UFS storage.
- Adreno 610 GPU, without the GMU path used by the reference Adreno 642L port.
- FocalTech touchscreen at I2C address `0x38`.
- Synopsys Femto USB2 PHY.
- The bootloader leaves a framebuffer at `0x5c000000`; the Android panel mode is 1080x2400. Darwin's UI path has also been tested at its native 1125x2436 composition size.

## Wi-Fi work in progress

The phone does not use the WCN6750/WPSS arrangement from the reference device. Its WLAN is part of the Khaje legacy WLAN path in the modem/MPSS firmware. The current work includes:

- Khaje MPSS address and PAS configuration.
- APCS, SMEM, SMP2P and GLINK bring-up experiments.
- Local, uncommitted firmware extraction for testing only.

The MPSS/WLAN path has not yet been validated end-to-end, so no Wi-Fi support is claimed here.

## Graphics work in progress

The Adreno 610 can be powered without a GMU and can execute a basic command path. Mesa identifies the GPU as `FD610` and OpenGL ES rendering has been observed in the Darwin compositor. This is not yet a finished graphics port: frame pacing and composition remain poor, and the Metal path still needs a proper Adreno 610 integration.

## Sources and provenance

Vendor firmware and Apple system images used for local testing are not part of this repository. Do not commit firmware blobs, Apple files, device serial numbers or host-specific paths. See the common HakiOS documentation for the porting and provenance checks.

- [HakiOS kernel](https://github.com/oosexgang/hakios-xnu-kernel)
- [HakiOS UEFI](https://github.com/oosexgang/hakios-uefi)
- [HakiOS port registry](https://hakios-developers.com/)

## Recovery

The port is experimental. Keep fastboot available and do not touch the vendor boot chain partitions. If a test hangs, return to fastboot with the hardware key combination and restore the active slot as needed.
