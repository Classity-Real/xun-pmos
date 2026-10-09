# Xiaomi Pad SE (xun) postmarketOS Port — Handoff

## Device
- Xiaomi Pad SE, codename `xun`, SoC Qualcomm SM6225 (Khaje/bengal)
- 8 GB RAM, UFS, 1200x1920 panel (m84_42_03_0c), A/B slots (active = a)
- Bootloader unlocked, rooted with KernelSU

## Environment
- Works ONLY on the tablet. No PC, no ADB.
- Linux env: Droidspaces container `Alph` (Alpine)
- `pmbootstrap install` fails inside Droidspaces (block ioctls blocked)
- Build pipeline: GitHub Actions (primary) + CircleCI, ARM64 runners

## Repo
https://github.com/Classity-Real/xun-pmos (public)
Maintainer: Classity-Real <uiportsarchives@gmail.com>
Local (Droidspaces): /root/xun-pmos

## Repo layout
.github/workflows/build.yml                GitHub Actions
.circleci/config.yml                        CircleCI
device/kernel-package/APKBUILD              Kernel package
device/kernel-package/config-postmarketos-qcom-sm6225.aarch64
device/kernel-package/sm6225-xiaomi-xun.dts our DTS
device/references/xun-downstream.dts        Full downstream DTS (790KB)
device/references/sm6225-lenovo-tb128fu.dts Lenovo reference
device/testing/device-xiaomi-xun/APKBUILD
device/testing/device-xiaomi-xun/deviceinfo
device/testing/device-xiaomi-xun/modules-initfs
NOTES.md                                    This file

## Build trigger (both platforms)
Only fires on `ci-trigger` branch:
  git push origin main                 # no build
  git push -f origin main:ci-trigger   # build

## Current state

### Working
- Kernel builds successfully (~21 min ARM64 cold)
- ccache_size = 5G confirmed in effective config
- pmbootstrap 3.12 config CLI keys: aports, ccache_size, device,
  service_manager, ui, kernel, is_default_channel, jobs, work

### Failing
- `pmbootstrap install` fails at `mkinitfs` step
- Export dir empty -> upload warns "No files found"
- Need log.txt tail to see actual mkinitfs error

