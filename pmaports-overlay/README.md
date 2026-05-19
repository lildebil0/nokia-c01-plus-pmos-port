# pmaports overlay for Nokia C01 Plus

Files to drop into a fresh pmbootstrap `pmaports` clone for porting Nokia C01 Plus (Unisoc SC9863A).

## Layout

```
device/testing/device-nokia-iris/       — device package skeleton (deviceinfo + APKBUILD + modules-initfs)
device/testing/linux-nokia-iris/        — kernel package (mainline 7.1-rc4 + custom SPRD config)
```

## How to apply

```bash
# After `pmbootstrap init` has cloned pmaports and you've picked vendor=nokia codename=iris:
PMA=~/.local/var/pmbootstrap/cache_git/pmaports
cp -r pmaports-overlay/device/testing/device-nokia-iris "$PMA/device/testing/"
cp -r pmaports-overlay/device/testing/linux-nokia-iris "$PMA/device/testing/"

# Regenerate checksums (tarball SHA was generated for our network, re-verify):
pmbootstrap checksum linux-nokia-iris
pmbootstrap checksum device-nokia-iris

# Build kernel
pmbootstrap build linux-nokia-iris
```

## Stage A goal (current)

Boot kernel via UART (`sprd_serial @ 0x70100000, 115200n8`), get to SSH/USB-net.
No display panel, no touch, no modem, no WiFi.

## Known requirements (verify on new machine)

- pmbootstrap **3.9.0+** (apt version 3.1.0 too old — install from git, see `HANDOFF.md`)
- WSL2 IPv4 preference via `/etc/gai.conf` (Python urllib otherwise picks broken IPv6)
- Alpine mirror change to `mirror.yandex.ru` if local ISP blocks Fastly
- `NOPASSWD` sudo (pmbootstrap calls sudo every ~10s)
