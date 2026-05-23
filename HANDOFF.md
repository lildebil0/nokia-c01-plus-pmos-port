# Nokia C01 Plus → pmOS port — build handoff

> **2026-05-23 status update**: build #8 mainline 7.1-rc4 compiled cleanly
> but did not boot on the device (SPRD drivers in mainline are partial:
> clock-driver present, pinctrl/regulator/MMC platform glue incomplete
> for SC9863A). Direction changed: switch to **strongtz/linux-sprd 4.14
> downstream** for Stage A headless boot.  See `PHASE_PLAN.md` for the
> full approach analysis and Phase 0/1/2 deliverables.  The instructions
> in this file below still apply for the WSL setup; only the kernel
> APKBUILD + config change.

Snapshot from kukuruza-laptop (Ryzen 5 6600H) on 2026-05-19 17:15. Moving build to a faster machine (13900K-class). This doc is everything you need to reproduce the WSL+pmbootstrap setup and resume build attempts.

## Status when handoff was made

- WSL2 + Debian 13 + systemd: working
- pmbootstrap 3.10.1 from git: installed
- pmaports cloned, device-nokia-iris skeleton generated, linux-nokia-iris kernel package written (mainline 7.1-rc4 + SPRD drivers enabled in config)
- 5 build attempts on laptop: 3 failed on chroot/network gotchas, 2 failed on packaging-stage make-target issues

## Update 2026-05-19 (desktop, B660 + ES 12900K)

- **Stage A kernel build PASSED** — build #8 on desktop. APK `linux-nokia-iris-7.1_rc4-r0.apk` (~71 MB) at `~/.local/var/pmbootstrap/packages/edge/aarch64/`.
- Two new gotchas hit on desktop, both now in deps below: missing `kpartx` (pmbootstrap 3.10+ init refuses to run), missing `zstd` in build-chroot makedepends (kernel 7.1 default `CONFIG_MODULE_COMPRESS_ZSTD=y` makes `modules_install` invoke `zstd`).
- Overlay-is-copy gotcha: `pmaports-overlay/` is `cp -r`'d into pmaports cache, not symlinked. Edits to APKBUILD in repo do NOT reach build until re-applied. Either edit both copies or re-run the `cp -r` step before each build.
- Channel `edge` got rewritten to `systemd-edge` by pmbootstrap because UI=none/console now defaults to systemd init.

## Steps on the new PC

### 1. Install WSL2 + Debian

If WSL not installed at all:

```powershell
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
# reboot
wsl --set-default-version 2
wsl --install -d Debian
```

If WSL kernel installed but no distro: `wsl --install -d Debian` is enough (no reboot).

First launch: create user (we used `kukuruza` / pwd `kukuruza`) — name doesn't matter, just remember it for sudo prompts.

### 2. Critical Debian-inside-WSL setup

**a. Alpine/Fastly mirror swap (if ISP blocks Fastly)**

Test first: `wsl -d Debian -- bash -lc 'timeout 5 bash -c "(echo > /dev/tcp/deb.debian.org/443)" && echo OK || echo BLOCKED'`

If BLOCKED — same ISP block as KG home. Patch sources:

```bash
sudo sed -i 's|https://deb.debian.org/debian|https://mirror.yandex.ru/debian|' \
  /etc/apt/sources.list.d/0000debian.sources
sudo apt-get update
```

If OK — skip.

**b. Install deps**

```bash
sudo apt-get install -y git curl pipx expect ca-certificates build-essential kpartx zstd
```

`kpartx` is required by pmbootstrap 3.10+ — without it, `pmbootstrap init` exits with "Can't find all programs required to run pmbootstrap. Please install the following: kpartx". `zstd` is needed on the host for tarball extraction in some flows; the makedepends in the kernel APKBUILD covers the in-chroot build path.

**c. Force IPv4 in /etc/gai.conf (Python urllib will pick broken IPv6 otherwise)**

```bash
sudo bash -c 'cat > /etc/gai.conf <<EOF
label  ::1/128       0
label  ::/0          1
label  2002::/16     2
label  ::/96         3
label  ::ffff:0:0/96 4
precedence  ::1/128       50
precedence  ::ffff:0:0/96 100
precedence  ::/0          40
precedence  2002::/16     30
precedence  ::/96         20
EOF'
```

