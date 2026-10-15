# Xiaomi Pad SE (xun) postmarketOS Port

## Boot strategy
- **boot.img** from CI (mainline kernel + appended DTB)
- **vendor_boot.img** = STOCK only (custom is rejected)
- Kernel head.S ignores bootloader FDT

## Flash
```
fastboot flash boot_a boot.img
fastboot flash vendor_boot_a <STOCK-vendor_boot.img>
fastboot set_active a
fastboot reboot
```

CI: push to `ci-trigger` branch.
