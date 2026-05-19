# Nokia C01 Plus (HMD Iris / T19545AA1 Board) — Device Info

Compiled from a unlocked-bootloader, root-Magisk Android 11 unit on 2026-05-19 for the purpose of seeding a postmarketOS / Alpine port. All per-device unique identifiers (serial, IMEI, MAC, vbmeta digest of the user image) are redacted.

## Identity

| Property | Value |
|----------|-------|
| Marketing name | Nokia C01 Plus |
| Model number | TA-1383 (also exists as TA-1396) |
| Codename | Iris |
| Manufacturer | HMD Global |
| SoC | Unisoc / Spreadtrum SC9863A (a.k.a. SP9863A / sharkl3) |
| CPU | 8× ARM Cortex-A55 @ 1.6 GHz, ARMv8 |
| Android userland ABI | `armeabi-v7a` (32-bit userspace on 64-bit CPU) |
| Stock kernel | Linux 4.14.193 (Spreadtrum BSP, Android Common) |
| Stock Android | 11 (Go edition), SDK 30 |
| RAM | 1 GB LPDDR (cmdline `androidboot.ddrsize=1024M`) |
| eMMC | 14.6 GB (15,665,725,440 bytes) usable; manfid 0xd6 |
| Display | GC9702 panel, MIPI-DSI, 1440×720, pixel_clock 76.8 MHz |
| Reference board (closest mainline) | `sp9863a-1h10` (sharkl3.dtsi) |
| Board ID (HMD-specific) | T19545BA1 / T19545AA1 |
| Region SKU sampled | 600RU |

## Boot chain / cmdline (sanitized)

```
earlycon=sprd_serial,0x70100000,115200n8
console=ttyS1,115200n8
loglevel=1
init=/init
root=/dev/ram0  rw
printk.devkmsg=on
androidboot.boot_devices=soc/soc:ap-ahb/20600000.sdio
androidboot.hardware=T19545AA1
androidboot.dtbo_idx=1
lcd_id=IDffff
lcd_name=lcd_GC9702_lide_mipi_hd
lcd_base=9e000000
lcd_size=1440x720
pixel_clock=76800000
logo_bpix=24
androidboot.ddrsize=1024M
androidboot.wdten=e551
modem=shutdown
ltemode=lcsfb
rfboard.id=1
rfhw.id=0
crystal=2
32k.less=1
modemboot.method=emmcboot
androidboot.board_id=T19545BA1
androidboot.product.hardware.sku=dualsim
androidboot.verifiedbootstate=orange
androidboot.skuid=600RU
androidboot.factory_mode=0
androidboot.tacode=TA-1383
androidboot.secureboot=1
androidboot.flash.locked=0
androidboot.slot_suffix=_a
androidboot.force_normal_boot=1
```

- **earlycon** at physical `0x70100000`, 115200n8 → key for getting kernel messages over UART without display
- **eMMC controller** at `0x20600000` (sdio host) on the `ap-ahb` bus
- **`flash.locked=0`** confirms the bootloader is **really** unlocked (matches `fastboot getvar unlocked: yes`)
- **`verifiedbootstate=orange`** = unlocked AVB state
- **A/B slots**, no recovery partition (recovery lives in `vendor_boot`)

## Bootloader / fastboot capabilities

```
fastboot getvar unlocked       -> yes
fastboot getvar product        -> Iris
fastboot getvar version        -> 1.0
fastboot getvar slot-count     -> 2
fastboot getvar current-slot   -> a
fastboot getvar serialno       -> (redacted)
```

`fastboot getvar all` returns empty (this Unisoc fastboot implements only flash / unlock — no inspection). Individual `getvar <key>` works for documented vars. Custom OEM commands not enumerated.

## Partition layout (eMMC, GPT)

Output of `ls -la /dev/block/by-name/` plus `/proc/partitions` (block = 1 KiB):

```
mmcblk0                 = 15,298,560 KiB (~14.6 GiB)
mmcblk0boot0            =       4096 KiB  (eMMC HW boot 0, contains primary bootloader)
mmcblk0boot1            =       4096 KiB  (eMMC HW boot 1, identical mirror of boot0 in this device)
```

Per-partition (slot A and slot B both exist for most; sizes are KiB):

