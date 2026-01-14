# AGENTS.md - bl-mt798x-dhcpd

> Instructions for AI coding agents working on this codebase.

## Project Overview

This is a **MediaTek MT798x bootloader project** (U-Boot + ARM Trusted Firmware) for router devices. It's a fork of hanwckf's bl-mt798x with DHCP support and WebUI enhancements.

**Primary languages**: C (embedded), Shell scripts, Device Tree Source (.dts)
**Target**: ARM64 (aarch64-linux-gnu)
**Supported SoCs**: MT7981, MT7986

## Build Commands

### Prerequisites
```bash
sudo apt install gcc-aarch64-linux-gnu build-essential flex bison libssl-dev device-tree-compiler qemu-user-static python2.7
```

### Main Build (Firmware Image Package)
```bash
# Format: SOC=<soc> BOARD=<board> VERSION=<version> ./build.sh
SOC=mt7981 BOARD=360t7 VERSION=2022 ./build.sh
SOC=mt7981 BOARD=cmcc_a10 VERSION=2024 ./build.sh
SOC=mt7986 BOARD=redmi_ax6000 VERSION=2025 ./build.sh

# With multi-layout support (for specific devices)
SOC=mt7981 BOARD=wr30u VERSION=2023 MULTI_LAYOUT=1 ./build.sh
```

**Version mapping:**
| VERSION | ATF Directory | U-Boot Directory |
|---------|---------------|------------------|
| 2022 | atf-20220606-637ba581b | uboot-mtk-20220606 |
| 2023 | atf-20231013-0ea67d76a | uboot-mtk-20230718-09eda825 |
| 2024 | atf-20240117-bacca82a8 | uboot-mtk-20230718-09eda825 |
| 2025 | atf-20250711 | uboot-mtk-20250711 |

### GPT Generation
```bash
# Requires python2.7
./generate_gpt.sh
SDMMC=1 ./generate_gpt.sh  # For SD/MMC support
```

### Output
- Build artifacts: `output/` directory
- GPT images: `output_gpt/` directory

## Testing

### U-Boot Tests (in uboot-mtk-20250711/)
```bash
cd uboot-mtk-20250711
make tests          # Run all sandbox tests
make qcheck         # Quick tests
./test/run          # Execute test runner directly
```

### No formal test suite for ATF
ATF is low-level firmware - testing is done on actual hardware.

## Code Style

### C Code (U-Boot)
- **License**: SPDX headers required at file start
  ```c
  // SPDX-License-Identifier: GPL-2.0+
  ```
- **Style**: Linux kernel coding style
- **Line length**: 100 characters max (checkpatch default)
- **Indentation**: Tabs (8 spaces width)
- **Checker**: `scripts/checkpatch.pl`
  ```bash
  ./scripts/checkpatch.pl --file path/to/file.c
  ./scripts/checkpatch.pl --git HEAD~1  # Check last commit
  ```

### C Code (ATF)
- **License**: BSD-3-Clause (SPDX header)
  ```c
  // SPDX-License-Identifier: BSD-3-Clause
  ```
- **Formatter**: `.clang-format` in atf-20250711/
  - Column limit: 80
  - Braces: After function, not after control statements
  - Indent: Tabs
- **Checker**: `make checkcodebase` or `make checkpatch`

### Shell Scripts
- Use `/bin/sh` or `/bin/bash` shebang
- Quote variables: `"$VAR"` not `$VAR`
- Check command existence before use

### Device Tree (.dts)
- Follow Linux kernel DTS conventions
- Use proper node naming: `node-name@unit-address`

## Directory Structure

