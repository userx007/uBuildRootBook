# 13. Device Tree & Board Support Packages (BSP)

## Maintaining `.dts`/`.dtsi` Overlays in `BR2_EXTERNAL`, `BR2_LINUX_KERNEL_DTS_SUPPORT`, and Structuring a Complete BSP Layer

---

## Table of Contents

1. [Introduction](#introduction)
2. [Device Tree Fundamentals](#device-tree-fundamentals)
3. [Buildroot BSP Architecture](#buildroot-bsp-architecture)
4. [BR2\_EXTERNAL Layer Structure](#br2_external-layer-structure)
5. [BR2\_LINUX\_KERNEL\_DTS\_SUPPORT](#br2_linux_kernel_dts_support)
6. [Writing DTS and DTSI Files](#writing-dts-and-dtsi-files)
7. [Device Tree Overlays](#device-tree-overlays)
8. [C Programming with Device Trees](#c-programming-with-device-trees)
9. [C++ Programming with Device Trees](#c-programming-with-device-trees-1)
10. [Rust Programming with Device Trees](#rust-programming-with-device-trees)
11. [BSP Package Integration](#bsp-package-integration)
12. [Testing and Debugging](#testing-and-debugging)
13. [Summary](#summary)

---

## Introduction

A **Board Support Package (BSP)** is the foundational software layer that makes a specific hardware platform usable by an operating system. In the Buildroot embedded Linux ecosystem, a BSP typically consists of:

- **Device Tree Source files** (`.dts`, `.dtsi`, `.dtbo`) describing hardware topology
- **Bootloader configuration** (U-Boot board files, SPL, etc.)
- **Linux kernel configuration** and patches
- **Board-specific packages** (GPIO utilities, sensor daemons, factory tools)
- **Filesystem overlays** with init scripts and device rules

The Device Tree is a data structure that describes the hardware to the Linux kernel at runtime, decoupling kernel code from board-specific knowledge. Instead of hardcoding GPIO pin numbers or I²C addresses, the kernel reads them from a compiled binary Device Tree Blob (`.dtb`).

```
ASCII: Relationship between BSP Components

  +--------------------------------------------------+
  |              BR2_EXTERNAL Tree                   |
  |                                                  |
  |  +------------+  +------------+  +-----------+   |
  |  |   board/   |  |  configs/  |  | package/  |   |
  |  |  myboard/  |  | myboard_   |  | mydriver/ |   |
  |  |            |  | defconfig  |  |           |   |
  |  | *.dts      |  +------------+  | Config.in |   |
  |  | *.dtsi     |                  | mydriver  |   |
  |  | *.dtbo     |  +------------+  | .mk       |   |
  |  | patches/   |  |   linux/   |  +-----------+   |
  |  | post-*.sh  |  | linux.conf |                  |
  |  +------------+  +------------+                  |
  +--------------------------------------------------+
           |                |               |
           v                v               v
  +----------------+  +-----------+  +------------+
  | Kernel DTS     |  |  Kernel   |  | Rootfs     |
  | build system   |  |  .config  |  | overlay    |
  | (dtc, cpp)     |  +-----------+  +------------+
  +----------------+
           |
           v
  +----------------+
  |  myboard.dtb   |  <-- loaded by bootloader
  +----------------+
```

---

## Device Tree Fundamentals

### Core Concepts

The Device Tree Specification (from devicetree.org) defines a hierarchical node structure. Each node represents a device or bus. Properties within nodes carry configuration data.

```
ASCII: Device Tree Node Hierarchy

  /  (root)
  |
  +-- cpus
  |     +-- cpu@0  [compatible="arm,cortex-a53"]
  |     +-- cpu@1  [compatible="arm,cortex-a53"]
  |
  +-- memory@80000000  [device_type="memory", reg=<0x80000000 0x40000000>]
  |
  +-- soc
        |
        +-- interrupt-controller@a002000
        |
        +-- serial@11002000  [compatible="ns16550a", reg, clocks, status="okay"]
        |
        +-- i2c@11014000
        |     +-- sensor@48  [compatible="ti,tmp102", reg=<0x48>]
        |
        +-- spi@1100a000
              +-- flash@0    [compatible="jedec,spi-nor", reg=<0>]
```

### Key Property Types

| Property         | Type          | Example                              |
|------------------|---------------|--------------------------------------|
| `compatible`     | string-list   | `"vendor,device", "generic-fallback"`|
| `reg`            | u32 cells     | `<0x11002000 0x1000>`                |
| `interrupts`     | u32 cells     | `<GIC_SPI 32 IRQ_TYPE_LEVEL_HIGH>`   |
| `clocks`         | phandle+args  | `<&clk_sys CLK_UART0>`               |
| `status`         | string        | `"okay"` or `"disabled"`             |
| `#address-cells` | u32           | `<1>`                                |
| `#size-cells`    | u32           | `<1>`                                |

---

## Buildroot BSP Architecture

### How Buildroot Handles BSPs

Buildroot does not have a built-in "BSP" concept. Instead, it provides two integration mechanisms:

1. **`BR2_EXTERNAL`** — an out-of-tree directory for board-specific packages, configs, and files
2. **`BR2_LINUX_KERNEL_DTS_SUPPORT`** — a Kconfig symbol that activates in-tree DTS compilation

The canonical approach is to combine both: keep your `.dts` files in `BR2_EXTERNAL/board/`, and tell the kernel build where to find them.

```
ASCII: Buildroot Build Flow with BSP

  make myboard_defconfig
        |
        v
  +---------------------+
  | .config (Buildroot) |
  | BR2_EXTERNAL=...    |
  | BR2_LINUX_KERNEL_   |
  |   DTS_SUPPORT=y     |
  | BR2_LINUX_KERNEL_   |
  |   INTREE_DTS_NAME=  |
  |   "myboard"         |
  +---------------------+
        |
        +-------> Toolchain build
        |
        +-------> Linux kernel build
        |               |
        |    cpp  + dtc v
        |         +-----------+
        |         | myboard   |
        |         |   .dtb    |
        |         +-----------+
        |
        +-------> Package builds
        |
        +-------> post-build.sh / post-image.sh
        |
        v
  output/images/
    |- Image (or zImage)
    |- myboard.dtb
    |- rootfs.ext4
    |- sdcard.img
```

---

## BR2\_EXTERNAL Layer Structure

### Directory Layout

A well-structured `BR2_EXTERNAL` tree for a board called `myboard` (SoC: `mysoc`) looks like:

```
my-bsp/
├── Config.in                          # Top-level Kconfig for external tree
├── external.mk                        # Include all package .mk files
├── external.desc                      # Name and description
│
├── board/
│   └── myboard/
│       ├── dts/
│       │   ├── mysoc.dtsi             # SoC-level include
│       │   ├── mysoc-pinctrl.dtsi     # Pinmux definitions
│       │   ├── myboard.dts            # Board-level top file
│       │   ├── myboard-overlay-i2s.dtbo  # Audio overlay
│       │   └── myboard-overlay-lcd.dtbo  # LCD overlay
│       │
│       ├── patches/
│       │   └── linux/
│       │       └── 0001-myboard-add-dts.patch
│       │
│       ├── rootfs-overlay/
│       │   ├── etc/
│       │   │   ├── inittab
│       │   │   └── network/interfaces
│       │   └── lib/firmware/
│       │       └── mysoc-wifi.bin
│       │
│       ├── post-build.sh
│       └── post-image.sh
│
├── configs/
│   └── myboard_defconfig
│
├── linux/
│   └── linux.config                   # Linux kernel fragment
│
└── package/
    ├── myboard-tools/
    │   ├── Config.in
    │   └── myboard-tools.mk
    └── mysoc-firmware/
        ├── Config.in
        └── mysoc-firmware.mk
```

### `external.desc`

```
name: MYBOARD_BSP
desc: Board Support Package for MyBoard (MySoC platform)
```

### Top-level `Config.in`

```kconfig
# Config.in - BR2_EXTERNAL Kconfig root

mainmenu "MyBoard BSP Configuration"

source "$BR2_EXTERNAL_MYBOARD_BSP_PATH/package/myboard-tools/Config.in"
source "$BR2_EXTERNAL_MYBOARD_BSP_PATH/package/mysoc-firmware/Config.in"
```

### `external.mk`

```makefile
# external.mk — include all package makefiles

include $(sort $(wildcard $(BR2_EXTERNAL_MYBOARD_BSP_PATH)/package/*/*.mk))
```

### `configs/myboard_defconfig`

```
# myboard_defconfig - Buildroot defconfig for MyBoard

BR2_aarch64=y
BR2_TOOLCHAIN_EXTERNAL=y
BR2_TOOLCHAIN_EXTERNAL_URL="https://toolchains.bootlin.com/..."

BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="6.6.30"
BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG=y
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="$(BR2_EXTERNAL_MYBOARD_BSP_PATH)/linux/linux.config"

# DTS support — key settings
BR2_LINUX_KERNEL_DTS_SUPPORT=y
BR2_LINUX_KERNEL_INTREE_DTS_NAME="myboard"
# Or for out-of-tree:
# BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_MYBOARD_BSP_PATH)/board/myboard/dts/myboard.dts"

BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y

BR2_PACKAGE_MYBOARD_TOOLS=y
BR2_PACKAGE_MYSOC_FIRMWARE=y

BR2_ROOTFS_OVERLAY="$(BR2_EXTERNAL_MYBOARD_BSP_PATH)/board/myboard/rootfs-overlay"
BR2_ROOTFS_POST_BUILD_SCRIPT="$(BR2_EXTERNAL_MYBOARD_BSP_PATH)/board/myboard/post-build.sh"
BR2_ROOTFS_POST_IMAGE_SCRIPT="$(BR2_EXTERNAL_MYBOARD_BSP_PATH)/board/myboard/post-image.sh"
```

---

## BR2\_LINUX\_KERNEL\_DTS\_SUPPORT

### What the Symbol Does

When `BR2_LINUX_KERNEL_DTS_SUPPORT=y` is set, Buildroot instructs the Linux kernel `Makefile` to compile Device Tree Source files and install the resulting `.dtb` files into `output/images/`.

The key Kconfig symbols in `linux/linux.mk` (simplified):

```
ASCII: DTS Kconfig Decision Tree

  BR2_LINUX_KERNEL_DTS_SUPPORT=y ?
         |
    YES  |
         v
  Use in-tree DTS ?          Use custom DTS path ?
  (INTREE_DTS_NAME)          (CUSTOM_DTS_PATH)
         |                           |
    +----+----+               +------+------+
    |         |               |             |
  single   multiple        single       multiple
  name     names (space    .dts file   .dts files
           separated)      or dir      (space sep.)
    |         |               |             |
    v         v               v             v
  dtbs: $(INTREE)  Copy to kernel src tree, then compile
```

### In-tree DTS Names

For boards whose DTS ships with the mainline kernel (e.g. Raspberry Pi, BeagleBone):

```
# Single board
BR2_LINUX_KERNEL_INTREE_DTS_NAME="bcm2711-rpi-4-b"

# Multiple DTBs for the same board variant
BR2_LINUX_KERNEL_INTREE_DTS_NAME="bcm2711-rpi-4-b bcm2711-rpi-400"
```

Buildroot passes these to:

```makefile
$(MAKE) $(LINUX_MAKE_FLAGS) -C $(LINUX_DIR) \
    $(LINUX_KERNEL_DTS_TARGETS)
```

where `LINUX_KERNEL_DTS_TARGETS` expands to `dtbs` with the named files.

### Custom (Out-of-Tree) DTS Path

For proprietary or not-yet-upstream boards:

```
# Single file
BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="board/myboard/dts/myboard.dts"

# Entire directory (all .dts files compiled)
BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="board/myboard/dts/"
```

Buildroot copies the custom `.dts`/`.dtsi` files into the kernel source tree's `arch/arm64/boot/dts/vendor/` (or equivalent) before the kernel build runs.

### Linux Kernel Fragment for DTS

Place a kernel config fragment at `linux/linux.config`:

```kconfig
# linux.config — kernel configuration fragment

CONFIG_OF=y
CONFIG_OF_EARLY_FLATTREE=y
CONFIG_OF_KOBJ=y
CONFIG_OF_DYNAMIC=y
CONFIG_OF_ADDRESS=y
CONFIG_OF_IRQ=y
CONFIG_OF_NET=y
CONFIG_OF_MDIO=y
CONFIG_OF_OVERLAY=y        # needed for .dtbo overlays
CONFIG_OF_CONFIGFS=y       # apply overlays from userspace via configfs
```

---

## Writing DTS and DTSI Files

### SoC-Level DTSI: `mysoc.dtsi`

```dts
// SPDX-License-Identifier: GPL-2.0-or-later
/*
 * MySoC — SoC-level Device Tree include
 * Copyright (C) 2024 MyCompany
 */

#include <dt-bindings/interrupt-controller/arm-gic.h>
#include <dt-bindings/clock/mysoc-clk.h>
#include <dt-bindings/gpio/gpio.h>

/ {
    #address-cells = <2>;
    #size-cells = <2>;
    interrupt-parent = <&gic>;

    /* -------------------------------------------------------
     * CPU cluster
     * ------------------------------------------------------- */
    cpus {
        #address-cells = <1>;
        #size-cells = <0>;

        cpu0: cpu@0 {
            device_type = "cpu";
            compatible = "arm,cortex-a53";
            reg = <0x0>;
            enable-method = "psci";
            next-level-cache = <&L2_0>;
        };

        cpu1: cpu@1 {
            device_type = "cpu";
            compatible = "arm,cortex-a53";
            reg = <0x1>;
            enable-method = "psci";
            next-level-cache = <&L2_0>;
        };

        L2_0: l2-cache0 {
            compatible = "cache";
            cache-level = <2>;
            cache-unified;
        };
    };

    /* -------------------------------------------------------
     * Power State Coordination Interface
     * ------------------------------------------------------- */
    psci {
        compatible = "arm,psci-1.0";
        method = "smc";
    };

    /* -------------------------------------------------------
     * Fixed clocks
     * ------------------------------------------------------- */
    osc24m: oscillator-24m {
        compatible = "fixed-clock";
        #clock-cells = <0>;
        clock-frequency = <24000000>;
        clock-output-names = "osc24M";
    };

    /* -------------------------------------------------------
     * SoC bus
     * ------------------------------------------------------- */
    soc: soc {
        #address-cells = <1>;
        #size-cells = <1>;
        compatible = "simple-bus";
        ranges;

        /* GIC interrupt controller */
        gic: interrupt-controller@a002000 {
            compatible = "arm,gic-400";
            #interrupt-cells = <3>;
            interrupt-controller;
            reg = <0x0a002000 0x1000>,
                  <0x0a004000 0x2000>;
        };

        /* Clock controller */
        ccu: clock@1c20000 {
            compatible = "myvendor,mysoc-ccu";
            reg = <0x01c20000 0x400>;
            clocks = <&osc24m>;
            #clock-cells = <1>;
            #reset-cells = <1>;
        };

        /* UART0 */
        uart0: serial@1c28000 {
            compatible = "snps,dw-apb-uart";
            reg = <0x01c28000 0x400>;
            interrupts = <GIC_SPI 0 IRQ_TYPE_LEVEL_HIGH>;
            reg-shift = <2>;
            reg-io-width = <4>;
            clocks = <&ccu CLK_BUS_UART0>, <&ccu CLK_UART0>;
            clock-names = "bus", "baud";
            resets = <&ccu RST_BUS_UART0>;
            status = "disabled";   /* enabled in board .dts */
        };

        /* I2C0 */
        i2c0: i2c@1c2ac00 {
            compatible = "myvendor,mysoc-i2c";
            reg = <0x01c2ac00 0x400>;
            interrupts = <GIC_SPI 6 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&ccu CLK_BUS_I2C0>;
            resets = <&ccu RST_BUS_I2C0>;
            #address-cells = <1>;
            #size-cells = <0>;
            status = "disabled";
        };

        /* SPI0 */
        spi0: spi@1c68000 {
            compatible = "myvendor,mysoc-spi";
            reg = <0x01c68000 0x1000>;
            interrupts = <GIC_SPI 65 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&ccu CLK_BUS_SPI0>, <&ccu CLK_SPI0>;
            clock-names = "ahb", "mod";
            resets = <&ccu RST_BUS_SPI0>;
            #address-cells = <1>;
            #size-cells = <0>;
            status = "disabled";
        };

        /* Ethernet MAC */
        emac: ethernet@1c30000 {
            compatible = "allwinner,sun8i-h3-emac";
            reg = <0x01c30000 0x10000>,
                  <0x01c00030 0x4>;
            interrupts = <GIC_SPI 82 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&ccu CLK_BUS_EMAC>;
            resets = <&ccu RST_BUS_EMAC>;
            phy-handle = <&phy0>;
            status = "disabled";
        };
    };
};
```

### Pinmux DTSI: `mysoc-pinctrl.dtsi`

```dts
// SPDX-License-Identifier: GPL-2.0-or-later
/*
 * MySoC pinmux definitions
 */

&pio {
    uart0_pins: uart0-pins {
        pins = "PH0", "PH1";
        function = "uart0";
    };

    i2c0_pins: i2c0-pins {
        pins = "PA11", "PA12";
        function = "i2c0";
        bias-pull-up;
    };

    spi0_pins: spi0-pins {
        pins = "PC0", "PC1", "PC2", "PC3";
        function = "spi0";
    };

    emac_pins: emac-pins {
        pins = "PD0", "PD1", "PD2", "PD3",
               "PD4", "PD5", "PD6", "PD7",
               "PD8", "PD9", "PD10", "PD11";
        function = "emac";
        drive-strength = <40>;
    };
};
```

### Board-Level DTS: `myboard.dts`

```dts
// SPDX-License-Identifier: GPL-2.0-or-later
/*
 * MyBoard — top-level Device Tree
 * Hardware: MySoC + 1GiB LPDDR4 + 8GiB eMMC
 * Copyright (C) 2024 MyCompany <bsp@mycompany.com>
 */

/dts-v1/;
#include "mysoc.dtsi"
#include "mysoc-pinctrl.dtsi"

/ {
    model = "MyCompany MyBoard v1.0";
    compatible = "mycompany,myboard", "myvendor,mysoc";

    /* -------------------------------------------------------
     * Memory
     * ------------------------------------------------------- */
    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x40000000>;   /* 1 GiB at 2 GiB offset */
    };

    /* -------------------------------------------------------
     * Chosen — kernel command line & boot args
     * ------------------------------------------------------- */
    chosen {
        stdout-path = "serial0:115200n8";
        bootargs = "console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw";
    };

    /* -------------------------------------------------------
     * Fixed regulators
     * ------------------------------------------------------- */
    reg_vcc3v3: regulator-vcc3v3 {
        compatible = "regulator-fixed";
        regulator-name = "VCC-3V3";
        regulator-min-microvolt = <3300000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
    };

    reg_vcc1v8: regulator-vcc1v8 {
        compatible = "regulator-fixed";
        regulator-name = "VCC-1V8";
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
    };

    /* -------------------------------------------------------
     * LEDs
     * ------------------------------------------------------- */
    leds {
        compatible = "gpio-leds";

        led-heartbeat {
            label = "myboard:green:heartbeat";
            gpios = <&pio 7 10 GPIO_ACTIVE_LOW>;   /* PH10 */
            linux,default-trigger = "heartbeat";
        };

        led-activity {
            label = "myboard:red:activity";
            gpios = <&pio 7 11 GPIO_ACTIVE_HIGH>;  /* PH11 */
            linux,default-trigger = "mmc0";
        };
    };

    /* -------------------------------------------------------
     * User push-button
     * ------------------------------------------------------- */
    keys {
        compatible = "gpio-keys";

        key-recovery {
            label = "RECOVERY";
            gpios = <&pio 6 3 GPIO_ACTIVE_LOW>;   /* PG3 */
            linux,code = <KEY_RESTART>;
            debounce-interval = <10>;
            wakeup-source;
        };
    };
};

/* -------------------------------------------------------
 * Enable UART0 as console
 * ------------------------------------------------------- */
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;
    status = "okay";
};

/* -------------------------------------------------------
 * I2C0 with an on-board temperature sensor
 * ------------------------------------------------------- */
&i2c0 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c0_pins>;
    clock-frequency = <400000>;
    status = "okay";

    tmp102: temperature-sensor@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
        interrupt-parent = <&pio>;
        interrupts = <1 15 IRQ_TYPE_LEVEL_LOW>;   /* PB15 ALERT */
        #thermal-sensor-cells = <1>;
    };
};

/* -------------------------------------------------------
 * SPI0 with NOR flash
 * ------------------------------------------------------- */
&spi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_pins>;
    status = "okay";

    flash: nor-flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <40000000>;
        #address-cells = <1>;
        #size-cells = <1>;

        partition@0 {
            label = "spl";
            reg = <0x000000 0x040000>;
            read-only;
        };

        partition@40000 {
            label = "u-boot";
            reg = <0x040000 0x0c0000>;
        };

        partition@100000 {
            label = "env";
            reg = <0x100000 0x010000>;
        };
    };
};

/* -------------------------------------------------------
 * Ethernet
 * ------------------------------------------------------- */
&emac {
    pinctrl-names = "default";
    pinctrl-0 = <&emac_pins>;
    phy-mode = "rgmii-id";
    phy-supply = <&reg_vcc3v3>;
    status = "okay";

    mdio: mdio {
        #address-cells = <1>;
        #size-cells = <0>;

        phy0: ethernet-phy@1 {
            compatible = "ethernet-phy-ieee802.3-c22";
            reg = <1>;
            reset-gpios = <&pio 3 14 GPIO_ACTIVE_LOW>;  /* PD14 */
            reset-assert-us = <10000>;
            reset-deassert-us = <30000>;
        };
    };
};
```

---

## Device Tree Overlays

Overlays (`.dtbo`) allow runtime modification of the Device Tree without recompiling the base blob. They are critical for optional hardware like expansion boards, HATs, and click modules.

```
ASCII: DT Overlay Application Flow

  Base DTB:                   Overlay DTBO:
  +------------------+        +------------------+
  | / root           |        | /fragment@0      |
  |   soc            |        |   target = &i2c0 |
  |     i2c0 {       |  --->  |   __overlay__ {  |
  |       status=    |  merge |     status=okay  |
  |         disabled |        |     sensor@1d {  |
  |     }            |        |       compat=... |
  +------------------+        +------------------+
          |                           |
          +----------+----------------+
                     |
                     v
          +------------------+
          | Merged in-memory |
          |   i2c0 {         |
          |     status=okay  |
          |     sensor@1d {} |
          |   }              |
          +------------------+
```

### Audio I2S Overlay: `myboard-overlay-i2s.dtbo`

```dts
/dts-v1/;
/plugin/;

#include <dt-bindings/gpio/gpio.h>

&{/} {
    sound {
        compatible = "simple-audio-card";
        simple-audio-card,name = "MyBoard I2S Audio";
        simple-audio-card,format = "i2s";
        simple-audio-card,frame-master = <&codec_node>;
        simple-audio-card,bitclock-master = <&codec_node>;

        simple-audio-card,cpu {
            sound-dai = <&i2s0>;
        };

        codec_node: simple-audio-card,codec {
            sound-dai = <&pcm5102a>;
        };
    };

    pcm5102a: pcm5102a {
        compatible = "ti,pcm5102a";
        #sound-dai-cells = <0>;
        pcm512x,format = "i2s";
    };
};

&i2s0 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2s0_pins>;
    status = "okay";
};
```

### Applying Overlays via configfs (shell)

```bash
#!/bin/sh
# apply-overlay.sh — mount configfs and load a DTBO overlay

CONFIGFS=/sys/kernel/config/device-tree/overlays
OVERLAY_NAME="myboard-i2s"
OVERLAY_PATH="/lib/firmware/myboard-overlay-i2s.dtbo"

# Mount configfs if not already mounted
mountpoint -q /sys/kernel/config || mount -t configfs none /sys/kernel/config

# Create a slot for the overlay
mkdir -p "${CONFIGFS}/${OVERLAY_NAME}"

# Load the overlay binary
cat "${OVERLAY_PATH}" > "${CONFIGFS}/${OVERLAY_NAME}/dtbo"

echo "Overlay ${OVERLAY_NAME} applied."
echo "Status: $(cat ${CONFIGFS}/${OVERLAY_NAME}/status)"
```

---

## C Programming with Device Trees

### 1. Reading Device Tree Properties from Userspace via libfdt

`libfdt` (Flat Device Tree library) parses DTB blobs directly from userspace. It ships with dtc and is available as a Buildroot package.

```c
/*
 * dt_read_props.c
 * Read hardware description from /sys/firmware/fdt using libfdt.
 * Build: aarch64-linux-gnu-gcc -o dt_read_props dt_read_props.c -lfdt
 */

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <libfdt.h>

#define FDT_PATH "/sys/firmware/fdt"
#define MAX_FDT_SIZE (2 * 1024 * 1024)  /* 2 MiB */

/* Print a property value as hex bytes */
static void print_prop_hex(const char *name, const void *data, int len)
{
    const uint8_t *p = data;
    printf("  %-30s = <", name);
    for (int i = 0; i < len; i++) {
        if (i > 0 && (i % 4) == 0) printf(" ");
        printf("%02x", p[i]);
    }
    printf(">\n");
}

/* Walk and print all nodes matching a compatible string */
static int find_compatible(const void *fdt, const char *compat)
{
    int offset = 0;
    int found = 0;

    printf("Searching for compatible=\"%s\"\n\n", compat);

    for (;;) {
        offset = fdt_node_offset_by_compatible(fdt, offset, compat);
        if (offset < 0)
            break;

        found++;
        printf("Node: %s\n", fdt_get_name(fdt, offset, NULL));

        /* Dump all properties of this node */
        int prop_offset;
        fdt_for_each_property_offset(prop_offset, fdt, offset) {
            int len;
            const char *name;
            const void *data = fdt_getprop_by_offset(fdt, prop_offset,
                                                      &name, &len);
            if (data)
                print_prop_hex(name, data, len);
        }
        printf("\n");
    }

    return found;
}

/* Read a u32 property from a specific path */
static int read_u32_prop(const void *fdt, const char *path,
                         const char *prop_name, uint32_t *out)
{
    int offset = fdt_path_offset(fdt, path);
    if (offset < 0) {
        fprintf(stderr, "Node not found: %s (%s)\n",
                path, fdt_strerror(offset));
        return -1;
    }

    int len;
    const uint32_t *val = fdt_getprop(fdt, offset, prop_name, &len);
    if (!val || len < 4) {
        fprintf(stderr, "Property not found: %s\n", prop_name);
        return -1;
    }

    *out = fdt32_to_cpu(*val);
    return 0;
}

int main(void)
{
    /* Load FDT from sysfs */
    int fd = open(FDT_PATH, O_RDONLY);
    if (fd < 0) {
        perror("open /sys/firmware/fdt");
        return EXIT_FAILURE;
    }

    void *fdt = malloc(MAX_FDT_SIZE);
    if (!fdt) {
        perror("malloc");
        close(fd);
        return EXIT_FAILURE;
    }

    ssize_t n = read(fd, fdt, MAX_FDT_SIZE);
    close(fd);

    if (n < 0) {
        perror("read fdt");
        free(fdt);
        return EXIT_FAILURE;
    }

    /* Validate the DTB magic */
    if (fdt_check_header(fdt) != 0) {
        fprintf(stderr, "Invalid FDT header\n");
        free(fdt);
        return EXIT_FAILURE;
    }

    printf("FDT version: %d, size: %d bytes\n\n",
           fdt_version(fdt), fdt_totalsize(fdt));

    /* Print model string */
    const char *model = fdt_getprop(fdt, 0, "model", NULL);
    if (model)
        printf("Board model: %s\n\n", model);

    /* Find temperature sensors */
    find_compatible(fdt, "ti,tmp102");

    /* Read memory base address */
    uint32_t mem_base = 0;
    if (read_u32_prop(fdt, "/memory@80000000", "reg", &mem_base) == 0)
        printf("Memory base: 0x%08x\n", mem_base);

    free(fdt);
    return EXIT_SUCCESS;
}
```

### 2. Kernel Driver using OF (Open Firmware) API

```c
/*
 * myboard_sensor.c
 * Example kernel driver that uses Device Tree bindings.
 * Demonstrates of_property_read_*, of_get_named_gpio, of_clk_get.
 */

#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/of_gpio.h>
#include <linux/gpio/consumer.h>
#include <linux/clk.h>
#include <linux/i2c.h>
#include <linux/regmap.h>
#include <linux/interrupt.h>
#include <linux/thermal.h>

#define DRIVER_NAME "myboard-sensor"

struct myboard_sensor {
    struct device          *dev;
    struct i2c_client      *client;
    struct regmap          *regmap;
    struct clk             *clk;
    struct gpio_desc       *reset_gpio;
    struct gpio_desc       *alert_gpio;
    int                     irq;
    struct thermal_zone_device *tz;
    /* Cached readings */
    s32                     last_temp_mC;
};

/* Regmap configuration for the sensor */
static const struct regmap_config myboard_regmap_config = {
    .reg_bits  = 8,
    .val_bits  = 16,
    .max_register = 0x03,
};

static irqreturn_t myboard_sensor_irq(int irq, void *data)
{
    struct myboard_sensor *s = data;
    dev_info(s->dev, "ALERT: temperature threshold crossed\n");
    thermal_zone_device_update(s->tz, THERMAL_EVENT_UNSPECIFIED);
    return IRQ_HANDLED;
}

static int myboard_sensor_get_temp(struct thermal_zone_device *tz, int *temp)
{
    struct myboard_sensor *s = thermal_zone_device_priv(tz);
    unsigned int raw;
    int ret;

    ret = regmap_read(s->regmap, 0x00, &raw);  /* TMP102 temp register */
    if (ret)
        return ret;

    /* TMP102: 12-bit value, 0.0625°C per LSB, in upper 12 bits */
    s16 raw12 = (s16)(raw & 0xFFF0) >> 4;
    *temp = (int)raw12 * 625 / 10;  /* millidegrees Celsius */
    s->last_temp_mC = *temp;
    return 0;
}

static const struct thermal_zone_device_ops myboard_tz_ops = {
    .get_temp = myboard_sensor_get_temp,
};

static int myboard_sensor_probe(struct i2c_client *client)
{
    struct device *dev = &client->dev;
    struct device_node *np = dev->of_node;
    struct myboard_sensor *s;
    u32 poll_interval_ms;
    int ret;

    s = devm_kzalloc(dev, sizeof(*s), GFP_KERNEL);
    if (!s)
        return -ENOMEM;

    s->dev    = dev;
    s->client = client;
    i2c_set_clientdata(client, s);

    /* ---- Parse Device Tree properties ---- */

    /* Optional: polling interval (default 1000 ms) */
    if (of_property_read_u32(np, "mycompany,poll-interval-ms",
                              &poll_interval_ms))
        poll_interval_ms = 1000;

    dev_info(dev, "poll interval: %u ms\n", poll_interval_ms);

    /* Optional clock */
    s->clk = devm_clk_get_optional(dev, "sensor");
    if (IS_ERR(s->clk))
        return dev_err_probe(dev, PTR_ERR(s->clk), "failed to get clock\n");

    if (s->clk) {
        ret = clk_prepare_enable(s->clk);
        if (ret)
            return dev_err_probe(dev, ret, "failed to enable clock\n");
    }

    /* Active-low reset GPIO */
    s->reset_gpio = devm_gpiod_get_optional(dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(s->reset_gpio))
        return dev_err_probe(dev, PTR_ERR(s->reset_gpio),
                             "failed to get reset GPIO\n");

    if (s->reset_gpio) {
        /* Pulse reset: assert for 1 ms, then deassert */
        gpiod_set_value_cansleep(s->reset_gpio, 1);
        usleep_range(1000, 2000);
        gpiod_set_value_cansleep(s->reset_gpio, 0);
        usleep_range(5000, 10000);
    }

    /* Alert interrupt GPIO */
    s->alert_gpio = devm_gpiod_get_optional(dev, "alert", GPIOD_IN);
    if (IS_ERR(s->alert_gpio))
        return dev_err_probe(dev, PTR_ERR(s->alert_gpio),
                             "failed to get alert GPIO\n");

    /* ---- Set up regmap ---- */
    s->regmap = devm_regmap_init_i2c(client, &myboard_regmap_config);
    if (IS_ERR(s->regmap))
        return dev_err_probe(dev, PTR_ERR(s->regmap),
                             "failed to init regmap\n");

    /* ---- Register thermal zone ---- */
    s->tz = devm_thermal_of_zone_register(dev, 0, s, &myboard_tz_ops);
    if (IS_ERR(s->tz))
        return dev_err_probe(dev, PTR_ERR(s->tz),
                             "failed to register thermal zone\n");

    /* ---- Request interrupt ---- */
    if (s->alert_gpio) {
        s->irq = gpiod_to_irq(s->alert_gpio);
        ret = devm_request_threaded_irq(dev, s->irq, NULL,
                                        myboard_sensor_irq,
                                        IRQF_TRIGGER_LOW | IRQF_ONESHOT,
                                        DRIVER_NAME, s);
        if (ret)
            return dev_err_probe(dev, ret, "failed to request IRQ\n");
    }

    dev_info(dev, "MyBoard sensor probed at 0x%02x\n", client->addr);
    return 0;
}

/* ---- Device Tree matching table ---- */
static const struct of_device_id myboard_sensor_of_match[] = {
    { .compatible = "mycompany,myboard-sensor" },
    { .compatible = "ti,tmp102" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, myboard_sensor_of_match);

static const struct i2c_device_id myboard_sensor_id[] = {
    { "myboard-sensor", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, myboard_sensor_id);

static struct i2c_driver myboard_sensor_driver = {
    .driver = {
        .name           = DRIVER_NAME,
        .of_match_table = myboard_sensor_of_match,
    },
    .probe    = myboard_sensor_probe,
    .id_table = myboard_sensor_id,
};

module_i2c_driver(myboard_sensor_driver);

MODULE_AUTHOR("MyCompany BSP Team <bsp@mycompany.com>");
MODULE_DESCRIPTION("MyBoard temperature sensor driver");
MODULE_LICENSE("GPL");
```

### 3. Userspace GPIO Control via libgpiod

```c
/*
 * board_gpio_test.c
 * Toggle board LEDs using libgpiod (the modern GPIO userspace API).
 * GPIO chip and line numbers come from DTS gpio-leds node.
 * Build: aarch64-linux-gnu-gcc -o board_gpio_test board_gpio_test.c -lgpiod
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <gpiod.h>

/*
 * From DTS:
 *   gpios = <&pio 7 10 GPIO_ACTIVE_LOW>;   PH10 = chip0, line 10+(7*32)
 *
 * In practice, find the chip and offset via:
 *   $ gpiodetect
 *   $ gpioinfo gpiochip0
 */
#define GPIO_CHIP        "gpiochip0"
#define LED_HEARTBEAT    (7 * 32 + 10)   /* PH10 */
#define LED_ACTIVITY     (7 * 32 + 11)   /* PH11 */
#define BLINK_COUNT      10
#define BLINK_PERIOD_US  500000

int main(void)
{
    struct gpiod_chip *chip;
    struct gpiod_line *led_hb, *led_act;
    int ret;

    chip = gpiod_chip_open_by_name(GPIO_CHIP);
    if (!chip) {
        perror("gpiod_chip_open");
        return EXIT_FAILURE;
    }

    led_hb = gpiod_chip_get_line(chip, LED_HEARTBEAT);
    if (!led_hb) {
        perror("get heartbeat LED line");
        goto err_chip;
    }

    led_act = gpiod_chip_get_line(chip, LED_ACTIVITY);
    if (!led_act) {
        perror("get activity LED line");
        goto err_chip;
    }

    /* Request lines as output, initially off (active low = value 0 means LED on) */
    ret = gpiod_line_request_output(led_hb, "board_gpio_test", 1);
    if (ret < 0) { perror("request heartbeat"); goto err_chip; }

    ret = gpiod_line_request_output(led_act, "board_gpio_test", 0);
    if (ret < 0) { perror("request activity"); goto err_chip; }

    printf("Blinking LEDs %d times...\n", BLINK_COUNT);

    for (int i = 0; i < BLINK_COUNT; i++) {
        /* Alternate: heartbeat off, activity on */
        gpiod_line_set_value(led_hb,  1);
        gpiod_line_set_value(led_act, 1);
        usleep(BLINK_PERIOD_US / 2);

        /* Alternate: heartbeat on, activity off */
        gpiod_line_set_value(led_hb,  0);
        gpiod_line_set_value(led_act, 0);
        usleep(BLINK_PERIOD_US / 2);

        printf("  Blink %2d/%d\r", i + 1, BLINK_COUNT);
        fflush(stdout);
    }

    printf("\nDone.\n");

    /* Release lines */
    gpiod_line_release(led_hb);
    gpiod_line_release(led_act);

err_chip:
    gpiod_chip_close(chip);
    return EXIT_SUCCESS;
}
```

---

## C++ Programming with Device Trees

### DT Property Parser Class

```cpp
/*
 * dt_parser.cpp
 * C++ RAII wrapper around libfdt for userspace DT inspection.
 * Build: aarch64-linux-gnu-g++ -std=c++17 -o dt_parser dt_parser.cpp -lfdt
 */

#include <cstdint>
#include <cstring>
#include <fstream>
#include <functional>
#include <iostream>
#include <optional>
#include <span>
#include <stdexcept>
#include <string>
#include <string_view>
#include <vector>

extern "C" {
#include <libfdt.h>
}

namespace bsp {

class DeviceTree {
public:
    /* Load the FDT from the running system */
    explicit DeviceTree(const std::string& path = "/sys/firmware/fdt")
    {
        std::ifstream file(path, std::ios::binary | std::ios::ate);
        if (!file.is_open())
            throw std::runtime_error("Cannot open FDT: " + path);

        auto size = file.tellg();
        blob_.resize(static_cast<std::size_t>(size));
        file.seekg(0);
        file.read(reinterpret_cast<char*>(blob_.data()), size);

        if (fdt_check_header(blob_.data()) != 0)
            throw std::runtime_error("Invalid FDT header in: " + path);
    }

    /* Non-copyable, movable */
    DeviceTree(const DeviceTree&) = delete;
    DeviceTree& operator=(const DeviceTree&) = delete;
    DeviceTree(DeviceTree&&) = default;

    /* --- Query API --- */

    std::string_view model() const
    {
        const char* m = static_cast<const char*>(
            fdt_getprop(blob_.data(), 0, "model", nullptr));
        return m ? m : "(unknown)";
    }

    std::optional<std::string> stringProp(std::string_view path,
                                           std::string_view prop) const
    {
        int offset = fdt_path_offset(blob_.data(), std::string(path).c_str());
        if (offset < 0) return std::nullopt;

        int len = 0;
        const char* val = static_cast<const char*>(
            fdt_getprop(blob_.data(), offset, std::string(prop).c_str(), &len));
        if (!val) return std::nullopt;
        return std::string(val, static_cast<std::size_t>(len > 0 ? len - 1 : 0));
    }

    std::optional<uint32_t> u32Prop(std::string_view path,
                                     std::string_view prop) const
    {
        int offset = fdt_path_offset(blob_.data(), std::string(path).c_str());
        if (offset < 0) return std::nullopt;

        int len = 0;
        const void* val = fdt_getprop(blob_.data(), offset,
                                       std::string(prop).c_str(), &len);
        if (!val || len < 4) return std::nullopt;
        return fdt32_to_cpu(*static_cast<const uint32_t*>(val));
    }

    /* Iterate over all nodes with a given compatible string */
    void forEachCompatible(std::string_view compat,
                            std::function<void(std::string_view name,
                                               int offset)> cb) const
    {
        int offset = 0;
        for (;;) {
            offset = fdt_node_offset_by_compatible(
                blob_.data(), offset, std::string(compat).c_str());
            if (offset < 0) break;
            const char* name = fdt_get_name(blob_.data(), offset, nullptr);
            cb(name ? name : "(noname)", offset);
        }
    }

    /* Get reg property as vector of {addr, size} pairs */
    std::vector<std::pair<uint64_t, uint64_t>>
    regProp(std::string_view path) const
    {
        int offset = fdt_path_offset(blob_.data(), std::string(path).c_str());
        if (offset < 0) return {};

        int len = 0;
        const uint32_t* reg = static_cast<const uint32_t*>(
            fdt_getprop(blob_.data(), offset, "reg", &len));
        if (!reg) return {};

        std::vector<std::pair<uint64_t, uint64_t>> result;
        int cells = len / sizeof(uint32_t);
        /* Assume #address-cells=1, #size-cells=1 for simplicity */
        for (int i = 0; i + 1 < cells; i += 2) {
            result.emplace_back(fdt32_to_cpu(reg[i]),
                                fdt32_to_cpu(reg[i + 1]));
        }
        return result;
    }

    int version() const { return fdt_version(blob_.data()); }
    int totalSize() const { return fdt_totalsize(blob_.data()); }

private:
    std::vector<uint8_t> blob_;
};

} // namespace bsp

/* ---- Demo main ---- */

int main()
{
    try {
        bsp::DeviceTree dt;

        std::cout << "=== Device Tree Inspector ===\n";
        std::cout << "Version   : " << dt.version() << "\n";
        std::cout << "Total size: " << dt.totalSize() << " bytes\n";
        std::cout << "Model     : " << dt.model() << "\n\n";

        /* Find all UARTs */
        std::cout << "--- UART Nodes ---\n";
        dt.forEachCompatible("snps,dw-apb-uart",
            [&](std::string_view name, int /*offset*/) {
                std::cout << "  Found: " << name << "\n";
            });

        /* Read memory layout */
        std::cout << "\n--- Memory ---\n";
        auto regs = dt.regProp("/memory@80000000");
        for (auto& [addr, size] : regs) {
            std::cout << "  Base: 0x" << std::hex << addr
                      << "  Size: 0x" << size
                      << " (" << std::dec << (size >> 20) << " MiB)\n";
        }

        /* Read chosen bootargs */
        std::cout << "\n--- Chosen ---\n";
        if (auto bootargs = dt.stringProp("/chosen", "bootargs"))
            std::cout << "  bootargs: " << *bootargs << "\n";

    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

### C++ BSP Initialization Manager

```cpp
/*
 * bsp_init.cpp
 * High-level C++ BSP initializer that reads DT, configures peripherals,
 * and manages device lifecycle using RAII.
 */

#include <chrono>
#include <filesystem>
#include <format>
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>
#include <thread>
#include <vector>

namespace bsp {

/* -------------------------------------------------------
 * Peripheral base class
 * ------------------------------------------------------- */
class Peripheral {
public:
    virtual ~Peripheral() = default;
    virtual void initialize()  = 0;
    virtual void shutdown()    = 0;
    virtual std::string name() const = 0;

protected:
    bool initialized_ = false;
};

/* -------------------------------------------------------
 * LED controller (reads from DTS gpio-leds)
 * ------------------------------------------------------- */
class LedController : public Peripheral {
public:
    struct LedEntry {
        std::string label;
        std::string sysfs_path;  /* /sys/class/leds/<label>/ */
    };

    explicit LedController() = default;

    void initialize() override
    {
        namespace fs = std::filesystem;
        const fs::path led_base = "/sys/class/leds";

        if (!fs::exists(led_base)) {
            throw std::runtime_error("LED sysfs not available");
        }

        for (auto& entry : fs::directory_iterator(led_base)) {
            leds_.push_back({entry.path().filename().string(),
                             entry.path().string()});
        }

        initialized_ = true;
        std::cout << std::format("[{}] Discovered {} LED(s)\n",
                                  name(), leds_.size());
    }

    void setTrigger(const std::string& led_label,
                    const std::string& trigger) const
    {
        for (auto& led : leds_) {
            if (led.label.find(led_label) != std::string::npos) {
                auto path = led.sysfs_path + "/trigger";
                if (std::ofstream ofs(path); ofs)
                    ofs << trigger;
                return;
            }
        }
        throw std::runtime_error("LED not found: " + led_label);
    }

    void shutdown() override
    {
        /* Turn off all LEDs */
        for (auto& led : leds_) {
            auto path = led.sysfs_path + "/brightness";
            if (std::ofstream ofs(path); ofs)
                ofs << "0";
        }
        initialized_ = false;
    }

    std::string name() const override { return "LedController"; }

private:
    std::vector<LedEntry> leds_;
};

/* -------------------------------------------------------
 * Thermal monitor (uses DT thermal-zones)
 * ------------------------------------------------------- */
class ThermalMonitor : public Peripheral {
public:
    struct Zone {
        std::string name;
        std::string sysfs_path;
    };

    void initialize() override
    {
        namespace fs = std::filesystem;
        const fs::path tz_base = "/sys/class/thermal";

        if (!fs::exists(tz_base)) {
            std::cerr << "[ThermalMonitor] thermal sysfs not available\n";
            return;
        }

        for (auto& entry : fs::directory_iterator(tz_base)) {
            if (entry.path().filename().string().starts_with("thermal_zone")) {
                std::string tz_name;
                std::ifstream type_f(entry.path().string() + "/type");
                if (type_f) std::getline(type_f, tz_name);
                zones_.push_back({tz_name, entry.path().string()});
            }
        }

        initialized_ = true;
        std::cout << std::format("[{}] Found {} thermal zone(s)\n",
                                  name(), zones_.size());
    }

    /* Read temperature in milli-Celsius from a named zone */
    std::optional<int> readTemp(const std::string& zone_name) const
    {
        for (auto& z : zones_) {
            if (z.name == zone_name) {
                std::ifstream temp_f(z.sysfs_path + "/temp");
                int milliC;
                if (temp_f >> milliC)
                    return milliC;
            }
        }
        return std::nullopt;
    }

    void shutdown() override { initialized_ = false; }
    std::string name() const override { return "ThermalMonitor"; }

private:
    std::vector<Zone> zones_;
};

/* -------------------------------------------------------
 * BSP Manager — owns all peripherals
 * ------------------------------------------------------- */
class BspManager {
public:
    BspManager()
    {
        peripherals_.push_back(std::make_unique<LedController>());
        peripherals_.push_back(std::make_unique<ThermalMonitor>());
    }

    void start()
    {
        std::cout << "=== BSP Initialization ===\n";
        for (auto& p : peripherals_) {
            try {
                p->initialize();
            } catch (const std::exception& e) {
                std::cerr << "[WARN] " << p->name()
                          << " init failed: " << e.what() << "\n";
            }
        }
        std::cout << "=== BSP Ready ===\n\n";
    }

    ~BspManager()
    {
        for (auto& p : peripherals_) {
            try { p->shutdown(); }
            catch (...) {}
        }
    }

    template<typename T>
    T* get()
    {
        for (auto& p : peripherals_) {
            if (auto* found = dynamic_cast<T*>(p.get()))
                return found;
        }
        return nullptr;
    }

private:
    std::vector<std::unique_ptr<Peripheral>> peripherals_;
};

} // namespace bsp

int main()
{
    bsp::BspManager bsp;
    bsp.start();

    /* Configure LEDs */
    if (auto* leds = bsp.get<bsp::LedController>()) {
        try {
            leds->setTrigger("heartbeat", "heartbeat");
            leds->setTrigger("activity",  "mmc0");
        } catch (const std::exception& e) {
            std::cerr << "LED config: " << e.what() << "\n";
        }
    }

    /* Monitor temperature in a loop */
    auto* thermal = bsp.get<bsp::ThermalMonitor>();
    for (int i = 0; i < 5; ++i) {
        if (thermal) {
            if (auto t = thermal->readTemp("cpu-thermal")) {
                std::cout << std::format("CPU temp: {:.1f} °C\n",
                                          *t / 1000.0);
            }
        }
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }

    return EXIT_SUCCESS;
}
```

---

## Rust Programming with Device Trees

### Buildroot Rust Package Setup

```makefile
# package/myboard-dt-tool/myboard-dt-tool.mk
# Buildroot package for a Rust DT inspection tool.

MYBOARD_DT_TOOL_VERSION       = 0.1.0
MYBOARD_DT_TOOL_SITE          = $(BR2_EXTERNAL_MYBOARD_BSP_PATH)/package/myboard-dt-tool/src
MYBOARD_DT_TOOL_SITE_METHOD   = local
MYBOARD_DT_TOOL_CARGO_INSTALL = YES

$(eval $(cargo-package))
```

### `Cargo.toml`

```toml
[package]
name    = "myboard-dt-tool"
version = "0.1.0"
edition = "2021"

[dependencies]
# fdt = lightweight no_std FDT parser
fdt         = "0.1"
# gpio-cdev  = libgpiod Rust bindings
gpio-cdev   = "0.6"
# anyhow for ergonomic error handling
anyhow      = "1"
# libc for raw file I/O
libc        = "0.2"
```

### Main Rust Source: `src/main.rs`

```rust
//! myboard-dt-tool — Rust BSP utility for MyBoard
//!
//! Reads the live Device Tree, reports board identity, thermal state,
//! and blinks a GPIO LED — demonstrating DT + GPIO access from Rust.

use anyhow::{Context, Result};
use std::fs;
use std::io::Read;
use std::path::Path;
use std::thread;
use std::time::Duration;

// -----------------------------------------------------------------------
// Device Tree reader using the `fdt` crate
// -----------------------------------------------------------------------

fn load_fdt(path: &str) -> Result<Vec<u8>> {
    let mut file = fs::File::open(path)
        .with_context(|| format!("Opening FDT at {path}"))?;
    let mut buf = Vec::new();
    file.read_to_end(&mut buf)?;
    Ok(buf)
}

fn print_board_info(blob: &[u8]) -> Result<()> {
    let fdt = fdt::Fdt::new(blob)
        .map_err(|e| anyhow::anyhow!("FDT parse error: {:?}", e))?;

    println!("=== Board Information ===");
    println!("  Model      : {}", fdt.root().model());
    println!("  Compatible : {}",
        fdt.root().compatible().all().collect::<Vec<_>>().join(", "));

    // Print chosen / bootargs
    if let Some(chosen) = fdt.chosen() {
        if let Some(args) = chosen.bootargs() {
            println!("  Bootargs   : {}", args);
        }
        if let Some(stdout) = chosen.stdout() {
            println!("  stdout     : {}", stdout.name);
        }
    }

    // Print memory regions
    println!("\n  Memory regions:");
    for region in fdt.memory().regions() {
        println!("    0x{:08x} .. +0x{:08x} ({} MiB)",
            region.starting_address as usize,
            region.size.unwrap_or(0),
            region.size.unwrap_or(0) >> 20);
    }

    // Find all I2C nodes
    println!("\n  I2C controllers:");
    for node in fdt.all_nodes() {
        if node.compatible()
               .map(|c| c.all().any(|s| s.contains("i2c")))
               .unwrap_or(false)
        {
            println!("    {}", node.name);
            // List children (devices on the bus)
            for child in node.children() {
                println!("      child: {}", child.name);
            }
        }
    }

    Ok(())
}

// -----------------------------------------------------------------------
// Thermal zone reader (sysfs)
// -----------------------------------------------------------------------

fn read_thermal_zones() -> Vec<(String, i32)> {
    let base = Path::new("/sys/class/thermal");
    let mut results = Vec::new();

    let Ok(entries) = fs::read_dir(base) else { return results; };

    for entry in entries.flatten() {
        let name = entry.file_name().to_string_lossy().to_string();
        if !name.starts_with("thermal_zone") {
            continue;
        }

        let tz_path = entry.path();
        let type_str = fs::read_to_string(tz_path.join("type"))
            .unwrap_or_default()
            .trim()
            .to_string();

        if let Ok(temp_str) = fs::read_to_string(tz_path.join("temp")) {
            if let Ok(temp_mc) = temp_str.trim().parse::<i32>() {
                results.push((type_str, temp_mc));
            }
        }
    }

    results
}

// -----------------------------------------------------------------------
// GPIO LED blinker using gpio-cdev
// -----------------------------------------------------------------------

fn blink_led(chip_name: &str, line_offset: u32, count: u32) -> Result<()> {
    use gpio_cdev::{Chip, LineRequestFlags};

    let mut chip = Chip::new(format!("/dev/{chip_name}"))
        .with_context(|| format!("Opening GPIO chip {chip_name}"))?;

    let handle = chip
        .get_line(line_offset)
        .context("Getting GPIO line")?
        .request(LineRequestFlags::OUTPUT, 0, "myboard-dt-tool")
        .context("Requesting GPIO line as output")?;

    println!("\n=== LED Blink Test (GPIO {chip_name}:{line_offset}) ===");

    for i in 0..count {
        handle.set_value(1)?;
        thread::sleep(Duration::from_millis(200));
        handle.set_value(0)?;
        thread::sleep(Duration::from_millis(200));
        print!("  blink {}/{}\r", i + 1, count);
        use std::io::Write;
        std::io::stdout().flush().ok();
    }
    println!("\n  Done.");

    Ok(())
}

// -----------------------------------------------------------------------
// Main
// -----------------------------------------------------------------------

fn main() -> Result<()> {
    // 1. Parse and display FDT
    let blob = load_fdt("/sys/firmware/fdt")?;
    print_board_info(&blob)?;

    // 2. Show thermal zones
    println!("\n=== Thermal Zones ===");
    let zones = read_thermal_zones();
    if zones.is_empty() {
        println!("  (no thermal zones found)");
    } else {
        for (zone_type, temp_mc) in &zones {
            println!("  {:20}: {:5.1} °C", zone_type, *temp_mc as f32 / 1000.0);
        }
    }

    // 3. Blink LED on PH10 (chip0, line 234 = 7*32+10)
    //    This will fail gracefully if not running on the target
    if let Err(e) = blink_led("gpiochip0", 234, 5) {
        eprintln!("  [GPIO skip] {e}");
    }

    Ok(())
}
```

### Rust Kernel Module Stub (for `rust-out-of-tree` usage)

```rust
// myboard_sensor_rust/src/lib.rs
// Skeleton for an out-of-tree Rust kernel module using the
// kernel crate (available from linux/rust/ since Linux 6.1).
// Build with: make -C $KERNEL_DIR M=$PWD modules

#![no_std]
#![feature(allocator_api)]

use kernel::prelude::*;
use kernel::{
    module,
    of,
    platform,
    device::Device,
    io_mem::IoMem,
};

module! {
    type: MyBoardSensorModule,
    name: "myboard_sensor_rust",
    author: "MyCompany BSP Team",
    description: "MyBoard sensor driver in Rust",
    license: "GPL",
}

struct MyBoardSensorModule;

impl kernel::Module for MyBoardSensorModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("MyBoard Sensor (Rust) loaded\n");
        Ok(Self)
    }
}

impl Drop for MyBoardSensorModule {
    fn drop(&mut self) {
        pr_info!("MyBoard Sensor (Rust) unloaded\n");
    }
}

// Device Tree match table
kernel::of_device_table!(
    OF_TABLE,
    MODULE_OF_TABLE,
    <MyBoardSensorDriver as platform::Driver>::IdInfo,
    [
        (of::DeviceId::new("mycompany,myboard-sensor"), ()),
    ]
);

struct MyBoardSensorDriver;

impl platform::Driver for MyBoardSensorDriver {
    type IdInfo = ();
    const ID_TABLE: platform::IdTable<()> = &OF_TABLE;

    fn probe(pdev: &mut platform::Device, _id_info: Option<&()>) -> Result<Pin<Box<Self>>> {
        dev_info!(pdev.as_ref(), "Rust driver probing...\n");
        Ok(Pin::from(Box::try_new(Self)?))
    }
}
```

---

## BSP Package Integration

### Package Makefile: `myboard-tools.mk`

```makefile
################################################################################
# myboard-tools — Board-specific userspace utilities
# Installs: board-init, dt-inspect, factory-test
################################################################################

MYBOARD_TOOLS_VERSION = 1.2.0
MYBOARD_TOOLS_SITE    = $(BR2_EXTERNAL_MYBOARD_BSP_PATH)/package/myboard-tools/src
MYBOARD_TOOLS_SITE_METHOD = local

MYBOARD_TOOLS_DEPENDENCIES = libgpiod libfdt util-linux

MYBOARD_TOOLS_CFLAGS  = $(TARGET_CFLAGS) -Wall -Wextra -O2
MYBOARD_TOOLS_LDFLAGS = $(TARGET_LDFLAGS) -lgpiod -lfdt

define MYBOARD_TOOLS_BUILD_CMDS
    $(TARGET_CC) $(MYBOARD_TOOLS_CFLAGS) \
        $(@D)/board_gpio_test.c \
        $(MYBOARD_TOOLS_LDFLAGS) \
        -o $(@D)/board-gpio-test

    $(TARGET_CC) $(MYBOARD_TOOLS_CFLAGS) \
        $(@D)/dt_read_props.c \
        $(MYBOARD_TOOLS_LDFLAGS) \
        -o $(@D)/dt-inspect
endef

define MYBOARD_TOOLS_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/board-gpio-test \
        $(TARGET_DIR)/usr/bin/board-gpio-test
    $(INSTALL) -D -m 0755 $(@D)/dt-inspect \
        $(TARGET_DIR)/usr/bin/dt-inspect
    $(INSTALL) -D -m 0755 \
        $(BR2_EXTERNAL_MYBOARD_BSP_PATH)/board/myboard/apply-overlay.sh \
        $(TARGET_DIR)/usr/sbin/apply-overlay
endef

$(eval $(generic-package))
```

### Post-Build Script: `post-build.sh`

```bash
#!/bin/bash
# post-build.sh — Runs after all packages are built,
# before the filesystem image is created.

set -euo pipefail

TARGET_DIR="$1"
BOARD_DIR="$(dirname "$0")"

echo "[post-build] Configuring MyBoard rootfs..."

# Install DTB overlays into /lib/firmware/
OVERLAY_DIR="${TARGET_DIR}/lib/firmware"
mkdir -p "${OVERLAY_DIR}"
cp "${BOARD_DIR}/dts/"*.dtbo "${OVERLAY_DIR}/" 2>/dev/null || true

# Set hostname
echo "myboard" > "${TARGET_DIR}/etc/hostname"

# Configure serial console in inittab
if [ -f "${TARGET_DIR}/etc/inittab" ]; then
    # Add ttyS0 if not present
    grep -q "ttyS0" "${TARGET_DIR}/etc/inittab" || \
        echo "ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100" \
        >> "${TARGET_DIR}/etc/inittab"
fi

# Install udev rules for board devices
cat > "${TARGET_DIR}/etc/udev/rules.d/99-myboard.rules" << 'EOF'
# MyBoard custom udev rules
SUBSYSTEM=="gpio", MODE="0660", GROUP="gpio"
SUBSYSTEM=="i2c-dev", MODE="0660", GROUP="i2c"
SUBSYSTEM=="spidev", MODE="0660", GROUP="spi"
EOF

echo "[post-build] Done."
```

### Post-Image Script: `post-image.sh`

```bash
#!/bin/bash
# post-image.sh — Assembles the final SD card image.

set -euo pipefail

BOARD_DIR="$(dirname "$0")"
BINARIES_DIR="${BINARIES_DIR:-output/images}"
GENIMAGE_CFG="${BOARD_DIR}/genimage.cfg"
GENIMAGE_TMP="${BUILD_DIR}/genimage.tmp"

# Use genimage to create the SD card layout
rm -rf "${GENIMAGE_TMP}"
genimage \
    --rootpath "${TARGET_DIR}" \
    --tmppath  "${GENIMAGE_TMP}" \
    --inputpath "${BINARIES_DIR}" \
    --outputpath "${BINARIES_DIR}" \
    --config "${GENIMAGE_CFG}"

echo "[post-image] SD card image: ${BINARIES_DIR}/sdcard.img"
```

### `genimage.cfg` — SD Card Layout

```
image boot.vfat {
    vfat {
        files = {
            "Image",
            "myboard.dtb",
            "myboard-overlay-i2s.dtbo",
            "myboard-overlay-lcd.dtbo"
        }
    }
    size = 32M
}

image sdcard.img {
    hdimage {}

    partition u-boot-spl {
        in-partition-table = "no"
        image = "u-boot-spl.bin"
        offset = 8K
    }

    partition u-boot {
        in-partition-table = "no"
        image = "u-boot.itb"
        offset = 40K
    }

    partition boot {
        partition-type = 0x0C
        bootable = "true"
        image = "boot.vfat"
    }

    partition rootfs {
        partition-type = 0x83
        image = "rootfs.ext4"
    }
}
```

---

## Testing and Debugging

### DT Debugging Commands

```bash
# Dump the live device tree in human-readable form
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# Check a specific node
cat /sys/firmware/devicetree/base/soc/serial@1c28000/status

# List all compatible strings on the system
find /sys/firmware/devicetree/base -name "compatible" | \
    xargs -I{} sh -c 'echo "{}:"; cat "{}" | tr "\0" "\n"; echo'

# Verify a DTB file offline (before booting)
dtc -I dtb -O dts -o /tmp/check.dts myboard.dtb && echo "OK"

# Check for DT overlay application status
ls /sys/kernel/config/device-tree/overlays/
cat /sys/kernel/config/device-tree/overlays/myboard-i2s/status

# Find which driver claimed a device
cat /sys/bus/platform/devices/1c28000.serial/driver/module/srcversion
```

### ASCII: Build and Boot Validation Flow

```
  Developer workstation              Target board
  =====================              ============

  $ make myboard_defconfig
  $ make                                |
  |                                     |
  output/images/                        |
  |- Image          -----------------> TFTP / SD card
  |- myboard.dtb    ----------------->
  |- rootfs.ext4    ----------------->
                                        |
                          U-Boot loads Image + myboard.dtb
                                        |
                          Kernel boots, /proc/device-tree
                          populated from myboard.dtb
                                        |
                          [ OK ] i2c 1c2ac00.i2c: adapter 0
                          [ OK ] tmp102 0-0048: 24750 mC
                          [ OK ] myboard-sensor-rust loaded
                                        |
                          Login: root
                          # dt-inspect
                          # board-gpio-test
```

### Common Pitfalls

```
ASCII: Troubleshooting Decision Tree

  Device not probed by kernel?
         |
         +---> Check `status = "okay"` in DTS
         |
         +---> Check compatible string matches driver
         |         cat /sys/bus/*/devices/*/of_node/compatible
         |
         +---> Check clocks are enabled
         |         cat /sys/kernel/debug/clk/clk_summary
         |
         +---> Check pinmux conflicts
         |         cat /sys/kernel/debug/pinctrl/*/pinmux-pins
         |
         +---> Enable dynamic debug
                   echo "file drivers/i2c/* +p" > \
                       /sys/kernel/debug/dynamic_debug/control
```

---

## Summary

Device Tree and BSP management in Buildroot is the discipline that bridges generic Linux infrastructure and specific hardware reality. The key concepts and their relationships are:

**Device Tree files** (`.dts`/`.dtsi`) describe hardware declaratively: CPUs, memory maps, peripheral addresses, interrupt lines, clock trees, GPIO assignments, and pin-mux configurations. The hierarchical include system (`dtsi`) promotes reuse — SoC-level details live in one file, board-specific node enablement and parameter overrides live in another. Overlays (`.dtbo`) allow runtime reconfiguration without recompiling the base blob.

**`BR2_EXTERNAL`** is Buildroot's designated mechanism for keeping board-specific material out of the upstream source tree. A well-structured external tree holds DTS sources under `board/<boardname>/dts/`, defconfigs under `configs/`, packages under `package/`, and kernel config fragments under `linux/`. The `external.desc`, `external.mk`, and `Config.in` files wire the layer into Buildroot's build and Kconfig systems.

**`BR2_LINUX_KERNEL_DTS_SUPPORT`** is the Kconfig symbol that activates DTB compilation as part of the kernel build. For mainline-supported boards, `BR2_LINUX_KERNEL_INTREE_DTS_NAME` selects the in-tree file. For proprietary or pre-upstream hardware, `BR2_LINUX_KERNEL_CUSTOM_DTS_PATH` points to files in the external tree; Buildroot copies them into the kernel source before invoking `make dtbs`.

**C drivers** interact with DT through the kernel's Open Firmware (`of_*`) API — `of_property_read_u32`, `devm_gpiod_get`, `devm_clk_get` — and through matching tables (`of_device_id[]`). Userspace utilities use `libfdt` to parse the live `/sys/firmware/fdt` blob directly.

**C++ BSP code** wraps these primitives in RAII classes, using `std::optional` for fallible property reads, range-based algorithms for node enumeration, and ownership semantics to guarantee clean shutdown. This pattern is well suited for system daemons and factory test frameworks.

**Rust** brings memory-safety guarantees to both userspace (via the `fdt` crate and `gpio-cdev`) and kernel space (via the in-tree `kernel` crate available since Linux 6.1). The `cargo-package` Buildroot infrastructure handles cross-compilation transparently.

**Packaging** (`.mk` files, `post-build.sh`, `post-image.sh`, `genimage.cfg`) ties everything together: overlays land in `/lib/firmware/`, udev rules grant the right permissions, and `genimage` assembles a bootable SD card image in a single `make` invocation.

```
ASCII: Complete BSP Dependency Graph

  BR2_EXTERNAL/
       |
       +-- board/myboard/dts/ -----> BR2_LINUX_KERNEL_CUSTOM_DTS_PATH
       |        |                              |
       |        v                              v
       |   mysoc.dtsi                   make dtbs
       |   myboard.dts  ─────────────> myboard.dtb ──> output/images/
       |   *.dtbo       ─────────────> /lib/firmware/  output/images/
       |
       +-- configs/myboard_defconfig --> .config
       |
       +-- linux/linux.config -------> CONFIG_OF=y, CONFIG_OF_OVERLAY=y
       |
       +-- package/myboard-tools/
       |        |
       |        v
       |   dt_read_props.c  (C)
       |   dt_parser.cpp    (C++)
       |   myboard-dt-tool/ (Rust)
       |        |
       |        v
       |   output/target/usr/bin/dt-inspect
       |   output/target/usr/bin/board-gpio-test
       |
       +-- board/myboard/post-build.sh --> rootfs configuration
       +-- board/myboard/post-image.sh --> sdcard.img via genimage
```

A mature BSP encapsulates all board knowledge in `BR2_EXTERNAL`, leaving the Buildroot core and kernel tree unmodified. This clean separation makes it straightforward to upgrade the kernel, Buildroot version, or toolchain independently of the board-specific layer — and to share the BSP as a standalone repository that teams can drop into any Buildroot workspace with a single `BR2_EXTERNAL` variable.

---

*Document: Buildroot Series — Chapter 13: Device Tree & Board Support Packages*
*Covers: Buildroot ≥ 2024.02, Linux kernel ≥ 6.6, DTS v1 specification*