### pmbootstrap 3.12 config keys (correct)
- `service_manager` = openrc   (NOT `systemd`; doesn't exist in 3.12)
- `is_default_channel` = True  (edge; False = systemd-edge)
- `ccache_size` = 5G           (REQUIRED for ccache to enable at all)

## DTS (sm6225-xiaomi-xun.dts)
Based on Lenovo TB128FU. Key values:
- qcom,board-id = <0x20022 0x00>
- qcom,msm-id    = <0x206 0x10000>
- model = "Xiaomi Pad SE"
- compatible = "xiaomi,xun", "qcom,sm6225"
- simple-framebuffer @ 0x5c000000, 1200x1920, stride 4800, a8r8g8b8
- Compiles to 5.8 KB DTB

## Kernel fork
TQMatvey/linux-sm6225, commit 849f3f471b6268280dc8418e0bddda435ae5d4a5
Reports kernel version 6.1.0-sm6125 (fork quirk)
sm6225.dtsi is minimal (367 lines) - no UART, display, or storage driver
Lenovo DTS is 40 lines - uses simple-framebuffer

## Downstream DTS values
From /proc/device-tree -> /root/xun.dts (790KB):
  model = "Qualcomm Technologies, Inc. KHAJE IDP nopmi xun"
  compatible = "qcom,khaje-idp", "qcom,khaje", "qcom,idp"
  qcom,board-id = <0x20022 0x00>
  qcom,msm-id    = <0x206 0x10000>

  Memory (8GB):
    <0x00 0x40000000 0x00 0x3bb00000>
    <0x01 0x40000000 0x01 0x00>
    <0x00 0x80000000 0x00 0xc0000000>

  Debug UART: qcom,qup_uart@4a90000 (SE4), IRQ 331
  Console: console=ttyMSM0,115200n8
  Splash: 0x5c000000, size 0xf00000
  Panel: m84_42_03_0c_fhdp_video, 1200x1920 (0x4b0 x 0x780)

## Workflow steps (GitHub Actions; CircleCI mirrors)
1.  checkout
2.  actions/cache /home/runner/.local/var/pmbootstrap/cache_ccache
3.  apt deps: multipath-tools parted e2fsprogs dtc libsparse-utils
4.  clone pmbootstrap -> $HOME/pmbootstrap, symlink /usr/local/bin
5.  clone pmaports -> $WORK/cache_git/pmaports, `echo 8 > $WORK/version`
6.  copy device + kernel packages into pmaports (from repo)
7.  write config via `pmbootstrap --as-root config <key> <value>`
8.  mkdir/chmod cache_ccache
9.  sudo -E pmbootstrap --as-root build --force linux-postmarketos-qcom-sm6225
10. sudo -E pmbootstrap --as-root build --force device-xiaomi-xun
11. sudo -E pmbootstrap --as-root install --password=147147
12. sudo -E pmbootstrap --as-root export
13. save ccache
14. check ccache state
15. verify export contents
16. dump pmbootstrap log (config + log.txt tail 200 + export dir)
17. fix artifact permissions
18. upload-artifact /tmp/postmarketOS-export/

## Next steps
1. Check "Dump pmbootstrap log" step output from last failed run.
   log.txt tail will show the actual mkinitfs error.
2. Common mkinitfs failures:
   - `/boot/dtbs/qcom/sm6225-xiaomi-xun.dtb not found` -> awk didn't inject
     Makefile line correctly (needs real TAB, not spaces)
   - Missing kernel modules referenced in modules-initfs
   - boot-deploy can't find boot.img output path
3. If DTB wasn't built, verify awk in device/kernel-package/APKBUILD:
     awk '/sm6225-lenovo-tb128fu\.dtb/ {
             print
             print "dtb-$(CONFIG_ARCH_QCOM)\t+= sm6225-xiaomi-xun.dtb"
             next
     }
     { print }' Makefile > /tmp/mk && mv /tmp/mk Makefile
4. After export succeeds: download artifacts, use `fastboot boot boot.img`
   (do NOT flash boot_a yet). Backups needed first:
     su
     dd if=/dev/block/by-name/boot_a of=/sdcard/boot_a-backup.img
     dd if=/dev/block/by-name/vbmeta_a of=/sdcard/vbmeta_a-backup.img
     dd if=/dev/block/by-name/dtbo_a of=/sdcard/dtbo_a-backup.img
   Copy off-device.

## Rules for assistant
- No ADB (no PC). Commands must work on-device or in CI.
- No "use a Linux PC" — workflow is on-device + GitHub Actions/CircleCI.
- Copy-paste ready commands, no placeholders.
- When unsure about an error, ask for log dump rather than guessing.
- Don't recommend flashing boot_a until backups verified off-device.
- Keep both CI configs in sync.

## Past errors (fixed)
| Error | Fix |
|---|---|
| partprobe: Operation not permitted (Droidspaces) | Move to CI |
| deviceinfo_bootimg_qcdt -> /boot/dt.img not found | Set qcdt=false |
| work folder version needs migration | echo 8 > $WORK/version |
| Config file ignored (systemd-edge fallback) | Use pmbootstrap config CLI |
| pmbootstrap config systemd invalid | Use service_manager |
| pmbootstrap config channel invalid | Use is_default_channel True |
| mkdir /root permission denied | sudo mkdir -p /root/.config |
| CircleCI << heredoc parse error | Use printf, not heredoc |
| qemu-amd64.img in artifact | Write config file directly |
| Kernel build 1h27m on x86 | Switch to ubuntu-24.04-arm |
| ccache never populated | Add ccache_size 5G to config |

## Open issues
1. mkinitfs failing during pmbootstrap install - need log.txt tail
2. Export dir empty -> depends on install succeeding
3. CircleCI ARM runners often stuck "Blocked" - prefer GitHub Actions

## Status
- Kernel compiles                    YES (21 min ARM cold)
- Device package builds              YES
- Rootfs install completes           NO  (mkinitfs error)
- Artifacts downloadable             NO  (empty export dir)
- Image ever flashed                 NO  (never tested on device)
