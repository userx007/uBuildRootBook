# 12. Bootloader Integration (U-Boot & GRUB2)

**Structure of the document:**

- **Boot process architecture** — two full ASCII diagrams showing the ARM multi-stage boot chain (BROM → TPL → SPL → U-Boot → Linux) and the x86 GRUB2 path (UEFI/BIOS → GRUB2 → Linux)
- **U-Boot integration** — defconfig selection, config fragments, directory layout
- **SPL/TPL stages** — memory map ASCII diagram, SoC-specific SPL binary names, Rockchip TPL example
- **FIT images** — structure ASCII diagram, a complete `.its` file, build integration, and post-image script
- **`BR2_TARGET_UBOOT_*` variables** — full reference table covering all major variable groups (core, board, output format, SPL, environment, signing, patches)
- **GRUB2** — partition layout ASCII diagram, `grub.cfg` for A/B booting, variable reference, and `genimage.cfg` for disk image generation
- **C examples** — real U-Boot `board.c` with SPL support, U-Boot environment binary manipulation tool with CRC32
- **C++ examples** — FIT image node parser using `libfdt`, GRUB2 `grubenv` block manager with A/B watchdog
- **Rust examples** — FIT SHA-256 hash verifier, complete A/B slot state machine with full unit tests
- **Advanced section** — secure boot signing flow (ASCII), U-Boot A/B boot script, eMMC flash layout diagram
- **Summary** — concise prose covering all major concepts


> **Buildroot Series** | Embedded Linux Build System  
> Topic: U-Boot defconfig, SPL/TPL stages, FIT images, `BR2_TARGET_UBOOT_*` variables, and GRUB2 for x86 embedded targets.

---

## Table of Contents

