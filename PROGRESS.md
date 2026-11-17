# sapphiren port progress

Updated: 2026-11-17. **Wi-Fi is not functional yet.** This is a handoff for continuing the experimental port, not a claim of device support. The phone is a Redmi Note 13 4G NFC (`sapphiren`, Khaje/SM6225) with an older, partly functional iOS port, **not Android**.

## Verified today

- Darwin 22.5.0 / iOS 16.5 reaches SpringBoard; the existing Adreno 610 path draws but graphics performance and Metal integration need major work.
- A 50 MB diagnostic APFS image was built and independently checked. It contains the phone's MPSS firmware, `modemuw.jsn` and `wlanmdsp.mbn`. It is **local test material, not committed here**.
- With `modem_no_ipa_qmi=1`, MPSS runs without the QMI-IPA/GSI assertion and stops cleanly. Reading `modemuw.jsn` adds the `msm/modem/wlan_pd` services to the service registry, including `kernel/elf_loader` and `wlan/fw`.
- Serving `wlanmdsp.mbn` (3,815,872 bytes) from the resident diagnostic image through TFTP causes the modem to announce **WLFW service `0x45`, version 1, instance 0**, among 42 QRTR services (`151106-run.log`). This is not itself working Wi-Fi.
- The opt-in Khaje WLFW probe received successful replies to `IND_REGISTER` (`0x20`), `HOST_CAP` (`0x34`) and `MSA_INFO` (`0x2d`). The firmware returned two regions within the device tree's 1 MiB WLAN MSA reservation: `0x51900000+0x10000` (secure) and `0x51910000+0xf0000` (non-secure). These were measured without changing ownership (`152416-run.log`).
- A separate opt-in `modem_wlfw_assign=1` attempted the first TrustZone MSA assignment. SCM returned **`-2`, answer `0x56`**; the code correctly **did not send `MSA_READY`**. MPSS remained up for the observation period and shut down cleanly (`153017-run.log`). The same SCM code already occurred when trying to reclaim the RMTFS buffer in earlier runs, so do not attribute the WLAN failure to an unverified single cause.

## Blockers and next work

1. **MSA ownership:** compare the SCM MP_ASSIGN ABI, source-owner VMID set and destination permissions with the phone's actual boot state and the Khaje/ath10k implementation. Do not repeat the rejected assignment unchanged or assume that `0x56` proves one particular cause. The first assignment failed before any `MSA_READY` request.
2. **WLFW and data path:** after a justified, successful MSA handoff, implement/verify `MSA_READY`, capability and board-ID handling, transfer the phone's own `bdwlan.*` data, wait for firmware readiness, and configure the WCN3990 copy engines and network stack. The existing WCN6750/WPSS `wifi=1` path and its WLFW client are **not** this architecture. No scan, association, IP connectivity or Darwin Wi-Fi UI has been demonstrated.
3. **IPA 4.2:** the device tree reports IPA 4.2 (`qcom,ipa-hw-ver = <0x10>`), whereas the common IPA/GSI backend still models IPA 4.11. Sending QMI-IPA with the four hash TLVs removed gets past the earlier assertion but fails at GSI channel 1 with `errval 6` (invalid parameters). `modem_no_ipa_qmi=1` is a diagnostic workaround, not a complete mobile-data implementation; the IPA 4.2 port remains separate work.
4. **Graphics after Wi-Fi:** adapt only selected Adreno 610/Metal changes from the separate experimental fork to the current `HakiosGPU` service. Do not merge that fork wholesale: it expects `SapphireGPU`, and some performance changes froze another device.

## Local work and evidence

- The Khaje MPSS, TFTP and gated WLFW changes are in the **uncommitted common HakiOS tree**, mainly `drivers/modem/hakios.mpss.c`. The optional WLAN MSA assignment currently fails; do not present the code as validated Wi-Fi support. Other local changes in that tree predate this note; do not commit them wholesale.
- Diagnostic logs retained locally: `150029-run.log` (40 services, no WLFW), `151106-run.log` (WLFW appears after WLAN firmware transfer), `152038-run.log` (registration and host capability accepted), `152416-run.log` (MSA layout read-only), `153017-run.log` (SCM rejection). The diagnostic APFS image is also local and is not tracked by this repository.
- This repository has **separate uncommitted changes** in `soc.args` and `configs/sapphiren.config`. They are intentionally excluded from the documentation commit.

## Safe test procedure

- Before **each** kernel test, physically return from `xnuloader` to fastboot with **Power + Volume Down**; do not run a second kernel in the same loader session. Verify `product: sapphiren` and `current-slot: b` before proceeding. Device state must be checked afresh rather than inferred from this note.
- Use only the known-working original loader via `fastboot boot`, a small validated APFS diagnostic image in RAM, and `XL_FOLLOW=` to preserve the final log. **Do not flash partitions**, replace `root.apfs`/`flashboot.apfs`/`system.apfs`, or use the rejected auxiliary logdump loader. Keep the test kernel from accidentally sending the IPA 4.11 QMI handshake.
- Vendor firmware, Apple images, personal device identifiers and host-specific absolute paths must stay out of this public repository.

The last observed test ended in `xnuloader`. The person has since indicated the phone was returned to fastboot; a subsequent check reported `sapphiren`, slot `b`. **Check again before any future boot.**