| Partition | Slot A blk | Slot B blk | Size KiB | Notes |
|-----------|-----------|-----------|---------:|-------|
| `prodnv`         | mmcblk0p1   | (single)   |  10240 | Unique per-device, holds IMEI/MAC, **do not redistribute** |
| `elabel`         | mmcblk0p2   | (single)   |   5120 | Regulatory labels |
| `miscdata`       | mmcblk0p3   | (single)   |   1024 |  |
| `misc`           | mmcblk0p4   | (single)   |   1024 |  |
| `trustos`        | mmcblk0p5   | mmcblk0p6  |   6144 | TEE OS |
| `sml`            | mmcblk0p7   | mmcblk0p8  |   1024 | Secure monitor |
| `uboot`          | mmcblk0p9   | mmcblk0p10 |   1024 | Unisoc U-Boot |
| `uboot_log`      | mmcblk0p11  | (single)   |   4096 |  |
| `logo`           | mmcblk0p12  | (single)   |   6144 | Boot logo |
| `fbootlogo`      | mmcblk0p13  | (single)   |   6144 |  |
| `l_fixnv1`       | mmcblk0p14  | mmcblk0p16 |   2048 | Modem fixed NV (**per device**) |
| `l_fixnv2`       | mmcblk0p15  | mmcblk0p17 |   2048 |  |
| `l_runtimenv1`   | mmcblk0p18  | (single)   |   2048 |  |
| `l_runtimenv2`   | mmcblk0p19  | (single)   |   2048 |  |
| `gpsgl`          | mmcblk0p20  | mmcblk0p21 |   1024 | GPS GLONASS |
| `gpsbd`          | mmcblk0p22  | mmcblk0p23 |   1024 | GPS BeiDou |
| `wcnmodem`       | mmcblk0p24  | mmcblk0p25 |  10240 | WiFi/BT firmware blob |
| `persist`        | mmcblk0p26  | (single)   |   2048 |  |
| `l_modem`        | mmcblk0p27  | mmcblk0p28 |  25600 | LTE modem firmware |
| `l_deltanv`      | mmcblk0p29  | mmcblk0p30 |   1024 |  |
| `l_gdsp`         | mmcblk0p31  | mmcblk0p32 |  10240 | DSP firmware |
| `l_ldsp`         | mmcblk0p33  | mmcblk0p34 |  20480 | DSP firmware |
| `pm_sys`         | mmcblk0p35  | mmcblk0p36 |   1024 | Power-mgmt subsystem |
| `teecfg`         | mmcblk0p37  | mmcblk0p38 |   1024 |  |
| `hypervsior`     | mmcblk0p39  | mmcblk0p40 |   2048 | (sic) — hypervisor blob |
| `boot`           | mmcblk0p41  | mmcblk0p42 |  65536 | Android boot.img v3, contains kernel + DTB appended |
| `vendor_boot`    | mmcblk0p43  | mmcblk0p44 | 102400 | Android vendor_boot.img v3 (vendor ramdisk) |
| `dtb`            | mmcblk0p45  | mmcblk0p46 |   8192 | **empty in this device** (DTB lives in `boot`) |
| `dtbo`           | mmcblk0p47  | mmcblk0p48 |   8192 | Android DTBO image v0, 2 overlays |
| `super`          | mmcblk0p49  | (single)   | 2693120 | Dynamic partitions (LP v10.2, Virtual A/B) |
| `cache`          | mmcblk0p50  | (single)   |  20480 |  |
| `socko`          | mmcblk0p51  | mmcblk0p52 |  76800 |  |
| `odmko`          | mmcblk0p53  | mmcblk0p54 |  25600 |  |
| `vbmeta`         | mmcblk0p55  | mmcblk0p56 |   1024 | AVB chained, currently signed by HMD but flag indicates user is in `unlocked` state |
| `metadata`       | mmcblk0p57  | (single)   |  16384 |  |
| `vbmeta_system`        | p58 | (slot A only above) |   1024 | (also `_b` exists symmetrically) |
| `vbmeta_vendor`        | p60 |   |   1024 |  |
| `vbmeta_hmdodm`        | p62 |   |   1024 | HMD ODM verity meta |
| `vbmeta_system_ext`    | p64 |   |   1024 |  |
| `vbmeta_product`       | p66 |   |   1024 |  |
| `vbmeta_odm`           | p68 |   |   1024 |  |
| `sysdumpdb`            | p70 | (single) |  …   |  |
| `avbmeta_rs`           | p71 |   |   1024 |  |
| `common_rs1`           | p73 |   |   2048 |  |
| `userdata`             | p75 | (single) | rest |  |

(Slot B duplicates exist for everything except clearly-single partitions: `prodnv`, `elabel`, `miscdata`, `misc`, `uboot_log`, `logo`, `fbootlogo`, `l_runtimenv1/2`, `persist`, `cache`, `metadata`, `sysdumpdb`, `userdata`.)

## `super` (dynamic partitions) layout

`lpdump` output, slot 0:

```
Metadata version: 10.2
Header flags: virtual_ab_device
group_unisoc_a:
  system_a      ~640 MiB  (fragmented across super)
  system_ext_a  ~170 MiB
  vendor_a       291 MiB   ← contains proprietary firmware blobs
  product_a    ~1024 MiB
  hmdodm_a       410 MiB   ← HMD-specific overlay
cow:
  system_a-cow   114 MiB
```

