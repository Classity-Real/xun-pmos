# Xiaomi Pad SE (xun) postmarketOS Port

## Device
- Xiaomi Pad SE, codename `xun`, SoC Qualcomm SM6225 (Khaje)
- 8 GB RAM, UFS, 1200x1920 panel (m84_*), A/B slots
- Bootloader unlocked

## Environment
- Work is done on the tablet (Droidspaces) + second Android for fastboot
- No PC / no ADB required for flashing
- CI: GitHub Actions on `ci-trigger` branch (`ubuntu-24.04-arm`)

## Repo
https://github.com/Classity-Real/xun-pmos

## Boot strategy (current — do not change without hardware evidence)

| Image | Source | Why |
|-------|--------|-----|
| **boot.img** | CI (pmOS) | Mainline kernel + **appended DTB** (`append_dtb=true`) |
| **vendor_boot.img** | **STOCK only** | Bootloader rejects custom vendor_boot (even 96 MB padded) |

Kernel package patches `head.S` so the kernel **ignores the bootloader FDT** and uses the DTB appended to `Image`. That is how we can keep stock vendor_boot.

### Flash
```
fastboot flash boot_a boot.img
fastboot flash vendor_boot_a <STOCK-vendor_boot.img>
fastboot set_active a
fastboot reboot
```

### Recovery if stuck
```
fastboot flash boot_a <stock-boot.img>
fastboot flash vendor_boot_a <stock-vendor_boot.img>
fastboot reboot
```
Then check `/sys/fs/pstore/` for ramoops.

## Stock image facts
- boot.img: ANDROID! header v4, ~128 MB partition
- vendor_boot.img: VNDRBOOT header v4, **96 MB** (100663296), cmdline starts with `video=vfb:640x400...`
- Debug UART: `qcom,qup_uart@4a90000`, `console=ttyMSM0,115200n8`
- board-id / msm-id: `<0x20022 0x00>` / `<0x206 0x10000>`
- Splash: `0x5c000000` size `0xf00000`

## Kernel
- Fork: TQMatvey/linux-sm6225 @ 849f3f47…
- Bundle replaces stripped sm6225.dtsi with mainline sm6115-family dtsi + dt-bindings
- DTB target size after bundle: **~51 KB** (was ~6 KB without bundle)

## What works / does not
- Kernel + device package build in CI: YES
- Artifacts produced: YES (run #71 and later)
- Custom vendor_boot accepted: **NO**
- Boot past logo with stock vendor_boot + mainline boot: testing
- Display / touch / WiFi: not yet (no panel driver on mainline for m84)

## Trigger CI
```
git push origin main
git push -f origin main:ci-trigger
```

## Lessons from Seuj09/pmos-infinix-x6525
- Document every stock header / live DT value
- header_version=4 does embed DTB in generated vendor_boot when boot-deploy runs — but Xiaomi still rejects non-stock vendor_boot content/structure
- Prefer a path that keeps the stock vendor partition when the bootloader is strict
- ramoops + FIRST_BOOT-style checklist for bring-up
