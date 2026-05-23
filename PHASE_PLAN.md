# Nokia C01 Plus pmOS port — Phase plan

Documents which approach we use, why mainline 7.1-rc4 (build #8) did not
boot, and what the next iterations are.

## Diagnosis (build #8 mainline 7.1-rc4)

Build #8 on desktop 13900K compiled cleanly (~71 MB APK at
`~/.local/var/pmbootstrap/packages/edge/aarch64/linux-nokia-iris-7.1_rc4-r0.apk`)
but the device did not boot when flashed.

Likely root cause: pmOS mainline + sed-enabled SPRD drivers covers only
a subset of SC9863A:

- `drivers/clk/sprd/sc9863a-clk.c` — present in mainline
- `drivers/pinctrl/sprd/pinctrl-sprd-sc9860.c` — present but SC9863A pin
  map incomplete; SC9863A registers many pins not in upstream tables
- `drivers/regulator/sprd-pmic-*` — SC2731 PMIC driver upstream, partial
- `drivers/mmc/host/sdhci-sprd.c` — present, missing platform glue
- `drivers/usb/dwc3/dwc3-sprd.c` — present
- `drivers/tty/serial/sprd_serial.c` — present, OK

`CONFIG_*=y` switches the build to include the driver but does not
synthesize missing platform glue; result on first boot is silent UART
(clock framework probably comes up, but pinctrl never registers the
serial pin function, so console output never reaches the pad).

## Approach matrix (summary)

| Path | Time to Stage A | Long-term | Verdict |
|------|-----------------|-----------|---------|
| A. Mainline + sed-enables (current) | weeks (driver work) | ✅ future-proof | ❌ slow for headless |
| B. Downstream strongtz 4.14 | days | ❌ EOL kernel | ✅ for Stage A |
| C. Mainline base + backport drivers | months | ✅ | overkill for hobby |
| F. **B for Stage A, optional C later** | days | flexible | ✅ chosen |

## Approach F — chosen path

### Phase 0 (now, no internet, no phone)

- [x] Audit extracted DTS (`extracted_dts/iris_dtbo_*`); confirm T19545AA1
      overlay differs from sp9863a-1h10 reference ONLY in LCD panel
      definitions (HD vs FHD); UART/MMC/clocks unchanged.
- [x] Write `APKBUILD.v2-downstream` (strongtz `d834a127a`,
      sprd_sharkl3_defconfig base, clang LLVM=1).
- [x] Write `config-nokia-iris.aarch64.v2-fragment` (overlay on top of
      sharkl3 defconfig: USB-net for SSH, OpenRC/cgroup compat,
      disable Android bloat).
- [x] Document phase plan (this file).
- [x] Commit + push via Windows-side gh.exe (WSL has no outbound while
      AmneziaVPN is on; gh on Windows works).

### Phase 1 — Stage A boot (downstream)

Requires: WSL outbound (AmneziaVPN window OFF or split-tunnel) + phone
connected.

1. Rename `APKBUILD.v2-downstream` -> `APKBUILD` (replacing mainline one);
   rename `config-nokia-iris.aarch64.v2-fragment` ->
   `config-nokia-iris.aarch64`.
2. Re-apply overlay into pmaports cache (`cp -r`, see HANDOFF.md step 5).
3. `pmbootstrap zap -p` (clean previous mainline state).
4. `pmbootstrap checksum linux-nokia-iris` (downloads strongtz tarball,
   computes hash; copy back into APKBUILD `sha512sums=""`).
5. `pmbootstrap build linux-nokia-iris`.  Expected first failures:
   - Clang 22 vs Linux 4.14 — most warnings should be fine, occasional
     `-Werror` may surface.  Uncomment `KCFLAGS=-w` line in APKBUILD.
   - Old DTC syntax — usually clang/LLVM build accepts both.
6. `pmbootstrap build device-nokia-iris`.
7. `pmbootstrap install --no-fde --split`.
8. `pmbootstrap export` -> nokia-iris-boot.img + rootfs.img.
9. `fastboot boot nokia-iris-boot.img` (one-shot, no write).
10. UART capture on phone test pads (115200 8N1, USB-TTL adapter, 1.8V
    level — likely need MAX3221 or divider if adapter is 3.3V-only).

### Phase 2 — Userspace bring-up

Once UART shows kernel reaches userspace:

1. `g_ncm` / USB configfs setup script in initramfs so phone exposes
   ethernet-over-USB.
2. OpenRC service for dropbear/sshd.
3. SSH from host via the USB-net interface (`ssh root@10.0.0.2` or
   whatever DHCP assigns).
4. Document working state, persist with
   `fastboot flash boot_a nokia-iris-boot.img`.

### Phase 3 — Optional

Either: stop (Stage A delivered headless server) or start Approach C
(backport SPRD drivers strongtz -> mainline LTS).

## Init system decision: OpenRC

- Linux 4.14 lacks cgroupv2 unified hierarchy required by modern
  systemd (>= 254).
- systemd >= 257 (in pmOS edge) refuses to start when only cgroupv1 is
  available.
- OpenRC works fine on 4.14, uses cgroupv1, is the historical pmOS
  default for headless devices.
- Reconfigure: `pmbootstrap config init openrc` before Phase 1.

When pmbootstrap re-runs install, it will pull `openrc` + base
service set (dropbear, networkmanager, etc.) instead of systemd units.

## What we are NOT doing in Phase 1

- Custom DTS port for T19545AA1 (defer to Stage B — only matters for
  display panels, which are absent on a headless server).
- Stage B/C work (display, touch, modem, WiFi, BT).
- Mainline backport (defer to Phase 3 if anyone needs it).

## File layout after Phase 0

```
pmaports-overlay/device/testing/linux-nokia-iris/
├── APKBUILD                                   (mainline 7.1-rc4, current; will be replaced)
├── APKBUILD.v2-downstream                     (new, ready to swap in)
├── config-nokia-iris.aarch64                  (mainline-generated, current; will be replaced)
└── config-nokia-iris.aarch64.v2-fragment      (new fragment for sharkl3 defconfig)

pmaports-overlay/device/testing/device-nokia-iris/
├── APKBUILD                                   (unchanged)
├── deviceinfo                                 (unchanged — dtb still sprd/sp9863a-1h10)
└── modules-initfs                             (unchanged)
```

Mainline files kept for now (commented git history); will delete after
downstream path proves Stage A boot.