Verify:
```bash
python3 -c "import urllib.request; print(urllib.request.urlopen('http://mirror.postmarketos.org/postmarketos/main/x86_64/APKINDEX.tar.gz', timeout=8).status)"
# Should print: 200
```

**d. NOPASSWD sudo (WSL-only, pmbootstrap calls sudo every ~10s)**

```bash
echo 'YOUR_USERNAME ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/99-nopass
sudo chmod 0440 /etc/sudoers.d/99-nopass
sudo -n true && echo NOPASSWD_OK
```

**e. systemd in WSL** (pmbootstrap needs it for proper chroot mounting)

```bash
sudo bash -c 'cat > /etc/wsl.conf <<EOF
[boot]
systemd=true

[user]
default=YOUR_USERNAME
EOF'
```

Back on Windows side:
```powershell
wsl --shutdown
wsl -d Debian
# inside: `systemctl is-system-running` should print `running` or `degraded` (not `offline`)
```

### 3. Install pmbootstrap from git (apt 3.1.0 too old — pmaports needs 3.9.0+)

```bash
mkdir -p ~/src && cd ~/src
git clone --depth=1 https://gitlab.postmarketos.org/postmarketOS/pmbootstrap.git
sudo ln -sf ~/src/pmbootstrap/pmbootstrap.py /usr/local/bin/pmbootstrap
pmbootstrap --version   # 3.10.1+
```

### 4. pmbootstrap init

```bash
pmbootstrap init
```

Answers (Stage A headless):

| Prompt | Answer |
|---|---|
| Work path | (default, Enter) |
| Channel | `edge` |
| Vendor | `nokia` |
| Device codename | `iris` |
| Create device package? | `y` |
| Manufacturer | `Nokia` |
| Marketing name | `Nokia C01 Plus` |
| Year | `2021` |
| Chassis | `handset` |
| Has screen? | `y` |
| Has keyboard? | `n` |
| Has external storage? | `y` |
| Arch | `aarch64` |
| Kernel choice | `mainline` |
| Init system | (default, Enter) |
| Username | `user` |
| UI | `none` |
| Extra packages | (Enter) |
| Hostname | `nokia-iris` |
| Encrypt? | `n` |
| Save config? | `y` |
| Zap chroots? | `y` |

After: `pmbootstrap status` should show `Device: nokia-iris (aarch64)`.

### 5. Apply pmaports overlay (THIS REPO)

```bash
cd ~/src
git clone https://github.com/lildebil0/nokia-c01-plus-pmos-port.git
cd nokia-c01-plus-pmos-port

PMA=~/.local/var/pmbootstrap/cache_git/pmaports
cp -r pmaports-overlay/device/testing/device-nokia-iris "$PMA/device/testing/"
cp -r pmaports-overlay/device/testing/linux-nokia-iris "$PMA/device/testing/"

# Yandex Alpine mirror in pmbootstrap (if Fastly blocked):
cat > ~/.config/pmbootstrap_v3.cfg <<EOF
[pmbootstrap]
device = nokia-iris
is_default_channel = False

[providers]

[mirrors]
alpine = http://mirror.yandex.ru/mirrors/alpine/
EOF
```

### 6. Checksum + build

```bash
pmbootstrap checksum linux-nokia-iris    # downloads kernel.org tarball ~250 MB
pmbootstrap checksum device-nokia-iris
pmbootstrap build linux-nokia-iris       # ~5-10 min on 13900K (was 25 min on 6600H)
```

### 7. If build fails

Look at error: `pmbootstrap log` (full log at `~/.local/var/pmbootstrap/log.txt`).

Most likely remaining issues (we hit these on 6600H + 12900K):

- **`No rule to make target 'modules.order'`** — solved: APKBUILD `build()` line includes `Image.gz modules dtbs`. If still happening, double-check `build()` target list.
- **`Missing file: arch/arm64/boot/vmlinuz.efi` in zinstall** — solved: APKBUILD `package()` no longer uses `zinstall` target, manually copies `Image.gz` to `$pkgdir/boot/vmlinuz-$_flavor`. If still happens, check `package()` body.
- **`/bin/sh: zstd: not found` during `make modules_install`** — solved: APKBUILD `makedepends` includes `zstd`. Kernel 7.1 default config sets `CONFIG_MODULE_COMPRESS_ZSTD=y`, so the build-chroot needs the binary even though the host has it. If hit again, verify `zstd` appears in `makedepends` of both repo and pmaports-cache APKBUILD copies.
- **Kconfig olddefconfig prompts** — if interactive, prepend `yes "" |` or kill chroot and retry.
- **Edits to APKBUILD in repo have no effect** — overlay is `cp -r`'d, not symlinked. Re-run the `cp -r` from step 5 after any edit, or edit `~/.local/var/pmbootstrap/cache_git/pmaports/device/testing/linux-nokia-iris/APKBUILD` directly. Keep both copies in sync before committing.

