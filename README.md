# RTL8196C DAP-1360 D1 Bootloader

Custom bootloader for **D-Link DAP-1360 (H/W revision D1)** based on Realtek RTL8196C SoC.

## Hardware

| Component | Specification |
|-----------|---------------|
| SoC | Realtek RTL8196C (RLX4181, Lexra LX4181) |
| CPU | MIPS32, 389 MHz |
| RAM | 32 MB SDR SDRAM |
| Flash | 4 MB SPI NOR (Winbond W25Q32) |
| WiFi | RTL8192CD (2.4 GHz 802.11n, 2x2 MIMO) |
| Ethernet | 5-port 10/100 switch (2 ports on board) |
| Serial | 38400 baud, ttyS0 |

## Flash Layout

| Offset | Size | Partition | Content |
|--------|------|-----------|---------|
| 0x00000 | 64 KB | boot | Bootloader (this project) |
| 0x10000 | 64 KB | MAC | MAC address, HW settings |
| 0x20000 | 64 KB | config | NVRAM, bootinfo |
| 0x30000 | 1 MB | kernel | Linux kernel (cr6b header) |
| 0x130000 | 2.8 MB | rootfs | SquashFS root filesystem |

## Building

### Prerequisites

- Docker with `openwrt-1407-builder` image
- RSDK toolchain (auto-downloaded by Makefile)

### Build

```bash
# Using Docker
docker run -d --name builder openwrt-1407-builder sleep infinity
docker cp bootcode_rtl8196c_98 builder:/build/
docker exec builder sh -c "cd /build/bootcode_rtl8196c_98 && make clean && make"
```

### Build Output

| File | Description |
|------|-------------|
| `bin/boot.bin` | Bootcode with header (for HTTP upload) |
| `bin/boot` | Raw bootcode binary |
| `bin/wboot.img` | Wrapped boot image |

## Flashing

### Via HTTP (if bootloader is running)

Upload `bin/boot.bin` through the bootloader's web interface at the router's IP address.

### Via SPI Programmer (CH341A)

1. Connect CH341A to the SPI flash chip (Winbond W25Q32)
2. Read current flash content (backup)
3. Write `bin/boot.bin` to offset 0x0

## Reset Button Behavior

The DAP-1360 has a single button (Port B, GPIO pin 3) with dual function:

| Hold Duration | Action |
|---------------|--------|
| < 4 seconds | Normal boot |
| 4–8 seconds | Reset config to defaults |
| ≥ 8 seconds | Crash mode (TFTP/HTTP download mode) |

## Configuration

Key settings in `.config`:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `CONFIG_RTL8196C` | y | RTL8196C SoC |
| `CONFIG_D32_16` | y | 32 MB SDRAM |
| `CONFIG_SPI_FLASH` | y | SPI flash mode |
| `CONFIG_BOOT_CODE_SIZE` | 0x8000 | Max bootcode size (32 KB) |
| `CONFIG_LZMA_ENABLE` | y | LZMA compression |
| `CONFIG_DHCP_SERVER` | y | Built-in DHCP server |
| `CONFIG_HTTP_SERVER` | y | Built-in HTTP server |

## Flash Offsets (utility.h)

| Offset | Name | Description |
|--------|------|-------------|
| 0x6000 | HS_IMAGE_OFFSET | Hardware settings |
| 0x8000 | DS_IMAGE_OFFSET | Default settings |
| 0xC000 | CS_IMAGE_OFFSET | Current settings |
| 0x10000 | CODE_IMAGE_OFFSET | Kernel search start |
| 0x30000 | CODE_IMAGE_OFFSET3 | Kernel (primary) |
| 0xE0000 | ROOT_FS_OFFSET | Root filesystem search start |

## Stock Firmware Compatibility

This bootloader is compatible with the stock D-Link DAP-1360 firmware (v3.0.0). However, the stock firmware's `updateboot` routine (in `/lib/libdhal.so`) will overwrite the bootloader on first boot. To prevent this, build a custom OpenWrt firmware without the `updateboot` routine.

## References

- [OpenWrt SDK for DAP-1360](../14.07/openwrt-14.07-dap1360/)
- [Stock bootloader v1.2 release](https://github.com/kiper292/rtl8196c-dap1360-d1-bootloader/releases/tag/stock-bootloader-v1.2)