```
bl-mt798x-dhcpd/
├── build.sh                    # Main build script
├── generate_gpt.sh             # GPT image generator
├── show_gpt.sh                 # GPT info viewer
├── atf-20220606-637ba581b/     # ATF version 2022
├── atf-20231013-0ea67d76a/     # ATF version 2023
├── atf-20240117-bacca82a8/     # ATF version 2024
├── atf-20250711/               # ATF version 2025
├── uboot-mtk-20220606/         # U-Boot version 2022
├── uboot-mtk-20230718-09eda825/# U-Boot version 2023/2024
├── uboot-mtk-20250711/         # U-Boot version 2025
├── mt798x_gpt/                 # GPT partition configs
├── modify-patch/               # Optional patches
├── depends/                    # Build dependencies list
├── document/                   # Documentation and images
└── output/                     # Build output (gitignored)
```

## Adding New Board Support

### 1. Create ATF defconfig
Location: `atf-<version>/configs/<soc>_<board>_defconfig`
```
CONFIG_PLAT_MT7981=y
CONFIG_TARGET_FIP_NO_SEC_BOOT=y
CONFIG_FLASH_DEVICE_SPIM_NAND=y
CONFIG_BGA=y
CONFIG_LOG_LEVEL_INFO=y
```

### 2. Create U-Boot defconfig
Location: `uboot-<version>/configs/<soc>_<board>_defconfig`
- Start from similar board config
- Set `CONFIG_DEFAULT_DEVICE_TREE`
- Configure flash type, partition layout

### 3. Create Device Tree
Location: `uboot-<version>/arch/arm/dts/<soc>-<board>.dts`

### 4. Test Build
```bash
SOC=<soc> BOARD=<board> VERSION=<version> ./build.sh
```

## Important Patterns

### Defconfig Naming
- ATF: `<soc>_<board>_defconfig`
- U-Boot: `<soc>_<board>_defconfig`
- Multi-layout variant: `<soc>_<board>_multi_layout_defconfig`

### Flash Types
- `SPIM_NAND`: SPI NAND flash
- `SPIM_NOR`: SPI NOR flash
- `SNFI_NAND`: SNFI NAND
- `EMMC`: eMMC storage

### DHCP Customization
DHCP settings in `uboot-mtk-*/net/mtk_dhcpd.c`:
- `DHCPD_DEFAULT_IP_STR`: Router IP (default: 192.168.33.1)
- `DHCPD_POOL_START_STR`: Pool start (default: 192.168.33.100)
- `DHCPD_POOL_END_STR`: Pool end (default: 192.168.33.200)

## CI/CD

GitHub Actions workflows:
- `.github/workflows/FIP-build.yml` - Main FIP build
- `.github/workflows/FIP-FIT-build.yml` - FIT image build

Build matrix runs on Ubuntu 22.04.

## Common Issues

### Build Fails: Cross-compiler not found
```bash
sudo apt install gcc-aarch64-linux-gnu
export CROSS_COMPILE=aarch64-linux-gnu-
```

### Build Fails: Python not found
```bash
sudo apt install python2.7  # For GPT tools
sudo apt install python3    # For U-Boot tools
```

### Defconfig not found
Ensure both ATF and U-Boot defconfigs exist:
- `atf-<version>/configs/<soc>_<board>_defconfig`
- `uboot-<version>/configs/<soc>_<board>_defconfig`

## Patches

Available patches in `modify-patch/`:
- `0001-overclocking-mt7981-to-1.6GHz-by-modify-atf.patch` - MT7981 overclock
- `0002-uboot-2022-fit-merge-code-from-1715173329-to-support.patch` - FIT image support

Apply with: `git apply modify-patch/<patch-file>`

## Key Files for Common Tasks

| Task | Files |
|------|-------|
| Add new board | `*/configs/*_defconfig`, `*/arch/arm/dts/*.dts` |
| Modify boot menu | `uboot-*/cmd/bootmenu*.c` |
| Change DHCP behavior | `uboot-*/net/mtk_dhcpd.c` |
| Modify WebUI | `uboot-*/httpd/` |
| GPT partitions | `mt798x_gpt/*.json` |
| Build process | `build.sh` |
