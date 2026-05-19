# nokia-c01-plus-pmos-port

Pre-porting reconnaissance for **Nokia C01 Plus** (HMD codename **Iris**, model TA-1383 / TA-1396, board T19545AA1). Targets a **postmarketOS / Alpine Linux** port — primary intent is a low-power web-server / always-on Linux box.

This repo contains only **sanitized hardware description** harvested from a unit with unlocked bootloader and Magisk root. No proprietary firmware, no per-device IDs (IMEI / MAC / vbmeta hash) are included.

## TL;DR

| | |
|---|---|
| SoC | Unisoc / Spreadtrum SC9863A (sharkl3, 8× Cortex-A55) |
| RAM / Storage | 1 GiB / 16 GB eMMC |
| Display | GC9702 MIPI-DSI, 1440×720 |
| Mainline kernel SoC support | **yes** (in `arch/arm64/boot/dts/sprd/` since 2020) |
| Mainline board support | **no** (this board has never been ported) |
| Bootloader | Unisoc, A/B slots, unlockable, AVB orange |
| Reference DTS to clone | `sp9863a-1h10.dts` |
| pmOS port status | **not started** — this repo is step 0 |

## Why this device, why pmOS

- 8 Cortex-A55 cores at 1.6 GHz for ~3 W idle — usable as a passively-cooled web-server / Caddy / nginx box
- 1 GiB RAM cramped under Android 11; freeing the 5 GiB Android footprint lets userspace breathe
- mainline already has the SoC, so port effort is bounded (driver gap is mostly LCD, panel, modem, WiFi blob loading)
- camera/touch/audio not needed for server use case → big chunks of vendor stack can be skipped

## What's in this repo

```
device_info.md                            Sanitized full device write-up. Drop into pmOS wiki.
extracted_dts/
  iris_soc_sc9863a.dts                    Decompiled DTS extracted from stock boot_a (the SoC tree)
  iris_soc_sc9863a.dtb                    Original binary
  iris_dtbo_00_sp9863a_1h10_board.dts     DTBO entry 0 — generic 1h10 reference (unused at runtime)
  iris_dtbo_01_T19545AA1_board_USED.dts   DTBO entry 1 — HMD's actual board overlay (idx=1 per cmdline)
  iris_xenvm_stub.dts                     Small XenVM stub, not interesting
```

## What's deliberately NOT in this repo

- Full eMMC image (`mmcblk0_full.img`, 14.6 GiB) — contains user data and per-device unique IDs
- Per-device factory partitions: `prodnv`, `l_fixnv*`, `persist` — IMEI / MAC / WiFi calibration; redistributing these is identity theft
- Proprietary firmware (`wcnmodem`, `l_modem`, `l_gdsp`, `l_ldsp`, `vendor_a` blobs) — HMD / Unisoc copyright
- Stock `boot_a` / `vendor_boot_a` binaries — HMD copyright, can be re-extracted from any unit

If you have an Iris unit and want to port: dump the same partitions from **your own** device with `dd`, you do not need anyone else's.

## Bring-up checklist for a porter

1. **Confirm unlock** — `fastboot getvar unlocked` must return `yes`. If not, see `patrislav1/unisoc-unlock` and HMD's identifier-token flow. Iris uses signed boot even when unlocked, but unlock disables AVB verification of `boot`.
2. **UART console** — solder/probe to `ttyS1` (`sprd_serial @ 0x70100000`, 115200n8). Earlycon works from before pmbootstrap's image even mounts.
3. **Custom boot image** — `fastboot boot pmos.img` first (RAM-only, does not flash). Once it boots and gives you a shell, flash to **slot B**: `fastboot flash boot_b pmos.img`. Keep slot A as Android fallback.
4. **Disable AVB verification on the new slot** — flash an empty vbmeta with `--disable-verification --disable-verity` flags, or sign with your test keys.
5. **Pull mainline DTS skeleton** — start from `sp9863a-1h10.dts` upstream, layer Iris-specific bits found in `extracted_dts/iris_dtbo_01_*.dts` on top: panel driver (GC9702, not nt35695), eMMC mode, RAM size, USB, sensors that matter for the use case.
6. **Network bring-up** — pmbootstrap stock initramfs gives USB-OTG networking out of the box. For Iris use the standard usb-tethering approach; native WiFi requires `wcnmodem` proprietary blob loaded into a downstream wcn driver (not present in mainline 6.x).

## Suggested pmaports recipe layout (when ready)

```
device-nokia-iris/
  APKBUILD
  deviceinfo
  modules-initfs
  config.deviceinfo
  uboot.fragment
  panel-gc9702.patch
  ...
linux-postmarketos-qcom-sc9863a/       (or your own kernel-package name)
  APKBUILD
  config-postmarketos.aarch64
  patches/
    0001-arm64-dts-sprd-add-iris.patch
    ...
firmware-nokia-iris/                    (your own — fishes proprietary blobs out of stock vendor_a at install time)
  APKBUILD
```

## Project status

| Component | Status |
|---|---|
| Device-info gathering | ✅ Complete |
| DTS extraction | ✅ Complete |
| Mainline kernel patch | ⏳ Not started |
| pmbootstrap recipe | ⏳ Not started |
| UART console working | ⏳ Not tested |
| `fastboot boot` of custom image | ⏳ Not attempted |
| USB-OTG networking | ⏳ Pending |
| eMMC access from mainline | ⏳ Pending |
| Display init | ⏳ Pending (low priority, headless target) |
| WiFi | ⏳ Open (proprietary blob from `wcnmodem`) |
| Modem | ❌ Not in scope (web-server use case) |
| Touch | ❌ Not in scope |

## References

- Linux mainline: [`arch/arm64/boot/dts/sprd/sp9863a-1h10.dts`](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/sprd/sp9863a-1h10.dts)
- postmarketOS Wiki, [Spreadtrum SC9863A category](https://wiki.postmarketos.org/wiki/Category:Spreadtrum)
- [`strongtz/linux-sprd`](https://github.com/strongtz/linux-sprd) — downstream Spreadtrum kernel forks (v4.14)
- [`almondnguyen/twrp_device_samsung_a3core`](https://github.com/almondnguyen/twrp_device_samsung_a3core) — TWRP tree for a peer SC9863A device (m168)
- [`patrislav1/unisoc-unlock`](https://github.com/patrislav1/unisoc-unlock) — identifier-token unlock for Unisoc
- pmaports: [GitLab `postmarketOS/pmaports`](https://gitlab.com/postmarketOS/pmaports)
- Contributing to pmOS: [Porting guide](https://wiki.postmarketos.org/wiki/Porting_to_a_new_device)

## License

The DTS files in `extracted_dts/` are derived from the device's stock firmware. Their copyright belongs to the original authors (Spreadtrum / HMD Global / Microarray for the fingerprint compat). They are reproduced here for hardware-description / interoperability purposes only.

Sanitized device documentation (`device_info.md`, this `README.md`) is released under **CC0 / public domain** by the contributor — use freely.