### 8. After kernel apk built

```bash
ls ~/.local/var/pmbootstrap/packages/edge/aarch64/linux-nokia-iris-*.apk
tar tzf ~/.local/var/pmbootstrap/packages/edge/aarch64/linux-nokia-iris-*.apk | grep -E 'vmlinuz|sp9863a'
# Expected: boot/vmlinuz-nokia-iris + boot/dtbs/sprd/sp9863a-1h10.dtb
```

Then full rootfs install:

```bash
pmbootstrap build device-nokia-iris
pmbootstrap install --no-fde --split
# --split keeps boot.img and rootfs.img separate (needed when flashing
# boot to one slot without touching userdata/system on the phone).
pmbootstrap export
# → produces nokia-iris-boot.img + nokia-iris-rootfs.img in
#   ~/.local/var/pmbootstrap/chroot_native/home/pmos/rootfs/
# Confirm with: ls ~/.local/var/pmbootstrap/chroot_native/home/pmos/rootfs/
```

Then flash test on the actual phone:

```bash
# Power off phone, hold Vol- + plug USB → fastboot mode
# Slot _a is the active slot for this device (verified via getvar current-slot).
# Prefer `fastboot boot` first — it boots once without writing to flash:
fastboot boot ~/.local/var/pmbootstrap/chroot_native/home/pmos/rootfs/nokia-iris-boot.img

# Once UART confirms the mainline kernel comes up, persist with:
fastboot flash boot_a ~/.local/var/pmbootstrap/chroot_native/home/pmos/rootfs/nokia-iris-boot.img
```

Watch UART (USB-serial dongle on phone test points) at 115200n8.

## Files in this repo overlay

```
pmaports-overlay/device/testing/device-nokia-iris/
├── APKBUILD                    (depends linux-nokia-iris)
├── deviceinfo                  (kernel_cmdline, flash offsets from stock boot.img v2)
└── modules-initfs              (empty for now)

pmaports-overlay/device/testing/linux-nokia-iris/
├── APKBUILD                    (mainline 7.1-rc4 from kernel.org, LLVM=1 build)
└── config-nokia-iris.aarch64   (pmOS mainline aarch64 config + SPRD drivers enabled)
```

## Key device facts (from stock boot.img v2 + getprop)

- SoC: Unisoc SC9863A (sharkl3)
- Board: T19545AA1, codename `iris`
- Stock kernel: 4.14.193 downstream; we're switching to **mainline 7.1-rc4**
- Boot.img v2 header (mkbootimg fields):
  - page_size=2048
  - kernel_addr=0x00008000, ramdisk_addr=0x05400000, tags_addr=0x00000100
  - dtb_addr=0x01f00000
  - cmdline (stock): `console=ttyS1,115200n8 buildvariant=user`
- Stock cmdline (extended at runtime): `earlycon=sprd_serial,0x70100000,115200n8 ...`
- Flash method: fastboot (Unisoc U-Boot supports basic flash/boot/unlock)
- DTB target: `arch/arm64/boot/dts/sprd/sp9863a-1h10.dts` (upstream mainline since 2020)
- Bootloader: **unlocked** (verified via `fastboot getvar unlocked: yes` + `androidboot.flash.locked=0`)

## What to do AFTER mainline kernel boots on phone

Stage B / C work, after Stage A boot confirmed via UART:

1. Custom DTS for T19545AA1 board (currently using upstream sp9863a-1h10.dts which targets Unisoc reference board, not Iris)
2. GC9702 panel driver (no mainline driver; needed for display)
3. Touch driver (focaltech or similar)
4. Modem (Unisoc out-of-tree; far horizon)
5. WiFi/BT

## Reference: kernel package APKBUILD (current snapshot, also in overlay)

See `pmaports-overlay/device/testing/linux-nokia-iris/APKBUILD`.