## Device tree

Stock kernel uses an **appended DTB inside `boot_a`** (the `dtb_a` partition is empty / zero-filled). The boot image carries:

* `dtb_00` @ offset `0x179c860` in `boot_a`, 85 408 bytes  
  `model = "Spreadtrum SC9863A SoC"`, `compatible = "sprd,sc9863a"` — the SoC-level tree
* `dtb_01` @ offset `0x17b1600`, 1 288 bytes — a small Xen-VM stub (not used at runtime on this device)

DTBO (`dtbo_a`, Android DTBO v0, 2 entries):

* Entry 0: `Spreadtrum SP9863A-1H10-GO-32b Board` — generic Unisoc reference 1H10 overlay (with `nt35695_truly_mipi_fhd` panel description — **not the panel actually used**)
* Entry 1: `Spreadtrum T19545AA1 Board` (compatible `sprd,T19545AA1`, `sprd,sc9863a`) — **this is the actually-applied overlay** (`androidboot.dtbo_idx=1`)

Entry 1 is the canonical Iris board overlay. It contains a `microarray,afs121` fingerprint sensor compatible string and HMD-specific tweaks. Decompiled DTS sources are in `extracted_dts/` alongside this file.

## What the actual stock cmdline tells us about hardware bring-up

* Console output for porting: connect a UART at `ttyS1` (mapped to `sprd_serial` at physical `0x70100000`, 115200n8). Earlycon will work from before `console=` is parsed.
* eMMC host needed by mainline DTS: a node compatible with `sprd,sdhci-r11` at `0x20600000` on `ap-ahb`.
* Modem boots from eMMC (`modemboot.method=emmcboot`) — no separate baseband flash chip.
* The display panel actually installed is **GC9702** (`lcd_GC9702_lide_mipi_hd`, 1440×720), MIPI-DSI 4-lane, pixel clock 76.8 MHz. The reference 1H10 DTBO mentions `nt35695` but Iris uses GC9702 — porters will need a GC9702 panel driver.
* `androidboot.skuid=600RU` is the Russian dual-SIM variant; other regions exist (TA-1383 vs TA-1396).

## What I dumped locally (for offline analysis only — do not redistribute)

The following images were taken via `dd if=/dev/block/by-name/<part>` while running on root (Magisk). All carry per-device fingerprints in places and should stay on the original owner's machine.

* `mmcblk0_full.img` — bit-perfect 14.6 GiB image of the whole eMMC (15 665 725 440 bytes, SHA-256 `800b40657f7b1fbd2eeb71e99f41e00542e6c951f29812af2fd84663bc2a6ed7`)
* `mmcblk0boot0.img` / `mmcblk0boot1.img` — eMMC HW boot partitions (4 MiB each, identical, SHA-256 `01ba662ff08f527490e91e062b08f990731b49e026c36b3cbf7450db4c8aacf9`)
* 46 individual partition images covering both A and B slots of everything critical

These are the **only artifacts that should travel publicly:**

* `extracted_dts/` (in this directory): all 4 DTS sources extracted from `boot_a` and `dtbo_a` — these are hardware description only, no secrets
* `device_info.md` (this file)

## Status of pmOS / Alpine port

* No upstream pmOS port exists for Nokia C01 Plus / Iris (verified against the pmaports tree as of 2026-05).
* Closest references usable as a starting tree:
  * Mainline `arch/arm64/boot/dts/sprd/sp9863a-1h10.dts` (Unisoc reference board, in Linux since 2020)
  * Samsung Galaxy A03 Core (`m168` / sp9863a) — a community-maintained TWRP/Android kernel tree exists on GitHub and shares the SC9863A SoC with a similar 1 GB RAM, 32-bit userland configuration
* TWRP: **none** for this device. Generic SC9863A TWRP 3.5.0 ports exist (unofficialtwrp.com) but authors warn touch/mount may not work. With Magisk root in stock Android, TWRP is not actually required for the porting workflow — `dd` + `fastboot flash`/`fastboot boot` cover the same ground more reliably.

## Useful refs

* `arch/arm64/boot/dts/sprd/` in Linus's tree (sc9863a.dtsi, sharkl3.dtsi, sp9863a-1h10.dts)
* postmarketOS Wiki, `Spreadtrum_SC9863A` page (community state of the chipset)
* `github.com/strongtz/linux-sprd` — downstream Spreadtrum kernel forks (v4.14)
* `github.com/almondnguyen/twrp_device_samsung_a3core` — TWRP tree for a peer SC9863A device, useful for cribbing the device tree skeleton
* `github.com/patrislav1/unisoc-unlock` — Python reimplementation of the Unisoc identifier-token unlock flow (relevant if anyone else with a locked Iris needs to unlock first)