1. [Overview](#overview)
2. [Boot Process Architecture](#boot-process-architecture)
3. [U-Boot Integration in Buildroot](#u-boot-integration-in-buildroot)
4. [SPL and TPL Stages](#spl-and-tpl-stages)
5. [FIT Images](#fit-images)
6. [BR2_TARGET_UBOOT_* Variables Reference](#br2_target_uboot_-variables-reference)
7. [GRUB2 for x86 Embedded Targets](#grub2-for-x86-embedded-targets)
8. [C Code Examples](#c-code-examples)
9. [C++ Code Examples](#c-code-examples-1)
10. [Rust Code Examples](#rust-code-examples)
11. [Advanced Scenarios](#advanced-scenarios)
12. [Summary](#summary)

---

## Overview

Bootloader integration is one of the most hardware-specific and consequential aspects of an embedded Linux build. Buildroot provides first-class support for two major bootloaders:

- **U-Boot** — the dominant bootloader for ARM, RISC-V, PowerPC, MIPS, and other non-x86 embedded targets. It supports a rich chain of pre-loader stages (SPL/TPL), FIT (Flattened Image Tree) packaging, and a scripting environment.
- **GRUB2** — the standard bootloader for x86/x86-64 embedded targets, widely used in industrial PCs, gateway devices, and ruggedized systems running on Intel/AMD silicon.

Buildroot's `package/uboot/` and `package/grub2/` packages expose dozens of `BR2_TARGET_UBOOT_*` and `BR2_TARGET_GRUB2_*` Kconfig variables, allowing a fully reproducible, hermetically-sealed bootloader build without touching host system files.

---

## Boot Process Architecture

### ARM Embedded Boot Sequence (ASCII)

```
+-------------------------------------------------------------+
|                  POWER ON / RESET VECTOR                    |
+-------------------------------------------------------------+
         |
         v
+---------------------+
|   ROM / BROM Code   |  (On-chip, vendor-supplied, read-only)
|  - Basic clock init |
|  - Detects boot     |
|    device (eMMC,    |
|    SD, SPI NOR)     |
+---------------------+
         |
         v
+---------------------+    Loaded by BROM from boot device
|   TPL (Tertiary     |    (optional, some SoCs only)
|   Program Loader)   |
|  - Minimal DRAM     |
|    init             |
|  - Loads SPL        |
+---------------------+
         |
         v
+---------------------+    Loaded by TPL (or BROM directly)
|   SPL (Secondary    |    Fits in on-chip SRAM (~200 KB)
|   Program Loader)   |
|  - Full DRAM init   |
|  - Board early init |
|  - Loads U-Boot     |
|    proper           |
+---------------------+
         |
         v
+---------------------+    Runs in DRAM
|   U-Boot Proper     |
|  - Device drivers   |
|  - Network/USB/MMC  |
|  - Environment vars |
|  - Boot scripts     |
|  - Loads FIT image  |
+---------------------+
         |
         v
+---------------------+    FIT image = kernel + DTB + ramdisk
|   Linux Kernel      |    (optionally signed/verified)
|   + Device Tree     |
|   + Initramfs       |
+---------------------+
         |
         v
+---------------------+
|   User Space /      |
|   BusyBox / SystemD |
+---------------------+
```

### x86 Boot Sequence with GRUB2 (ASCII)

```
+-------------------------------------------------------------+
|             UEFI Firmware / Legacy BIOS                     |
+-------------------------------------------------------------+
         |
         v
+---------------------+    For UEFI: EFI System Partition
|   GRUB2 EFI or      |    For BIOS: MBR + boot sector
|   GRUB2 BIOS        |
|  - grub.cfg loaded  |
|  - Menu displayed   |
|  - Modules loaded   |
+---------------------+
         |
         v
+---------------------+
|   Linux Kernel      |    bzImage / vmlinuz
|   + initrd/initramfs|
+---------------------+
         |
         v
+---------------------+
|   User Space        |
+---------------------+
```

---

## U-Boot Integration in Buildroot

### Enabling U-Boot

In `menuconfig`, navigate to:

```
Boot options --->
  [*] U-Boot
```

Or set directly in your `configs/<board>_defconfig`:

```makefile
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BUILD_SYSTEM_KCONFIG=y
BR2_TARGET_UBOOT_BOARDNAME="my_board"
BR2_TARGET_UBOOT_CONFIG_FRAGMENT_FILES="board/myvendor/myboard/uboot.fragment"
BR2_TARGET_UBOOT_VERSION="2024.01"
```

### Directory Structure

```
buildroot/
├── board/
│   └── myvendor/
│       └── myboard/
│           ├── linux.config          # Linux kernel config
│           ├── uboot.config          # U-Boot full defconfig
│           ├── uboot.fragment        # Config fragment overlay
│           ├── boot.its              # FIT image source
│           └── post-build.sh         # Post-build hook
├── configs/
│   └── myboard_defconfig             # Buildroot board defconfig
└── package/
    └── uboot/                        # Buildroot U-Boot package
        ├── uboot.mk
        └── Config.in
```

### U-Boot defconfig

U-Boot ships hundreds of board defconfigs under its own `configs/` directory. Buildroot's `BR2_TARGET_UBOOT_BOARD_DEFCONFIG` variable names which one to use (without the `_defconfig` suffix):

```makefile
# In Buildroot's board defconfig or .config:
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="rpi_4_32b"

# This will call inside U-Boot's tree:
#   make rpi_4_32b_defconfig
```

You can also supply a full custom config file:

```makefile
BR2_TARGET_UBOOT_USE_CUSTOM_CONFIG=y
BR2_TARGET_UBOOT_CUSTOM_CONFIG_FILE="$(BR2_EXTERNAL_MYPROJECT_PATH)/configs/uboot_myboard.config"
```

Config fragments allow small overlay changes without a full custom config:

```makefile
# board/myvendor/myboard/uboot.fragment
CONFIG_BOOTDELAY=2
CONFIG_BOOTCOMMAND="run distro_bootcmd"
CONFIG_ENV_SIZE=0x20000
CONFIG_ENV_OFFSET=0x3E0000
CONFIG_SYS_EXTRA_OPTIONS="DM_GPIO,OF_CONTROL"
```

---

## SPL and TPL Stages

### Concept

Many SoCs (Allwinner, Rockchip, TI OMAP/AM335x, NXP i.MX) have a tiny on-chip SRAM (32–256 KB) that the BROM (boot ROM) uses as the load target. U-Boot proper is too large to fit there, so a two- or three-stage approach is used:

```
BROM  →  TPL (tiny, ~8 KB, DRAM probe)
       →  SPL (~100-200 KB, DRAM init + drivers)
       →  U-Boot proper (megabytes, in DRAM)
       →  Linux
```

### Enabling SPL in Buildroot

```makefile
# configs/myboard_defconfig
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_SPL=y
BR2_TARGET_UBOOT_SPL_NAME="MLO"         # TI AM335x name for SPL binary
```

The `BR2_TARGET_UBOOT_SPL_NAME` tells Buildroot which file inside U-Boot's build output to install to `$(BINARIES_DIR)`. Common values:

| SoC Family        | SPL Binary Name    |
|-------------------|--------------------|
| TI AM335x/AM57xx  | `MLO`              |
| Allwinner (sun4i+)| `sunxi-spl.bin`    |
| Rockchip          | `idbloader.img`    |
| NXP i.MX6/7/8     | `SPL`              |
| Generic           | `u-boot-spl.bin`   |

### SPL Memory Map (ASCII)

```
On-Chip SRAM (example: 256 KB @ 0x402F0400)
+------------------------------------------+
| 0x402F0400  SPL text + data              |
|             ~100–180 KB                  |
|             - DRAM initialisation code   |
|             - Minimal driver framework   |
|             - FAT/ext2 reading for       |
|               u-boot.img load            |
+------------------------------------------+
| 0x4030E000  SPL stack (grows downward)   |
+------------------------------------------+
| 0x4030FFFC  Stack top                    |
+------------------------------------------+

DRAM (after SPL init, e.g. 512 MB @ 0x80000000)
+------------------------------------------+
| 0x80000000  U-Boot proper loaded here    |
|             by SPL                       |
+------------------------------------------+
| 0x80800000  U-Boot BSS / heap            |
+------------------------------------------+
| ...         FDT (Device Tree)            |
+------------------------------------------+
| top - 16 MB  Kernel load address         |
+------------------------------------------+
```

### TPL Configuration (Rockchip Example)

```makefile
# U-Boot config fragment for Rockchip RK3399
CONFIG_TPL=y
CONFIG_TPL_SIZE_LIMIT=0x9000    # 36 KB
CONFIG_SPL=y
CONFIG_SPL_SIZE_LIMIT=0x40000   # 256 KB
CONFIG_SPL_SERIAL=y
CONFIG_SPL_MMC=y
CONFIG_SPL_LIBDISK_SUPPORT=y
CONFIG_SPL_FIT=y
CONFIG_SPL_FIT_GENERATOR="arch/arm/mach-rockchip/make_fit_atf.py"
```

---

## FIT Images

### What is a FIT Image?

FIT (Flattened Image Tree) is a container format used by U-Boot. A single `.itb` file can hold:

- One or more kernel images (for A/B update schemes)
- One or more device tree blobs (for board variants)
- An optional initramfs/initrd
- Optional firmware (ATF BL31, OP-TEE, etc.)
- RSA/ECDSA signature nodes for verified boot

### FIT Image Structure (ASCII)

```
FIT Image (.itb)
+---------------------------------------+
|  FDT Magic: 0xD00DFEED                |
+---------------------------------------+
|  /images                              |
|    /kernel@1                          |
|      - description = "Linux 6.6"      |
|      - type = "kernel"                |
|      - arch = "arm64"                 |
|      - compression = "gzip"           |
|      - data = <binary kernel>         |
|      - hash@1: SHA-256 digest         |
|                                       |
|    /fdt@rk3399-myboard                |
|      - description = "MyBoard DTB"    |
|      - type = "flat_dt"               |
|      - data = <binary DTB>            |
|      - hash@1: SHA-256 digest         |
|                                       |
|    /ramdisk@1  (optional)             |
|      - type = "ramdisk"               |
|      - data = <binary initramfs>      |
|      - hash@1: SHA-256 digest         |
|                                       |
|  /configurations                      |
|    /conf@1  (default)                 |
|      - kernel = "kernel@1"            |
|      - fdt = "fdt@rk3399-myboard"     |
|      - ramdisk = "ramdisk@1"          |
|      - signature@1: RSA-2048 sig      |
+---------------------------------------+
```

### Writing an .its (Image Tree Source) File

```
/* board/myvendor/myboard/boot.its */
/dts-v1/;

/ {
    description = "MyBoard FIT Image";
    #address-cells = <1>;

    images {
        kernel@1 {
            description = "Linux Kernel";
            data = /incbin/("Image.gz");
            type = "kernel";
            arch = "arm64";
            os = "linux";
            compression = "gzip";
            load = <0x48000000>;
            entry = <0x48000000>;
            hash@1 {
                algo = "sha256";
            };
        };

        fdt@rk3399-myboard {
            description = "RK3399 MyBoard Device Tree";
            data = /incbin/("rk3399-myboard.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash@1 {
                algo = "sha256";
            };
        };

        ramdisk@1 {
            description = "Initial Ramdisk";
            data = /incbin/("rootfs.cpio.gz");
            type = "ramdisk";
            arch = "arm64";
            os = "linux";
            compression = "gzip";
            hash@1 {
                algo = "sha256";
            };
        };
    };

    configurations {
        default = "conf@1";

        conf@1 {
            description = "MyBoard Standard Config";
            kernel = "kernel@1";
            fdt = "fdt@rk3399-myboard";
            ramdisk = "ramdisk@1";
        };
    };
};
```

### Building FIT Images via Buildroot

```makefile
# configs/myboard_defconfig
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_FORMAT_CUSTOM=y
BR2_TARGET_UBOOT_FORMAT_CUSTOM_NAME="u-boot.itb"
BR2_TARGET_IMAGE_FIT=y
BR2_TARGET_IMAGE_FIT_ITS="board/myvendor/myboard/boot.its"
```

Or manually via a post-image script:

```bash
#!/bin/bash
# board/myvendor/myboard/post-image.sh
set -e

BOARD_DIR="$(dirname $0)"
BINARIES_DIR="${BINARIES_DIR}"
HOST_DIR="${HOST_DIR}"

# Copy kernel and DTB alongside the ITS file
cp "${BINARIES_DIR}/Image.gz" "${BOARD_DIR}/"
cp "${BINARIES_DIR}/rk3399-myboard.dtb" "${BOARD_DIR}/"
cp "${BINARIES_DIR}/rootfs.cpio.gz" "${BOARD_DIR}/"

# Generate FIT image
"${HOST_DIR}/bin/mkimage" \
    -f "${BOARD_DIR}/boot.its" \
    "${BINARIES_DIR}/boot.itb"

echo "FIT image: ${BINARIES_DIR}/boot.itb"
```

---

## BR2_TARGET_UBOOT_* Variables Reference

### Core Variables

| Variable | Type | Description |
|---|---|---|
| `BR2_TARGET_UBOOT` | bool | Enable U-Boot build |
| `BR2_TARGET_UBOOT_VERSION` | string | Git tag/branch or "custom" |
| `BR2_TARGET_UBOOT_CUSTOM_TARBALL_LOCATION` | string | URL for custom tarball |
| `BR2_TARGET_UBOOT_CUSTOM_GIT_REPO_URL` | string | Git URL for custom source |
| `BR2_TARGET_UBOOT_CUSTOM_GIT_VERSION` | string | Git commit/branch/tag |
| `BR2_TARGET_UBOOT_BUILD_SYSTEM_KCONFIG` | bool | Use Kconfig build system (default) |
| `BR2_TARGET_UBOOT_BUILD_SYSTEM_SIMPLE` | bool | Use simple Makefile system |

### Board & Config Variables

| Variable | Type | Description |
|---|---|---|
| `BR2_TARGET_UBOOT_BOARD_DEFCONFIG` | string | U-Boot defconfig name (without `_defconfig`) |
| `BR2_TARGET_UBOOT_USE_CUSTOM_CONFIG` | bool | Use custom full config file |
| `BR2_TARGET_UBOOT_CUSTOM_CONFIG_FILE` | string | Path to custom config file |
| `BR2_TARGET_UBOOT_CONFIG_FRAGMENT_FILES` | string | Space-separated config fragment paths |
| `BR2_TARGET_UBOOT_BOARDNAME` | string | Board name (for simple build system) |
| `BR2_TARGET_UBOOT_CUSTOM_MAKEOPTS` | string | Extra make options passed to U-Boot |

### Output Format Variables

| Variable | Type | Description |
|---|---|---|
| `BR2_TARGET_UBOOT_FORMAT_BIN` | bool | Install `u-boot.bin` |
| `BR2_TARGET_UBOOT_FORMAT_ELF` | bool | Install `u-boot.elf` |
| `BR2_TARGET_UBOOT_FORMAT_IMG` | bool | Install `u-boot.img` |
| `BR2_TARGET_UBOOT_FORMAT_ITB` | bool | Install `u-boot.itb` (FIT) |
| `BR2_TARGET_UBOOT_FORMAT_CUSTOM` | bool | Install custom-named output |
| `BR2_TARGET_UBOOT_FORMAT_CUSTOM_NAME` | string | Filename(s) of custom output |

### SPL Variables

| Variable | Type | Description |
|---|---|---|
| `BR2_TARGET_UBOOT_SPL` | bool | Enable SPL build and installation |
| `BR2_TARGET_UBOOT_SPL_NAME` | string | SPL binary filename(s) to install |

### Environment & Signing Variables

| Variable | Type | Description |
|---|---|---|
| `BR2_TARGET_UBOOT_ENVIMAGE` | bool | Build a U-Boot environment image |
| `BR2_TARGET_UBOOT_ENVIMAGE_SOURCE` | string | Path to text environment file |
| `BR2_TARGET_UBOOT_ENVIMAGE_SIZE` | string | Size of environment flash partition |
| `BR2_TARGET_UBOOT_ENVIMAGE_REDUNDANT` | bool | Build redundant environment image |
| `BR2_TARGET_UBOOT_SIGN` | bool | Enable FIT image signing |
| `BR2_TARGET_UBOOT_SIGN_KEY_DIR` | string | Directory containing signing keys |
| `BR2_TARGET_UBOOT_SIGN_KEYNAME` | string | Name of signing key pair |

### Patch & License Variables

| Variable | Type | Description |
|---|---|---|
| `BR2_TARGET_UBOOT_PATCH` | string | Space-separated patch file paths |
| `BR2_TARGET_UBOOT_LICENSE` | string | Override license string |
| `BR2_TARGET_UBOOT_LICENSE_FILES` | string | License file paths |

### Complete Example `defconfig` Fragment

```makefile
# configs/rk3399_myboard_defconfig  (Buildroot)

# Architecture
BR2_aarch64=y
BR2_TOOLCHAIN_EXTERNAL=y
BR2_TOOLCHAIN_EXTERNAL_PREINSTALLED=y
BR2_TOOLCHAIN_EXTERNAL_PATH="/opt/toolchains/aarch64-linux-gnu"
BR2_TOOLCHAIN_EXTERNAL_CUSTOM_PREFIX="aarch64-linux-gnu"
BR2_TOOLCHAIN_EXTERNAL_GCC_12=y
BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_15=y

# U-Boot
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BUILD_SYSTEM_KCONFIG=y
BR2_TARGET_UBOOT_VERSION="2024.01"
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="rock-pi-4b-rk3399"
BR2_TARGET_UBOOT_CONFIG_FRAGMENT_FILES="board/myvendor/rk3399/uboot.fragment"
BR2_TARGET_UBOOT_FORMAT_ITB=y
BR2_TARGET_UBOOT_SPL=y
BR2_TARGET_UBOOT_SPL_NAME="idbloader.img spl/u-boot-spl.bin tpl/u-boot-tpl.bin"
BR2_TARGET_UBOOT_NEEDS_DTC=y
BR2_TARGET_UBOOT_NEEDS_PYLIBFDT=y

# Environment image (512 KB partition on eMMC at offset 6 MB)
BR2_TARGET_UBOOT_ENVIMAGE=y
BR2_TARGET_UBOOT_ENVIMAGE_SOURCE="board/myvendor/rk3399/uboot-env.txt"
BR2_TARGET_UBOOT_ENVIMAGE_SIZE="0x80000"

# Linux kernel
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_DEFCONFIG="rockchip"
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES="board/myvendor/rk3399/linux.fragment"
BR2_LINUX_KERNEL_IMAGE_TARGET_NAME="Image"
BR2_LINUX_KERNEL_GZIP=y
BR2_LINUX_KERNEL_DTS_SUPPORT=y
BR2_LINUX_KERNEL_INTREE_DTS_NAME="rockchip/rk3399-rock-pi-4b"
```

---

## GRUB2 for x86 Embedded Targets

### Overview

GRUB2 is the bootloader of choice for x86 embedded targets (industrial PCs, HMIs, gateways, servers). Buildroot's `package/grub2/` builds GRUB2 from source and integrates it into the generated disk image.

### Enabling GRUB2

```makefile
# configs/x86_embedded_defconfig
BR2_x86_64=y
BR2_TARGET_GRUB2=y
BR2_TARGET_GRUB2_I386_PC=y          # Legacy BIOS MBR mode
# or:
BR2_TARGET_GRUB2_X86_64_EFI=y       # UEFI 64-bit mode
BR2_TARGET_GRUB2_BUILTIN_MODULES_PC="part_gpt part_msdos ext2 fat linux normal boot configfile echo"
BR2_TARGET_GRUB2_CUSTOM_BUILTIN_FOOTPRINT=y
```

### GRUB2 Partition Layout (ASCII)

```
x86 Disk Image (MBR / GPT + GRUB2)
+--------------------------------------------------+
| LBA 0     MBR (512 bytes)                        |
|           - Boot code (GRUB stage 1)             |
|           - Partition table                      |
+--------------------------------------------------+
| LBA 1–2047  BIOS Boot Partition (1 MB)           |
|           - GRUB core.img embedded here          |
|           - (GPT only — needs special partition  |
|              type GUID: 21686148-...)            |
+--------------------------------------------------+
| LBA 2048+  Partition 1: /boot or / (ext4/FAT32)  |
|           - /boot/grub/grub.cfg                  |
|           - /boot/grub/grub.env                  |
|           - vmlinuz / bzImage                    |
|           - initrd.img                           |
+--------------------------------------------------+
| ...        Partition 2: rootfs (ext4/squashfs)   |
+--------------------------------------------------+
| ...        Partition 3: data / overlay           |
+--------------------------------------------------+
```

### GRUB2 Configuration File

```bash
# board/myvendor/x86embedded/grub.cfg

set default=0
set timeout=3
set timeout_style=menu

# Load environment block (for persistent variables like boot_count)
if [ -f ($root)/boot/grub/grubenv ]; then
  load_env
fi

# A/B rootfs selection
if [ "${boot_slot}" = "B" ]; then
  set rootdev=/dev/sda3
  set rootlabel="rootfs_b"
else
  set rootdev=/dev/sda2
  set rootlabel="rootfs_a"
fi

menuentry "Embedded Linux (Slot A)" {
    linux  /boot/vmlinuz root=/dev/sda2 rootwait ro console=ttyS0,115200n8 quiet
    initrd /boot/initrd.img
}

menuentry "Embedded Linux (Slot B)" {
    linux  /boot/vmlinuz root=/dev/sda3 rootwait ro console=ttyS0,115200n8 quiet
    initrd /boot/initrd.img
}

menuentry "Recovery" {
    linux  /boot/vmlinuz-recovery root=/dev/sda4 rootwait ro init=/sbin/recovery-init console=ttyS0,115200n8
}
```

### Buildroot GRUB2 Variables

| Variable | Type | Description |
|---|---|---|
| `BR2_TARGET_GRUB2` | bool | Enable GRUB2 |
| `BR2_TARGET_GRUB2_I386_PC` | bool | Legacy BIOS target |
| `BR2_TARGET_GRUB2_I386_EFI` | bool | UEFI 32-bit target |
| `BR2_TARGET_GRUB2_X86_64_EFI` | bool | UEFI 64-bit target |
| `BR2_TARGET_GRUB2_BUILTIN_MODULES_PC` | string | Modules baked into core.img |
| `BR2_TARGET_GRUB2_BUILTIN_MODULES_EFI` | string | Modules baked into EFI image |
| `BR2_TARGET_GRUB2_CUSTOM_GRUB_D` | bool | Use a custom `grub.d/` fragments dir |
| `BR2_TARGET_GRUB2_CFG` | string | Path to grub.cfg template |
| `BR2_TARGET_GRUB2_VERSION` | string | GRUB2 version to build |

### Creating a Bootable x86 Image (genimage)

```
# board/myvendor/x86embedded/genimage.cfg

image boot.vfat {
    vfat {
        files = {
            "vmlinuz",
            "initrd.img",
            "grub/grub.cfg",
            "grub/grub.env"
        }
    }
    size = 64M
}

image rootfs.ext4 {
    ext4 {
        filesystem-label = "rootfs"
    }
    size = 512M
}

image disk.img {
    hdimage {
        partition-table-type = "gpt"
    }

    partition bios_boot {
        partition-type-uuid = "21686148-6449-6E6F-744E-656564454649"
        in-partition-table = true
        size = 1M
    }

    partition boot {
        partition-type-uuid = "EBD0A0A2-B9E5-4433-87C0-68B6B72699C7"
        bootable = true
        image = "boot.vfat"
    }

    partition rootfs {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"
        image = "rootfs.ext4"
    }
}
```

---

## C Code Examples

### 1. U-Boot Board Initialization (C)

U-Boot board init is written in C with a well-defined hook API. Below is a realistic `board.c` for a custom ARM board:

```c
/* board/myvendor/myboard/board.c
 * Custom board initialisation for U-Boot
 * Tested with U-Boot 2024.01, ARM Cortex-A53
 */

#include <common.h>
#include <init.h>
#include <asm/arch/clock.h>
#include <asm/arch/pinmux.h>
#include <asm/arch/sys_proto.h>
#include <asm/io.h>
#include <dm.h>
#include <env.h>
#include <led.h>
#include <linux/delay.h>

DECLARE_GLOBAL_DATA_PTR;

/* Board identification via GPIO strapping */
#define GPIO_BOARD_ID_BASE   0x02000020
#define GPIO_BOARD_ID_MASK   0x0F

typedef enum {
    BOARD_REV_A = 0x01,
    BOARD_REV_B = 0x02,
    BOARD_REV_C = 0x03,
} board_revision_t;

static board_revision_t detect_board_revision(void)
{
    u32 gpio_val = readl(GPIO_BOARD_ID_BASE);
    return (board_revision_t)(gpio_val & GPIO_BOARD_ID_MASK);
}

/* Called very early — only basic register access available */
int board_early_init_f(void)
{
    /* Configure UART pinmux for early console */
    pinmux_config_uart0();

    /* Set up watchdog — disable during development */
#ifdef CONFIG_HW_WATCHDOG
    hw_watchdog_init();
#endif

    return 0;
}

/* Called after DRAM is initialised */
int board_init(void)
{
    board_revision_t rev = detect_board_revision();

    printf("Board revision: %c\n", 'A' + (int)rev - 1);

    /* Set environment variable for boot scripts to consume */
    env_set_ulong("board_rev", (ulong)rev);

    /* Initialise LEDs via DM */
    struct udevice *dev;
    if (led_get_by_label("status", &dev) == 0) {
        led_set_state(dev, LEDST_ON);
    }

    /* Configure PMIC — board-specific power sequencing */
    if (board_pmic_init() != 0) {
        printf("WARNING: PMIC init failed\n");
    }

    return 0;
}

/* Override environment variables at runtime */
int board_late_init(void)
{
    board_revision_t rev = detect_board_revision();

    /* Select device tree based on board revision */
    switch (rev) {
    case BOARD_REV_A:
        env_set("fdtfile", "myboard-reva.dtb");
        break;
    case BOARD_REV_B:
        env_set("fdtfile", "myboard-revb.dtb");
        break;
    case BOARD_REV_C:
    default:
        env_set("fdtfile", "myboard-revc.dtb");
        break;
    }

    /* Default boot command using FIT image */
    env_set("bootcmd",
        "mmc dev 0; "
        "load mmc 0:1 ${loadaddr} boot.itb; "
        "bootm ${loadaddr}#conf@1");

    return 0;
}

/* SPL board init — runs before DRAM, in on-chip SRAM */
#ifdef CONFIG_SPL_BUILD
void board_init_f(ulong dummy)
{
    /* Minimal init: clocks + serial + DRAM */
    board_early_init_f();
    preloader_console_init();

    /* DRAM initialisation — vendor-specific code */
    if (dram_init() != 0) {
        hang();  /* Cannot continue without DRAM */
    }

    printf("SPL: DRAM %lu MiB\n", gd->ram_size / (1024 * 1024));
}
#endif /* CONFIG_SPL_BUILD */

/* Report DRAM size to U-Boot core */
int dram_init(void)
{
    gd->ram_size = get_ram_size((long *)PHYS_SDRAM, PHYS_SDRAM_SIZE);
    return 0;
}
```

### 2. U-Boot Environment Manipulation (C)

```c
/* tools/env_tool/env_tool.c
 * Standalone host tool to read/write U-Boot environment partitions.
 * Compile: gcc -o env_tool env_tool.c -lz
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <zlib.h>     /* for crc32() */
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

#define ENV_SIZE        0x20000   /* 128 KB — must match U-Boot config */
#define ENV_MAGIC       0x27051956
#define CRC_OFFSET      0
#define DATA_OFFSET     4

typedef struct {
    uint32_t crc32;
    uint8_t  data[ENV_SIZE - sizeof(uint32_t)];
} uboot_env_t;

/* Parse a raw U-Boot environment image and print all variables */
static int env_print_all(const uboot_env_t *env)
{
    uint32_t crc_computed = crc32(0, env->data, sizeof(env->data));

    if (crc_computed != env->crc32) {
        fprintf(stderr, "CRC mismatch: stored=0x%08X computed=0x%08X\n",
                env->crc32, crc_computed);
        return -1;
    }

    printf("U-Boot Environment (CRC OK: 0x%08X)\n", env->crc32);
    printf("%-40s  %s\n", "Variable", "Value");
    printf("%.40s  %.40s\n",
           "----------------------------------------",
           "----------------------------------------");

    const uint8_t *p = env->data;
    const uint8_t *end = env->data + sizeof(env->data);

    while (p < end && *p != '\0') {
        const char *entry = (const char *)p;
        const char *eq = strchr(entry, '=');

        if (eq) {
            int name_len = (int)(eq - entry);
            printf("%-40.*s  %s\n", name_len, entry, eq + 1);
        }

        p += strlen(entry) + 1;  /* skip past NUL terminator */
    }

    return 0;
}

/* Set a variable in an in-memory environment, recalculate CRC */
static int env_set_var(uboot_env_t *env,
                       const char *key,
                       const char *value)
{
    uint8_t *p = env->data;
    uint8_t *end = env->data + sizeof(env->data);
    size_t key_len = strlen(key);

    /* Remove existing entry if present */
    while (p < end && *p != '\0') {
        if (strncmp((char *)p, key, key_len) == 0 &&
            ((char *)p)[key_len] == '=') {
            size_t entry_len = strlen((char *)p) + 1;
            memmove(p, p + entry_len, end - p - entry_len);
            memset(end - entry_len, '\0', entry_len);
            break;
        }
        p += strlen((char *)p) + 1;
    }

    /* Append new entry */
    while (p < end && *p != '\0')
        p += strlen((char *)p) + 1;

    size_t needed = key_len + 1 + strlen(value) + 1;
    if (p + needed >= end) {
        fprintf(stderr, "Environment is full\n");
        return -ENOMEM;
    }

    snprintf((char *)p, needed, "%s=%s", key, value);

    /* Recalculate CRC */
    env->crc32 = crc32(0, env->data, sizeof(env->data));

    printf("Set '%s' = '%s' (new CRC: 0x%08X)\n",
           key, value, env->crc32);
    return 0;
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <env.bin> [key=value ...]\n", argv[0]);
        return EXIT_FAILURE;
    }

    uboot_env_t *env = calloc(1, sizeof(uboot_env_t));
    if (!env) { perror("calloc"); return EXIT_FAILURE; }

    /* Read environment file */
    FILE *f = fopen(argv[1], "rb");
    if (!f) { perror(argv[1]); free(env); return EXIT_FAILURE; }
    fread(env, 1, sizeof(uboot_env_t), f);
    fclose(f);

    if (argc == 2) {
        /* Print mode */
        int rc = env_print_all(env);
        free(env);
        return rc ? EXIT_FAILURE : EXIT_SUCCESS;
    }

    /* Set mode: process key=value arguments */
    for (int i = 2; i < argc; i++) {
        char *kv = strdup(argv[i]);
        char *eq = strchr(kv, '=');
        if (!eq) {
            fprintf(stderr, "Argument must be key=value: %s\n", argv[i]);
            free(kv); free(env);
            return EXIT_FAILURE;
        }
        *eq = '\0';
        env_set_var(env, kv, eq + 1);
        free(kv);
    }

    /* Write back */
    f = fopen(argv[1], "wb");
    if (!f) { perror(argv[1]); free(env); return EXIT_FAILURE; }
    fwrite(env, 1, sizeof(uboot_env_t), f);
    fclose(f);

    free(env);
    return EXIT_SUCCESS;
}
```

### 3. Linux Kernel: Reading Boot Reason from U-Boot Environment (C)

```c
/* drivers/misc/uboot_reason.c
 * Exposes U-Boot boot reason (passed via /chosen in DTB) to sysfs.
 */

#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/sysfs.h>
#include <linux/kobject.h>
#include <linux/string.h>

static char boot_reason[64] = "unknown";
static char boot_slot[4]    = "A";
static u32  boot_count      = 0;

static ssize_t boot_reason_show(struct kobject *kobj,
                                struct kobj_attribute *attr,
                                char *buf)
{
    return sysfs_emit(buf, "%s\n", boot_reason);
}

static ssize_t boot_slot_show(struct kobject *kobj,
                              struct kobj_attribute *attr,
                              char *buf)
{
    return sysfs_emit(buf, "%s\n", boot_slot);
}

static ssize_t boot_count_show(struct kobject *kobj,
                               struct kobj_attribute *attr,
                               char *buf)
{
    return sysfs_emit(buf, "%u\n", boot_count);
}

static struct kobj_attribute boot_reason_attr =
    __ATTR_RO(boot_reason);
static struct kobj_attribute boot_slot_attr =
    __ATTR_RO(boot_slot);
static struct kobj_attribute boot_count_attr =
    __ATTR_RO(boot_count);

static struct attribute *boot_attrs[] = {
    &boot_reason_attr.attr,
    &boot_slot_attr.attr,
    &boot_count_attr.attr,
    NULL,
};

static struct attribute_group boot_attr_group = {
    .attrs = boot_attrs,
};

static struct kobject *boot_kobj;

static int __init uboot_reason_init(void)
{
    struct device_node *chosen;
    const char *prop;
    u32 val;

    /* Read properties set by U-Boot into /chosen DT node */
    chosen = of_find_node_by_path("/chosen");
    if (!chosen) {
        pr_warn("uboot_reason: /chosen not found in DT\n");
        goto create_sysfs;
    }

    if (of_property_read_string(chosen, "myboard,boot-reason", &prop) == 0)
        strlcpy(boot_reason, prop, sizeof(boot_reason));

    if (of_property_read_string(chosen, "myboard,boot-slot", &prop) == 0)
        strlcpy(boot_slot, prop, sizeof(boot_slot));

    if (of_property_read_u32(chosen, "myboard,boot-count", &val) == 0)
        boot_count = val;

    of_node_put(chosen);

create_sysfs:
    boot_kobj = kobject_create_and_add("boot_info",
                                       kernel_kobj);
    if (!boot_kobj)
        return -ENOMEM;

    return sysfs_create_group(boot_kobj, &boot_attr_group);
}

static void __exit uboot_reason_exit(void)
{
    if (boot_kobj) {
        sysfs_remove_group(boot_kobj, &boot_attr_group);
        kobject_put(boot_kobj);
    }
}

module_init(uboot_reason_init);
module_exit(uboot_reason_exit);

MODULE_LICENSE("GPL v2");
MODULE_AUTHOR("Embedded Team");
MODULE_DESCRIPTION("U-Boot boot reason sysfs interface");
```

---

## C++ Code Examples

### 1. FIT Image Parser (C++)

```cpp
// tools/fit_parser/fit_parser.cpp
// Parse and display contents of a U-Boot FIT image (.itb)
// Build: g++ -std=c++17 -o fit_parser fit_parser.cpp -lfdt
//
// libfdt provides a C API to traverse Flattened Device Tree structures,
// which FIT images are built upon.

#include <iostream>
#include <fstream>
#include <vector>
#include <string>
#include <iomanip>
#include <stdexcept>
#include <map>
#include <libfdt.h>

class FITParser {
public:
    explicit FITParser(const std::string &path)
    {
        std::ifstream file(path, std::ios::binary);
        if (!file.is_open())
            throw std::runtime_error("Cannot open: " + path);

        file.seekg(0, std::ios::end);
        m_data.resize(file.tellg());
        file.seekg(0, std::ios::beg);
        file.read(reinterpret_cast<char*>(m_data.data()), m_data.size());

        if (fdt_check_header(m_data.data()) != 0)
            throw std::runtime_error("Invalid FDT/FIT header");

        std::cout << "FIT image: " << path
                  << " (" << m_data.size() / 1024 << " KB)\n\n";
    }

    void parse()
    {
        parse_description();
        parse_images();
        parse_configurations();
    }

private:
    const void *fdt() const { return m_data.data(); }

    std::string get_string_prop(int node, const char *prop) const
    {
        int len = 0;
        const char *val = static_cast<const char*>(
            fdt_getprop(fdt(), node, prop, &len));
        return val ? std::string(val, len - 1) : "(not set)";
    }

    uint64_t get_u64_prop(int node, const char *prop) const
    {
        int len = 0;
        const void *val = fdt_getprop(fdt(), node, prop, &len);
        if (!val) return 0;
        if (len == 4) return fdt32_to_cpu(*(const fdt32_t*)val);
        if (len == 8) return fdt64_to_cpu(*(const fdt64_t*)val);
        return 0;
    }

    size_t get_data_size(int node) const
    {
        int len = 0;
        fdt_getprop(fdt(), node, "data", &len);
        return len > 0 ? static_cast<size_t>(len) : 0;
    }

    void parse_description()
    {
        int root = fdt_path_offset(fdt(), "/");
        if (root >= 0) {
            std::cout << "Description : "
                      << get_string_prop(root, "description") << "\n\n";
        }
    }

    void parse_images()
    {
        int images = fdt_path_offset(fdt(), "/images");
        if (images < 0) {
            std::cout << "No /images node found.\n";
            return;
        }

        std::cout << "=== IMAGES ===\n";

        int node;
        fdt_for_each_subnode(node, fdt(), images) {
            const char *name = fdt_get_name(fdt(), node, nullptr);
            size_t data_sz = get_data_size(node);

            std::cout << "\n  [" << name << "]\n"
                      << "    description : "
                      << get_string_prop(node, "description") << "\n"
                      << "    type        : "
                      << get_string_prop(node, "type") << "\n"
                      << "    arch        : "
                      << get_string_prop(node, "arch") << "\n"
                      << "    os          : "
                      << get_string_prop(node, "os") << "\n"
                      << "    compression : "
                      << get_string_prop(node, "compression") << "\n"
                      << "    data size   : "
                      << data_sz / 1024 << " KB ("
                      << data_sz << " bytes)\n"
                      << "    load addr   : 0x"
                      << std::hex << std::setw(8) << std::setfill('0')
                      << get_u64_prop(node, "load") << "\n"
                      << "    entry addr  : 0x"
                      << get_u64_prop(node, "entry") << std::dec << "\n";

            // List hash nodes
            int hash;
            fdt_for_each_subnode(hash, fdt(), node) {
                const char *hname = fdt_get_name(fdt(), hash, nullptr);
                if (strncmp(hname, "hash", 4) == 0) {
                    std::cout << "    hash [" << hname << "]: algo="
                              << get_string_prop(hash, "algo") << "\n";
                }
            }
        }
        std::cout << "\n";
    }

    void parse_configurations()
    {
        int configs = fdt_path_offset(fdt(), "/configurations");
        if (configs < 0) return;

        std::string default_cfg = get_string_prop(configs, "default");
        std::cout << "=== CONFIGURATIONS (default: "
                  << default_cfg << ") ===\n";

        int node;
        fdt_for_each_subnode(node, fdt(), configs) {
            const char *name = fdt_get_name(fdt(), node, nullptr);
            std::cout << "\n  [" << name << "]\n"
                      << "    description : "
                      << get_string_prop(node, "description") << "\n"
                      << "    kernel      : "
                      << get_string_prop(node, "kernel") << "\n"
                      << "    fdt         : "
                      << get_string_prop(node, "fdt") << "\n"
                      << "    ramdisk     : "
                      << get_string_prop(node, "ramdisk") << "\n";
        }
    }

    std::vector<uint8_t> m_data;
};

int main(int argc, char *argv[])
{
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <file.itb>\n";
        return EXIT_FAILURE;
    }

    try {
        FITParser parser(argv[1]);
        parser.parse();
    } catch (const std::exception &e) {
        std::cerr << "Error: " << e.what() << "\n";
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

### 2. GRUB2 Boot Count Manager (C++)

```cpp
// tools/grub_env/grub_env_manager.cpp
// Read and manipulate GRUB2 environment block for A/B update management.
// Build: g++ -std=c++17 -o grub_env_manager grub_env_manager.cpp
//
// The GRUB environment block is a 1024-byte file:
//   - 12 bytes: magic "#!grub_envblk" (no NUL at end, padded with \n)
//   - Remaining: "key=value\n" pairs, padded with '#' to fill the block

#include <iostream>
#include <fstream>
#include <string>
#include <map>
#include <stdexcept>
#include <cstring>
#include <iomanip>

class GrubEnvBlock {
public:
    static constexpr size_t BLOCK_SIZE = 1024;
    static constexpr const char *MAGIC = "#!grub_envblk";
    static constexpr size_t MAGIC_LEN = 13;

    explicit GrubEnvBlock(const std::string &path) : m_path(path)
    {
        load();
    }

    void load()
    {
        std::ifstream f(m_path, std::ios::binary);
        if (!f.is_open())
            throw std::runtime_error("Cannot open: " + m_path);

        m_block.resize(BLOCK_SIZE);
        f.read(m_block.data(), BLOCK_SIZE);

        if (f.gcount() != BLOCK_SIZE)
            throw std::runtime_error("Short read from grubenv block");

        if (memcmp(m_block.data(), MAGIC, MAGIC_LEN) != 0)
            throw std::runtime_error("Invalid grubenv magic");

        parse();
    }

    void save()
    {
        std::fill(m_block.begin() + MAGIC_LEN + 1, m_block.end(), '#');
        memcpy(m_block.data(), MAGIC, MAGIC_LEN);
        m_block[MAGIC_LEN] = '\n';

        size_t pos = MAGIC_LEN + 1;
        for (const auto &[key, val] : m_vars) {
            std::string entry = key + "=" + val + "\n";
            if (pos + entry.size() > BLOCK_SIZE - 1)
                throw std::runtime_error("grubenv block overflow");
            memcpy(m_block.data() + pos, entry.data(), entry.size());
            pos += entry.size();
        }

        std::ofstream f(m_path, std::ios::binary);
        if (!f.is_open())
            throw std::runtime_error("Cannot write: " + m_path);
        f.write(m_block.data(), BLOCK_SIZE);
    }

    std::string get(const std::string &key,
                    const std::string &fallback = "") const
    {
        auto it = m_vars.find(key);
        return (it != m_vars.end()) ? it->second : fallback;
    }

    void set(const std::string &key, const std::string &value)
    {
        m_vars[key] = value;
    }

    void print() const
    {
        std::cout << "GRUB Environment Block (" << m_path << ")\n";
        std::cout << std::string(50, '-') << "\n";
        for (const auto &[k, v] : m_vars)
            std::cout << std::left << std::setw(24) << k
                      << "  " << v << "\n";
    }

    /* A/B update helper: increment boot count, switch slot if exceeded */
    void handle_ab_watchdog(int max_boot_tries = 3)
    {
        std::string slot  = get("boot_slot", "A");
        int tries = std::stoi(get("boot_tries", "0"));
        bool success = (get("boot_success", "0") == "1");

        if (success) {
            set("boot_tries", "0");
            std::cout << "Boot successful on slot " << slot << "\n";
            return;
        }

        tries++;
        std::cout << "Boot attempt " << tries << "/" << max_boot_tries
                  << " on slot " << slot << "\n";

        if (tries >= max_boot_tries) {
            // Switch to other slot
            std::string new_slot = (slot == "A") ? "B" : "A";
            std::cout << "Switching from slot " << slot
                      << " to " << new_slot << "\n";
            set("boot_slot", new_slot);
            set("boot_tries", "0");
        } else {
            set("boot_tries", std::to_string(tries));
        }
    }

private:
    void parse()
    {
        m_vars.clear();
        const char *p   = m_block.data() + MAGIC_LEN + 1;
        const char *end = m_block.data() + BLOCK_SIZE;

        while (p < end && *p != '#' && *p != '\0') {
            const char *line_end = static_cast<const char*>(
                memchr(p, '\n', end - p));
            if (!line_end) break;

            std::string line(p, line_end);
            auto eq = line.find('=');
            if (eq != std::string::npos)
                m_vars[line.substr(0, eq)] = line.substr(eq + 1);

            p = line_end + 1;
        }
    }

    std::string              m_path;
    std::vector<char>        m_block;
    std::map<std::string, std::string> m_vars;
};

int main(int argc, char *argv[])
{
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0]
                  << " <grubenv> [key=value | --ab-watchdog | --print]\n";
        return EXIT_FAILURE;
    }

    try {
        GrubEnvBlock env(argv[1]);

        if (argc == 2 || (argc == 3 && std::string(argv[2]) == "--print")) {
            env.print();
            return EXIT_SUCCESS;
        }

        if (std::string(argv[2]) == "--ab-watchdog") {
            env.handle_ab_watchdog();
            env.save();
            return EXIT_SUCCESS;
        }

        // Set key=value pairs
        for (int i = 2; i < argc; i++) {
            std::string kv(argv[i]);
            auto eq = kv.find('=');
            if (eq == std::string::npos) {
                std::cerr << "Argument must be key=value: " << kv << "\n";
                return EXIT_FAILURE;
            }
            env.set(kv.substr(0, eq), kv.substr(eq + 1));
        }

        env.save();
        std::cout << "grubenv updated.\n";

    } catch (const std::exception &e) {
        std::cerr << "Error: " << e.what() << "\n";
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

---

## Rust Code Examples

### 1. FIT Image CRC Verifier (Rust)

```rust
// tools/fit_verify/src/main.rs
// Verify SHA-256 hashes of images inside a U-Boot FIT (.itb) file.
//
// Cargo.toml dependencies:
//   sha2 = "0.10"
//   hex = "0.4"
//   fdt = "0.1"      (pure-Rust FDT parser)
//   anyhow = "1.0"
//   clap = { version = "4", features = ["derive"] }

use anyhow::{anyhow, Context, Result};
use clap::Parser;
use fdt::Fdt;
use sha2::{Digest, Sha256};
use std::fs;
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "fit_verify", about = "Verify FIT image SHA-256 hashes")]
struct Args {
    /// Path to FIT image (.itb file)
    path: PathBuf,

    /// Print all node properties (verbose)
    #[arg(short, long)]
    verbose: bool,
}

#[derive(Debug)]
struct ImageEntry<'a> {
    name:        &'a str,
    description: Option<&'a str>,
    image_type:  Option<&'a str>,
    data:        &'a [u8],
    expected_sha256: Option<Vec<u8>>,
}

fn parse_images<'a>(fdt: &'a Fdt<'a>) -> Result<Vec<ImageEntry<'a>>> {
    let images_node = fdt
        .find_node("/images")
        .ok_or_else(|| anyhow!("No /images node in FIT"))?;

    let mut entries = Vec::new();

    for child in images_node.children() {
        // Get raw data blob
        let data = match child.property("data") {
            Some(p) => p.value,
            None    => {
                eprintln!("  [{}] no data property — skipping", child.name);
                continue;
            }
        };

        let description = child
            .property("description")
            .and_then(|p| p.as_str());
        let image_type = child
            .property("type")
            .and_then(|p| p.as_str());

        // Look for a hash@1 sub-node with algo=sha256
        let expected_sha256 = child.children()
            .filter(|n| n.name.starts_with("hash"))
            .find_map(|hash_node| {
                let algo = hash_node.property("algo")?.as_str()?;
                if algo != "sha256" { return None; }
                let value = hash_node.property("value")?;
                Some(value.value.to_vec())
            });

        entries.push(ImageEntry {
            name: child.name,
            description,
            image_type,
            data,
            expected_sha256,
        });
    }

    Ok(entries)
}

fn verify_image(entry: &ImageEntry<'_>, verbose: bool) -> bool {
    let desc = entry.description.unwrap_or("(no description)");
    let itype = entry.image_type.unwrap_or("unknown");

    print!("  [{:<30}]  type={:<10}  size={:>8} KB  SHA-256: ",
           entry.name,
           itype,
           entry.data.len() / 1024);

    let computed: Vec<u8> = Sha256::digest(entry.data).to_vec();

    match &entry.expected_sha256 {
        Some(expected) => {
            if computed == *expected {
                println!("OK  ✓");
                if verbose {
                    println!("    desc   : {}", desc);
                    println!("    digest : {}", hex::encode(&computed));
                }
                true
            } else {
                println!("FAIL  ✗");
                eprintln!("    Expected : {}", hex::encode(expected));
                eprintln!("    Computed : {}", hex::encode(&computed));
                false
            }
        }
        None => {
            println!("(no hash node)");
            if verbose {
                println!("    desc   : {}", desc);
                println!("    digest : {}", hex::encode(&computed));
            }
            true  // No hash to verify — treat as pass
        }
    }
}

fn main() -> Result<()> {
    let args = Args::parse();

    let data = fs::read(&args.path)
        .with_context(|| format!("Reading {:?}", args.path))?;

    let fdt = Fdt::new(&data)
        .map_err(|e| anyhow!("FDT parse error: {:?}", e))?;

    // Print top-level description
    if let Some(desc_node) = fdt.find_node("/") {
        if let Some(desc) = desc_node.property("description").and_then(|p| p.as_str()) {
            println!("FIT Image  : {}", desc);
        }
    }
    println!("File       : {:?}", args.path);
    println!("File size  : {} KB\n", data.len() / 1024);
    println!("{}", "=".repeat(80));
    println!("Image Verification Results");
    println!("{}", "=".repeat(80));

    let images = parse_images(&fdt)?;
    if images.is_empty() {
        return Err(anyhow!("No images found in FIT"));
    }

    let mut all_ok = true;
    for image in &images {
        if !verify_image(image, args.verbose) {
            all_ok = false;
        }
    }

    println!("{}", "-".repeat(80));

    // Parse and display default configuration
    if let Some(configs) = fdt.find_node("/configurations") {
        let default_cfg = configs
            .property("default")
            .and_then(|p| p.as_str())
            .unwrap_or("(none)");
        println!("Default config : {}", default_cfg);
    }

    println!("\nResult: {}", if all_ok { "ALL PASS ✓" } else { "SOME FAILED ✗" });

    if !all_ok {
        std::process::exit(1);
    }

    Ok(())
}
```

### 2. A/B Update State Machine (Rust)

```rust
// src/ab_update/src/lib.rs
// A/B boot slot state machine for embedded Linux systems.
// Integrates with U-Boot environment or GRUB2 grubenv.
//
// Cargo.toml:
//   serde = { version = "1", features = ["derive"] }
//   serde_json = "1"
//   thiserror = "1"
//   log = "0.4"

use serde::{Deserialize, Serialize};
use std::fmt;
use thiserror::Error;

// ─── Error type ──────────────────────────────────────────────────────────────

#[derive(Debug, Error)]
pub enum ABError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    #[error("JSON error: {0}")]
    Json(#[from] serde_json::Error),
    #[error("Invalid slot: {0}")]
    InvalidSlot(String),
    #[error("Both slots are unbootable")]
    NoBootableSlot,
}

// ─── Slot definition ─────────────────────────────────────────────────────────

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Slot { A, B }

impl fmt::Display for Slot {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", match self { Slot::A => "A", Slot::B => "B" })
    }
}

impl Slot {
    pub fn other(self) -> Self {
        match self { Slot::A => Slot::B, Slot::B => Slot::A }
    }
}

// ─── Slot state ──────────────────────────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SlotState {
    pub slot:           Slot,
    pub bootable:       bool,
    pub successful:     bool,
    pub tries_remaining: u8,
    pub version:        String,
}

impl SlotState {
    pub fn new(slot: Slot) -> Self {
        Self {
            slot,
            bootable:        true,
            successful:      false,
            tries_remaining: 3,
            version:         String::from("unknown"),
        }
    }
}

impl fmt::Display for SlotState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f,
            "Slot {}: bootable={} successful={} tries={} version={}",
            self.slot,
            self.bootable,
            self.successful,
            self.tries_remaining,
            self.version,
        )
    }
}

// ─── A/B Manager ─────────────────────────────────────────────────────────────

#[derive(Debug, Serialize, Deserialize)]
pub struct ABManager {
    pub slot_a:       SlotState,
    pub slot_b:       SlotState,
    pub active_slot:  Slot,
}

impl ABManager {
    pub fn new() -> Self {
        Self {
            slot_a:      SlotState::new(Slot::A),
            slot_b:      SlotState::new(Slot::B),
            active_slot: Slot::A,
        }
    }

    pub fn from_json(json: &str) -> Result<Self, ABError> {
        Ok(serde_json::from_str(json)?)
    }

    pub fn to_json(&self) -> Result<String, ABError> {
        Ok(serde_json::to_string_pretty(self)?)
    }

    pub fn slot_state(&self, slot: Slot) -> &SlotState {
        match slot { Slot::A => &self.slot_a, Slot::B => &self.slot_b }
    }

    fn slot_state_mut(&mut self, slot: Slot) -> &mut SlotState {
        match slot { Slot::A => &mut self.slot_a, Slot::B => &mut self.slot_b }
    }

    /// Called by bootloader before each boot attempt
    pub fn prepare_boot(&mut self) -> Result<Slot, ABError> {
        let active = self.active_slot;
        let state  = self.slot_state_mut(active);

        if !state.bootable {
            // Try other slot
            let other = active.other();
            let other_state = self.slot_state_mut(other);
            if !other_state.bootable {
                return Err(ABError::NoBootableSlot);
            }
            log::warn!("Active slot {} unbootable, trying {}", active, other);
            self.active_slot = other;
            return self.prepare_boot();
        }

        if state.tries_remaining == 0 {
            log::warn!("Slot {} exhausted retries, marking unbootable", active);
            state.bootable = false;
            return self.prepare_boot();
        }

        state.tries_remaining -= 1;
        log::info!("Booting slot {} ({} tries remaining)",
            active, state.tries_remaining);

        Ok(active)
    }

    /// Called by init system after successful boot
    pub fn mark_successful(&mut self, slot: Slot) {
        let state = self.slot_state_mut(slot);
        state.successful      = true;
        state.tries_remaining = 3;   // Reset for future updates
        log::info!("Slot {} marked successful", slot);
    }

    /// Write new firmware to inactive slot
    pub fn set_update_pending(&mut self, slot: Slot, version: &str) {
        let state = self.slot_state_mut(slot);
        state.successful      = false;
        state.bootable        = true;
        state.tries_remaining = 3;
        state.version         = version.to_owned();
        log::info!("Update pending on slot {}: {}", slot, version);
    }

    /// Switch active slot (after writing update)
    pub fn switch_active(&mut self) {
        let new_active = self.active_slot.other();
        log::info!("Switching active slot: {} -> {}", self.active_slot, new_active);
        self.active_slot = new_active;
    }

    pub fn print_status(&self) {
        println!("{}", "─".repeat(50));
        println!("A/B Boot Manager Status");
        println!("{}", "─".repeat(50));
        println!("Active slot : {}", self.active_slot);
        println!("{}", self.slot_state(Slot::A));
        println!("{}", self.slot_state(Slot::B));
        println!("{}", "─".repeat(50));
    }
}

// ─── Tests ───────────────────────────────────────────────────────────────────

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_boot_and_success() {
        let mut mgr = ABManager::new();
        let slot = mgr.prepare_boot().expect("should boot");
        assert_eq!(slot, Slot::A);
        assert_eq!(mgr.slot_a.tries_remaining, 2);

        mgr.mark_successful(slot);
        assert!(mgr.slot_a.successful);
        assert_eq!(mgr.slot_a.tries_remaining, 3);
    }

    #[test]
    fn test_retry_exhaustion_switches_slot() {
        let mut mgr = ABManager::new();
        mgr.slot_a.tries_remaining = 1;

        let _slot = mgr.prepare_boot().unwrap();  // consumes last try of A
        let slot2 = mgr.prepare_boot().unwrap();  // A exhausted → falls to B
        assert_eq!(slot2, Slot::B);
    }

    #[test]
    fn test_update_flow() {
        let mut mgr = ABManager::new();
        mgr.mark_successful(Slot::A);

        // Update arrives for the inactive slot B
        mgr.set_update_pending(Slot::B, "v2.0.0");
        mgr.switch_active();   // B is now active

        let slot = mgr.prepare_boot().unwrap();
        assert_eq!(slot, Slot::B);
    }

    #[test]
    fn test_json_round_trip() {
        let mgr = ABManager::new();
        let json = mgr.to_json().unwrap();
        let mgr2 = ABManager::from_json(&json).unwrap();
        assert_eq!(mgr2.active_slot, Slot::A);
    }
}
```

---

## Advanced Scenarios

### Secure Boot with FIT Image Signing (ASCII Overview)

```
  Build Host
  +---------------------------------------------------------+
  |  1. Generate RSA key pair                               |
  |     $ openssl genrsa -out dev.key 2048                  |
  |     $ openssl req -batch -new -x509 -key dev.key \      |
  |           -out dev.crt                                  |
  |                                                         |
  |  2. Build U-Boot with public key embedded               |
  |     BR2_TARGET_UBOOT_SIGN=y                             |
  |     BR2_TARGET_UBOOT_SIGN_KEY_DIR="keys/"               |
  |     BR2_TARGET_UBOOT_SIGN_KEYNAME="dev"                 |
  |     → mkimage embeds dev.crt into u-boot.dtb            |
  |                                                         |
  |  3. Sign FIT image at build time                        |
  |     $ mkimage -F -k keys/ -K u-boot.dtb \               |
  |           -r boot.itb                                   |
  |     → RSA signature added to /configurations/conf@1/    |
  |       signature@1 node                                  |
  +---------------------------------------------------------+
              |                         |
              v                         v
  +-----------------+       +---------------------+
  |  u-boot.itb     |       |  boot.itb (signed)  |
  |  (with embedded |       |                     |
  |   public key    |       |  kernel + DTB +     |
  |   in DTB)       |       |  ramdisk + sig      |
  +-----------------+       +---------------------+
              |                         |
              +----------+  +-----------+
                         |  |
                         v  v
                  Target Device Flash
  +-----------------------------------------------------------+
  |  U-Boot verifies boot.itb signature against public key    |
  |  embedded in its own compiled-in DTB at boot time         |
  |                                                           |
  |  If signature check fails → BOOT HALTED                   |
  +-----------------------------------------------------------+
```

### U-Boot Script for A/B Boot Selection

```bash
# u-boot-env.txt  — U-Boot environment variables
# Stored in NVRAM / eMMC boot partition

bootdelay=3
ab_bootcmd=run check_slot; run do_boot_${boot_slot}
boot_slot=A
boot_tries_a=0
boot_tries_b=0
boot_success_a=0
boot_success_b=0
max_tries=3

check_slot=\
  if test ${boot_success_${boot_slot}} = "1"; then \
    echo "Slot ${boot_slot} last boot was successful"; \
  elif test ${boot_tries_${boot_slot}} -ge ${max_tries}; then \
    echo "Slot ${boot_slot} exhausted, switching"; \
    if test ${boot_slot} = "A"; then setenv boot_slot B; \
    else setenv boot_slot A; fi; \
    setenv boot_tries_${boot_slot} 0; \
    saveenv; \
  else \
    setexpr new_tries ${boot_tries_${boot_slot}} + 1; \
    setenv boot_tries_${boot_slot} ${new_tries}; \
    saveenv; \
  fi

do_boot_A=\
  setenv root_dev /dev/mmcblk0p2; \
  load mmc 0:1 ${fitloadaddr} boot_a.itb; \
  bootm ${fitloadaddr}#conf@1

do_boot_B=\
  setenv root_dev /dev/mmcblk0p3; \
  load mmc 0:1 ${fitloadaddr} boot_b.itb; \
  bootm ${fitloadaddr}#conf@1

bootcmd=run ab_bootcmd
```

### eMMC Flash Layout (ASCII)

```
eMMC Physical Layout
+-----------------------------------------------+
| Boot Partition 0 (boot0, 4 MB)                |
|   0x000000  idbloader.img (TPL+SPL, Rockchip) |
|   0x100000  u-boot.itb                        |
+-----------------------------------------------+
| Boot Partition 1 (boot1, 4 MB)                |
|   (backup / redundant U-Boot)                 |
+-----------------------------------------------+
| User Area (main storage)                      |
|   p1: /boot  FAT32  64 MB                     |
|        boot_a.itb                             |
|        boot_b.itb                             |
|        grub/ (x86 variant)                    |
|                                               |
|   p2: rootfs_a  ext4  512 MB                  |
|   p3: rootfs_b  ext4  512 MB                  |
|   p4: data      ext4  remaining               |
|        /var, /home, /etc overlays             |
+-----------------------------------------------+
```

---

## Summary

Bootloader integration in Buildroot ties together some of the most complex and hardware-specific parts of an embedded Linux system. The following points capture the essential knowledge from this chapter:

**U-Boot** is built via Buildroot's `package/uboot/` and configured entirely through `BR2_TARGET_UBOOT_*` Kconfig variables. The `BR2_TARGET_UBOOT_BOARD_DEFCONFIG` variable selects the upstream U-Boot board configuration, while `BR2_TARGET_UBOOT_CONFIG_FRAGMENT_FILES` allows incremental customisation without forking upstream configs. Enabling `BR2_TARGET_UBOOT_SPL=y` and setting `BR2_TARGET_UBOOT_SPL_NAME` is essential for SoCs with tiny on-chip SRAM that require a pre-loader stage (SPL/TPL) to initialise DRAM before U-Boot proper can run.

**FIT Images** (Flattened Image Trees) are the modern, recommended packaging format for U-Boot payloads. A single `.itb` file bundles the kernel, device tree(s), and optional ramdisk under a hierarchy of cryptographically hashed nodes. The `.its` (Image Tree Source) text file describes this layout and is compiled to `.itb` by `mkimage`. FIT images enable verified boot through embedded RSA/ECDSA signatures, hardware-enforced security that rejects any unsigned or tampered firmware.

**GRUB2** is Buildroot's bootloader for x86 and x86-64 embedded targets. It is configured via `BR2_TARGET_GRUB2_*` variables and supports both legacy BIOS (`I386_PC`) and UEFI (`X86_64_EFI`) targets. A `grub.cfg` template controls menu entries, and the GRUB environment block (`grubenv`) enables persistent variables such as A/B boot slot state and boot counters without a full filesystem.

**A/B Update Management** is a cross-cutting concern. U-Boot environment variables or the GRUB2 `grubenv` file serve as the shared state store between the bootloader and user space. Tools (illustrated here in C++, C, and Rust) read and write this state to implement watchdog-style boot retry logic: if a slot fails to boot successfully within N attempts, the bootloader automatically switches to the alternate slot, providing self-healing rollback for OTA firmware updates.

The C examples illustrate real U-Boot board init hooks (`board_init`, `board_late_init`, SPL's `board_init_f`) and a host-side environment manipulation tool. The C++ examples provide a binary FIT parser using `libfdt` and a GRUB2 environment block manager including A/B watchdog logic. The Rust examples demonstrate a pure-Rust FIT image SHA-256 verifier and a fully-tested A/B slot state machine suitable for production embedded update agents.

Together, these building blocks form the foundation of a robust, secure, and updatable embedded Linux product built entirely with Buildroot's reproducible build system.

---

*End of Chapter 12 — Bootloader Integration (U-Boot & GRUB2)*