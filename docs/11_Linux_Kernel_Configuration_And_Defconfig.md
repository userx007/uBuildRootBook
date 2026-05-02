# 11. Linux Kernel Configuration & Custom defconfig

**Architecture & Flow** — ASCII diagrams showing the full layered pipeline from upstream defconfig through fragments to the final `.config` and build output.

**`BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE`** — directory layout conventions, how to derive a starting defconfig from an upstream arch config, and what to put in `linux.config`.

**Config Fragments** — format, referencing multiple fragments, how Buildroot calls `merge_config.sh` internally, conditional fragment application, and how to verify symbols landed correctly.

**`make linux-menuconfig`** — TUI navigation in ASCII, key shortcuts table, the search interface, and the critical distinction between the full `.config` vs. the minimal defconfig that must be committed.

**Saving defconfigs** — `linux-savedefconfig`, `linux-update-defconfig`, round-trip validation, and managing multiple board variants with shared base + fragments.

**In-tree vs. Out-of-tree modules** — trade-off table (built-in vs. `=m`), Kconfig anatomy, the `obj-$(CONFIG_FOO)` Makefile pattern, and the full Buildroot `kernel-module` package structure.

**Code examples:**
- **C** — complete in-tree platform driver with MMIO, `cdev`, `file_operations`, DT matching, and a userspace test tool
- **C++** — a kernel `.config` parser and rule-checker for CI pipelines, plus a `KernelModule` class for sysfs interaction
- **Rust** — a Rust kernel module skeleton using the official kernel Rust bindings, a host tool generating fragments from TOML specs, and a Buildroot build script


> **Buildroot Series — Chapter 11**
> Topics: `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE`, kernel fragments (`kernel_config_frag`),
> `make linux-menuconfig`, saving defconfigs, in-tree vs. out-of-tree driver modules.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Buildroot Kernel Configuration Architecture](#2-buildroot-kernel-configuration-architecture)
3. [BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE](#3-br2_linux_kernel_custom_config_file)
4. [Kernel Config Fragments](#4-kernel-config-fragments-kernel_config_frag)
5. [make linux-menuconfig](#5-make-linux-menuconfig)
6. [Saving and Managing defconfigs](#6-saving-and-managing-defconfigs)
7. [In-Tree Driver Modules](#7-in-tree-driver-modules)
8. [Out-of-Tree Driver Modules](#8-out-of-tree-driver-modules)
9. [C Code Examples](#9-c-code-examples)
10. [C++ Code Examples](#10-c-code-examples-1)
11. [Rust Code Examples](#11-rust-code-examples)
12. [Summary](#12-summary)

---

## 1. Overview

The Linux kernel is the most critical component in an embedded system built with Buildroot. Getting the kernel configuration right determines which hardware is supported, which security features are active, how much memory is consumed, and how fast the system boots.

Buildroot provides a well-structured workflow for managing the kernel `.config`:

```
  ┌────────────────────────────────────────────────────────────────┐
  │                   Buildroot Kernel Config Flow                 │
  │                                                                │
  │  Board defconfig       Kernel defconfig      Kernel fragments  │
  │  (configs/myboard_    (board/myboard/        (board/myboard/   │
  │   defconfig)           linux.config)          linux.frag)      │
  │        │                    │                      │           │
  │        ▼                    ▼                      ▼           │
  │  ┌──────────────────────────────────────────────────────────┐  │
  │  │               BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE        │  │
  │  │               BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES     │  │
  │  └─────────────────────────────┬────────────────────────────┘  │
  │                                │                               │
  │                                ▼                               │
  │                   ┌────────────────────────┐                   │
  │                   │  merge_config.sh +     │                   │
  │                   │  scripts/kconfig/      │                   │
  │                   └───────────┬────────────┘                   │
  │                               │                                │
  │                               ▼                                │
  │                   ┌────────────────────────┐                   │
  │                   │   output/build/        │                   │
  │                   │   linux-x.y.z/.config  │                   │
  │                   └───────────┬────────────┘                   │
  │                               │                                │
  │                               ▼                                │
  │                       make linux-image                         │
  └────────────────────────────────────────────────────────────────┘
```

This chapter covers every step of that pipeline in depth.

---

## 2. Buildroot Kernel Configuration Architecture

Before diving into specifics, it is important to understand how Buildroot thinks about the kernel configuration at a high level.

### 2.1 Configuration Layers

Buildroot uses a layered approach:

```
  Layer 0: Upstream kernel default (arch/arm/configs/…)
      │
      ▼
  Layer 1: Board-specific defconfig   ← BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE
      │
      ▼
  Layer 2: Config fragments           ← BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES
      │
      ▼
  Layer 3: Interactive overrides      ← make linux-menuconfig (not persisted automatically)
      │
      ▼
  Final:   output/build/linux-x.y.z/.config
```

Each lower layer *adds to or overrides* the layer above it. Fragments only set the symbols they mention; everything else is inherited.

### 2.2 Key Buildroot Variables

| Variable | Purpose |
|---|---|
| `BR2_LINUX_KERNEL` | Enable kernel build |
| `BR2_LINUX_KERNEL_CUSTOM_VERSION` | Use a specific upstream version |
| `BR2_LINUX_KERNEL_CUSTOM_TARBALL` | Use a local tarball |
| `BR2_LINUX_KERNEL_CUSTOM_GIT` | Use a Git repository |
| `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE` | Path to your defconfig |
| `BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES` | Space-separated list of fragment files |
| `BR2_LINUX_KERNEL_DEFCONFIG` | Use an in-tree defconfig by name |
| `BR2_LINUX_KERNEL_IMAGE_TARGET` | Image type: `zImage`, `Image.gz`, `uImage`… |

### 2.3 Buildroot's `Config.in` for the Kernel

Inside `linux/Config.in`, Buildroot exposes all of the above. A minimal kernel section in your board's Buildroot `defconfig` looks like:

```ini
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="6.6.28"
BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG=y
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="board/myboard/linux.config"
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES="board/myboard/linux-usb.frag board/myboard/linux-debug.frag"
BR2_LINUX_KERNEL_IMAGE_TARGET="zImage"
BR2_LINUX_KERNEL_DTS="myboard"
BR2_LINUX_KERNEL_INTREE_DTS_NAME="arm/myboard"
```

---

## 3. `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE`

This variable points Buildroot to your board's kernel defconfig file. It is the foundation of your kernel configuration.

### 3.1 Directory Layout

A well-organized Buildroot project for a board called `myboard` looks like:

```
  project-root/
  ├── board/
  │   └── myboard/
  │       ├── linux.config            ← main kernel defconfig
  │       ├── linux-usb.frag          ← USB fragment
  │       ├── linux-debug.frag        ← debug options fragment
  │       ├── patches/
  │       │   └── linux/
  │       │       └── 0001-fix.patch
  │       └── rootfs-overlay/
  │           └── etc/
  │               └── init.d/
  │                   └── S99myapp
  ├── configs/
  │   └── myboard_defconfig           ← Buildroot board defconfig
  ├── package/
  │   └── mydriver/                   ← out-of-tree module package
  └── output/                         ← generated, never commit
```

### 3.2 Setting the Variable in `menuconfig`

Navigate:

```
  Buildroot menuconfig
  └── Kernel
      └── Linux Kernel
          ├── [*] Linux Kernel
          ├── Kernel configuration
          │     (X) Using a custom (def)config file
          └── Configuration file path
                → board/myboard/linux.config
```

### 3.3 Creating a Starting defconfig

You should not write a kernel defconfig from scratch. Instead start from the closest upstream defconfig for your architecture:

```bash
# Inside the Linux kernel source tree (Buildroot fetches it to output/build/linux-x.y.z/)
cd output/build/linux-6.6.28/

# List available defconfigs for ARM
ls arch/arm/configs/

# Start from a known-good one
make ARCH=arm versatile_defconfig

# Immediately save it as your board defconfig
make ARCH=arm savedefconfig
cp defconfig ../../../board/myboard/linux.config
```

### 3.4 What Goes into `linux.config`

The saved defconfig only stores symbols that differ from their default values. This keeps the file small and maintainable. A typical board defconfig excerpt:

```
# Board: myboard — ARM Cortex-A9
CONFIG_ARCH_MYMACHINE=y
CONFIG_MACH_MYBOARD=y

# CPU
CONFIG_CPU_V7=y
CONFIG_ARM_LPAE=y

# Timers
CONFIG_HAVE_ARM_ARCH_TIMER=y

# MMC/SD
CONFIG_MMC=y
CONFIG_MMC_SDHCI=y
CONFIG_MMC_SDHCI_PLTFM=y
CONFIG_MMC_SDHCI_OF_ARASAN=y

# Networking
CONFIG_NET=y
CONFIG_ETHERNET=y
CONFIG_NET_VENDOR_SMSC=y
CONFIG_SMSC911X=y

# Serial
CONFIG_SERIAL_AMBA_PL011=y
CONFIG_SERIAL_AMBA_PL011_CONSOLE=y

# Filesystems
CONFIG_EXT4_FS=y
CONFIG_TMPFS=y
CONFIG_PROC_FS=y
CONFIG_SYSFS=y

# Disable unused (saves ~300 KB)
# CONFIG_SOUND is not set
# CONFIG_USB_GADGET is not set
# CONFIG_WIRELESS is not set
```

---

## 4. Kernel Config Fragments (`kernel_config_frag`)

Config fragments allow you to overlay specific `CONFIG_` changes on top of your base defconfig, without editing it. This is the recommended approach for board variants, debug builds, and feature toggles.

### 4.1 Fragment File Format

A fragment is a plain text file with only the symbols you want to change. Symbols not mentioned are untouched.

```
# board/myboard/linux-usb.frag
# Enable USB host and CDC-ACM for USB-to-serial adapters
CONFIG_USB=y
CONFIG_USB_SUPPORT=y
CONFIG_USB_XHCI_HCD=y
CONFIG_USB_EHCI_HCD=y
CONFIG_USB_ACM=y
CONFIG_USB_SERIAL=y
CONFIG_USB_SERIAL_CH341=y
```

```
# board/myboard/linux-debug.frag
# Enable kernel debug features for development builds
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_MAGIC_SYSRQ=y
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_DYNAMIC_FTRACE=y
# CONFIG_OPTIMIZE_INLINING is not set
```

### 4.2 Referencing Fragments in the Buildroot defconfig

```ini
# configs/myboard_defconfig (excerpt)
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES="board/myboard/linux-usb.frag board/myboard/linux-debug.frag"
```

Multiple fragments are space-separated. They are applied left-to-right; later fragments win.

### 4.3 How Buildroot Merges Fragments

Internally Buildroot calls the kernel's own `scripts/kconfig/merge_config.sh`:

```
  base defconfig            fragment 1              fragment 2
  ┌──────────────┐       ┌──────────────┐       ┌───────────────┐
  │CONFIG_NET=y  │  +    │CONFIG_USB=y  │  +    │CONFIG_KGDB=y  │
  │CONFIG_MMC=y  │       │CONFIG_XHCI=y │       │CONFIG_FTRACE=y│
  └──────────────┘       └──────────────┘       └───────────────┘
           │                     │                     │
           └─────────────────────┴─────────────────────┘
                                 │
                                 ▼
                    merge_config.sh (kernel script)
                                 │
                                 ▼
                    output/build/linux-6.6.28/.config
                    (fully resolved, all symbols present)
```

### 4.4 Conditional Fragment Application

You can use Buildroot package logic to conditionally add fragments. Create a helper in `board/myboard/post-build.sh` or use Buildroot's `BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES` via `Config.in` conditionals:

```ini
# In your board's Config.in
config BR2_MYBOARD_DEBUG_KERNEL
    bool "Enable kernel debug features"
    default n
    help
      Adds KGDB, ftrace, and debug info to the kernel.
```

Then in your `myboard.mk` or top-level `Makefile`:

```makefile
ifeq ($(BR2_MYBOARD_DEBUG_KERNEL),y)
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES += board/myboard/linux-debug.frag
endif
```

### 4.5 Checking Fragment Application

After running `make linux-configure` (or `make`), verify your symbols were applied:

```bash
grep CONFIG_KGDB output/build/linux-6.6.28/.config
# Expected: CONFIG_KGDB=y

# Check for conflicts (symbols that could not be satisfied)
make linux-configure 2>&1 | grep "^WARNING"
```

---

## 5. `make linux-menuconfig`

`make linux-menuconfig` opens the kernel's ncurses-based configuration interface within the Buildroot context, using the current merged `.config` as a starting point.

### 5.1 Invoking It

```bash
# From Buildroot root — kernel source must already be extracted
make linux-menuconfig
```

Buildroot ensures the correct cross-compiler, architecture (`ARCH`), and cross-compile prefix (`CROSS_COMPILE`) are set before launching menuconfig.

### 5.2 TUI Navigation (ASCII Overview)

```
  ┌────────────────────── Linux Kernel Configuration ──────────────────────────┐
  │  Arrow keys navigate. <Enter> selects. <?> for help. <Esc><Esc> to exit.   │
  ├────────────────────────────────────────────────────────────────────────────┤
  │                                                                            │
  │    General setup  --->                                                     │
  │    [*] 64-bit kernel                                                       │
  │        Processor type and features  --->                                   │
  │        Power management and ACPI options  --->                             │
  │    [*] Networking support  --->                                            │
  │        Device Drivers  --->                                                │
  │        File systems  --->                                                  │
  │        Kernel hacking  --->                                                │
  │        Security options  --->                                              │
  │                                                                            │
  │  Legend:  [*] built-in   [ ] excluded   <M> module   < > module capable    │
  ├────────────────────────────────────────────────────────────────────────────┤
  │         <Select>    < Exit >    < Help >    < Save >    < Load >           │
  └────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 Key Shortcuts

| Key | Action |
|---|---|
| `↑` / `↓` | Move between entries |
| `→` / `Enter` | Enter submenu |
| `←` / `Esc Esc` | Go back / exit |
| `Y` | Enable built-in (`=y`) |
| `N` | Disable (`# not set`) |
| `M` | Enable as module (`=m`) |
| `/` | Search for CONFIG symbol |
| `?` | Help for current symbol |
| `S` | Save `.config` |
| `Q` | Quit (prompts to save) |

### 5.4 Searching for Symbols

Press `/` and type a symbol name:

```
  ┌──────────────────── Search Results ───────────────────────┐
  │  Symbol: USB_ACM [=m]                                     │
  │  Type  : tristate                                         │
  │  Prompt: USB Modem (CDC ACM) support                      │
  │    Location:                                              │
  │      -> Device Drivers                                    │
  │        -> USB support (USB_SUPPORT [=y])                  │
  │          -> USB Serial Converter support (USB_SERIAL [=y])│
  │  Depends on: USB_SUPPORT && USB [=y]                      │
  │  Selected by: (nothing)                                   │
  └───────────────────────────────────────────────────────────┘
```

### 5.5 Differences Between linux-menuconfig and linux-savedefconfig

An important subtlety: `make linux-menuconfig` saves a *full* `.config` (all symbols). This is NOT the same as a minimal defconfig. You must explicitly run `make linux-savedefconfig` afterwards to produce the minimal file that you commit to your board directory.

```
  make linux-menuconfig
         │
         │ (you make changes interactively)
         │
         ▼
  output/build/linux-x.y.z/.config   ← full, all symbols
         │
         │ make linux-savedefconfig
         ▼
  output/build/linux-x.y.z/defconfig ← minimal, only non-defaults
         │
         │ cp …/defconfig board/myboard/linux.config
         ▼
  board/myboard/linux.config          ← commit this!
```

---

## 6. Saving and Managing defconfigs

### 6.1 The `linux-savedefconfig` Target

```bash
# After making changes in linux-menuconfig:
make linux-savedefconfig

# The minimal defconfig is saved to:
#   output/build/linux-<version>/defconfig
# Copy it back to your board directory:
cp output/build/linux-6.6.28/defconfig board/myboard/linux.config
```

### 6.2 Buildroot's `BR2_LINUX_KERNEL_SAVEDEFCONFIG` Location Override

You can also use Buildroot's `linux-update-defconfig` target which automatically copies the defconfig to the location specified by `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE`:

```bash
make linux-update-defconfig
# Equivalent to savedefconfig + copy in one step
```

### 6.3 Validating Your defconfig Round-Trips Correctly

Always verify your saved defconfig produces the same `.config` when re-applied:

```bash
# Apply defconfig
make linux-configure

# Save minimal form
make linux-savedefconfig
cp output/build/linux-6.6.28/defconfig /tmp/new.config

# Compare with your board config
diff board/myboard/linux.config /tmp/new.config
# Empty diff means your config is clean and round-trips correctly
```

### 6.4 Maintaining Multiple Board Variants

For product families with a shared base but different options (e.g., `myboard-basic`, `myboard-pro`), use a shared base defconfig + fragments:

```
  board/
  └── myboard/
      ├── linux-base.config          ← shared base (all variants)
      ├── linux-basic.frag           ← minimal variant (no WiFi)
      └── linux-pro.frag             ← pro variant (+WiFi, +BT)

  configs/
  ├── myboard_basic_defconfig        ← uses linux-base.config + linux-basic.frag
  └── myboard_pro_defconfig          ← uses linux-base.config + linux-pro.frag
```

### 6.5 Using `check-package` and `check-symbol`

Buildroot provides a helper to sanity-check kernel configs:

```bash
# Check that required symbols are set
support/scripts/check-kernel-config.sh \
    output/build/linux-6.6.28/.config \
    board/myboard/required-symbols.txt
```

`required-symbols.txt` format:

```
CONFIG_NET=y
CONFIG_INET=y
CONFIG_EXT4_FS=y
```

---

## 7. In-Tree Driver Modules

In-tree drivers live inside the Linux kernel source tree. They are built simply by enabling them in the kernel config.

### 7.1 Architecture

```
  Linux kernel source tree (output/build/linux-6.6.28/)
  │
  ├── drivers/
  │   ├── net/
  │   │   ├── ethernet/
  │   │   │   └── smsc/
  │   │   │       ├── smsc911x.c    ← in-tree driver
  │   │   │       └── Makefile
  │   │   └── Kconfig               ← CONFIG_SMSC911X defined here
  │   └── …
  └── arch/
      └── …
```

### 7.2 Enabling In-Tree Modules via Config

In your `linux.config` or a fragment:

```
# Enable SMSC911X as a loadable module
CONFIG_SMSC911X=m

# Enable 8250 UART as built-in (=y, not =m)
CONFIG_SERIAL_8250=y
CONFIG_SERIAL_8250_CONSOLE=y
```

### 7.3 Module vs. Built-in: Trade-offs

```
  CONFIG_FOO=y  (built-in)           CONFIG_FOO=m  (module)
  ┌─────────────────────────┐        ┌─────────────────────────┐
  │ + Available at boot     │        │ + Smaller kernel image  │
  │ + No module filesystem  │        │ + Loaded on demand      │
  │   required              │        │ + Reloadable w/o reboot │
  │ - Larger vmlinuz        │        │ - Needs /lib/modules/…  │
  │ - Wastes RAM if unused  │        │ - Slightly longer init  │
  └─────────────────────────┘        └─────────────────────────┘
```

Rule of thumb: built-in for anything required during early boot (storage controllers, console); module for anything optional (USB audio, WiFi).

### 7.4 In-Tree Kconfig Anatomy

Understanding the Kconfig for an in-tree driver helps you write your own. Here is a representative example from `drivers/net/ethernet/smsc/Kconfig`:

```kconfig
config SMSC911X
    tristate "SMSC LAN911x/LAN921x families embedded ethernet"
    depends on ARM || ARM64 || MIPS || COMPILE_TEST
    select MII
    select CRC32
    select PHYLIB
    help
      This is the driver for the SMSC 911x / 921x families of ethernet
      controllers for embedded systems.

      To compile this driver as a module, choose M here. The module
      will be called smsc911x.
```

And its `Makefile`:

```makefile
# drivers/net/ethernet/smsc/Makefile
obj-$(CONFIG_SMSC911X) += smsc911x.o
```

The `obj-$(CONFIG_SMSC911X)` pattern evaluates to `obj-y` (built-in), `obj-m` (module), or nothing (disabled).

---

## 8. Out-of-Tree Driver Modules

Out-of-tree (external) modules live outside the kernel source tree. Buildroot integrates them as packages that hook into the kernel build system using `kernel-module` infrastructure.

### 8.1 Architecture

```
  Buildroot project/
  ├── package/
  │   └── mydriver/
  │       ├── Config.in              ← package configuration
  │       ├── mydriver.mk            ← Buildroot makefile
  │       └── src/                   ← driver source (or fetched remotely)
  │           ├── mydriver.c
  │           └── Makefile
  │
  └── output/
      └── build/
          ├── linux-6.6.28/          ← kernel build dir (headers/Module.symvers)
          └── mydriver-1.0/
              └── mydriver.ko        ← built against kernel headers
```

### 8.2 The External Module Makefile

The driver's `Makefile` is the standard out-of-tree kernel module Makefile:

```makefile
# package/mydriver/src/Makefile

KERNEL_SRC ?= /lib/modules/$(shell uname -r)/build

obj-m := mydriver.o
mydriver-objs := mydriver_core.o mydriver_spi.o

all:
	$(MAKE) -C $(KERNEL_SRC) M=$(PWD) modules

clean:
	$(MAKE) -C $(KERNEL_SRC) M=$(PWD) clean

install:
	$(MAKE) -C $(KERNEL_SRC) M=$(PWD) modules_install
```

### 8.3 The Buildroot Package `.mk` File

```makefile
# package/mydriver/mydriver.mk

MYDRIVER_VERSION = 1.2.0
MYDRIVER_SITE    = https://github.com/example/mydriver/archive
MYDRIVER_SOURCE  = mydriver-$(MYDRIVER_VERSION).tar.gz

# This macro registers the package as a kernel module package
$(eval $(kernel-module))
$(eval $(generic-package))
```

Buildroot's `kernel-module` infrastructure automatically:
- Passes `KERNEL_SRC`, `ARCH`, and `CROSS_COMPILE` to the module Makefile
- Ensures the module is built after the kernel
- Installs the `.ko` file into the target rootfs under `/lib/modules/<kver>/extra/`
- Runs `depmod` during post-build

### 8.4 `Config.in` for the Out-of-Tree Module

```kconfig
# package/mydriver/Config.in

config BR2_PACKAGE_MYDRIVER
    tristate "mydriver"
    depends on BR2_LINUX_KERNEL
    depends on BR2_USE_MMU
    help
      Kernel driver for MyHardware SPI peripheral.

      This driver provides /dev/mydevice character device interface.
```

### 8.5 Enabling the Out-of-Tree Module

In your Buildroot `defconfig`:

```ini
BR2_PACKAGE_MYDRIVER=y
```

Or interactively:

```
  Buildroot menuconfig
  └── Target packages
      └── Hardware handling
          └── [*] mydriver
```

---

## 9. C Code Examples

### 9.1 Minimal In-Tree Linux Kernel Driver

This is a complete, compilable in-tree character device driver that demonstrates the core patterns used in production embedded drivers.

```c
// drivers/misc/myboard_led.c
// In-tree LED driver for myboard — register, probe, file ops, cleanup

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/of_address.h>

#define DRIVER_NAME     "myboard_led"
#define DEVICE_NAME     "mbled"
#define CLASS_NAME      "myboard"

/* Register offsets in the LED controller MMIO block */
#define LED_CTRL_REG    0x00
#define LED_STATUS_REG  0x04
#define LED_BLINK_REG   0x08

struct myboard_led_dev {
    void __iomem    *base;          /* MMIO base address */
    struct cdev      cdev;
    struct device   *device;
    dev_t            devno;
    u32              num_leds;
};

static struct class  *led_class;
static dev_t          led_devno;

/* ---------- File Operations ---------- */

static ssize_t led_write(struct file *file, const char __user *buf,
                          size_t count, loff_t *pos)
{
    struct myboard_led_dev *ldev = file->private_data;
    char cmd[16];
    u32  val;

    if (count > sizeof(cmd) - 1)
        return -EINVAL;

    if (copy_from_user(cmd, buf, count))
        return -EFAULT;

    cmd[count] = '\0';

    if (sysfs_streq(cmd, "on")) {
        val = ioread32(ldev->base + LED_CTRL_REG);
        iowrite32(val | 0x01, ldev->base + LED_CTRL_REG);
    } else if (sysfs_streq(cmd, "off")) {
        val = ioread32(ldev->base + LED_CTRL_REG);
        iowrite32(val & ~0x01, ldev->base + LED_CTRL_REG);
    } else {
        return -EINVAL;
    }

    return count;
}

static ssize_t led_read(struct file *file, char __user *buf,
                         size_t count, loff_t *pos)
{
    struct myboard_led_dev *ldev = file->private_data;
    u32 status;
    char msg[8];
    int len;

    status = ioread32(ldev->base + LED_STATUS_REG);
    len = snprintf(msg, sizeof(msg), "%s\n", (status & 0x01) ? "on" : "off");

    return simple_read_from_buffer(buf, count, pos, msg, len);
}

static int led_open(struct inode *inode, struct file *file)
{
    struct myboard_led_dev *ldev =
        container_of(inode->i_cdev, struct myboard_led_dev, cdev);
    file->private_data = ldev;
    return 0;
}

static const struct file_operations led_fops = {
    .owner   = THIS_MODULE,
    .open    = led_open,
    .read    = led_read,
    .write   = led_write,
};

/* ---------- Platform Driver Probe/Remove ---------- */

static int myboard_led_probe(struct platform_device *pdev)
{
    struct myboard_led_dev *ldev;
    struct resource        *res;
    int                     ret;

    ldev = devm_kzalloc(&pdev->dev, sizeof(*ldev), GFP_KERNEL);
    if (!ldev)
        return -ENOMEM;

    /* Map MMIO region from device tree */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    ldev->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(ldev->base))
        return PTR_ERR(ldev->base);

    /* Read number of LEDs from device tree */
    if (of_property_read_u32(pdev->dev.of_node, "num-leds", &ldev->num_leds))
        ldev->num_leds = 1;

    /* Allocate character device number */
    ret = alloc_chrdev_region(&ldev->devno, 0, 1, DEVICE_NAME);
    if (ret)
        return ret;

    /* Initialize and add cdev */
    cdev_init(&ldev->cdev, &led_fops);
    ldev->cdev.owner = THIS_MODULE;
    ret = cdev_add(&ldev->cdev, ldev->devno, 1);
    if (ret)
        goto err_unreg;

    /* Create device node /dev/mbled */
    ldev->device = device_create(led_class, &pdev->dev,
                                  ldev->devno, NULL, DEVICE_NAME);
    if (IS_ERR(ldev->device)) {
        ret = PTR_ERR(ldev->device);
        goto err_del;
    }

    platform_set_drvdata(pdev, ldev);
    dev_info(&pdev->dev, "myboard_led: %u LED(s) registered at %p\n",
             ldev->num_leds, ldev->base);
    return 0;

err_del:
    cdev_del(&ldev->cdev);
err_unreg:
    unregister_chrdev_region(ldev->devno, 1);
    return ret;
}

static int myboard_led_remove(struct platform_device *pdev)
{
    struct myboard_led_dev *ldev = platform_get_drvdata(pdev);

    device_destroy(led_class, ldev->devno);
    cdev_del(&ldev->cdev);
    unregister_chrdev_region(ldev->devno, 1);
    return 0;
}

/* ---------- Device Tree Match Table ---------- */

static const struct of_device_id myboard_led_of_match[] = {
    { .compatible = "myvendor,myboard-led" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, myboard_led_of_match);

static struct platform_driver myboard_led_driver = {
    .probe  = myboard_led_probe,
    .remove = myboard_led_remove,
    .driver = {
        .name           = DRIVER_NAME,
        .of_match_table = myboard_led_of_match,
    },
};

/* ---------- Module Init/Exit ---------- */

static int __init myboard_led_init(void)
{
    int ret;

    led_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(led_class))
        return PTR_ERR(led_class);

    ret = platform_driver_register(&myboard_led_driver);
    if (ret) {
        class_destroy(led_class);
        return ret;
    }

    pr_info("myboard_led: driver loaded\n");
    return 0;
}

static void __exit myboard_led_exit(void)
{
    platform_driver_unregister(&myboard_led_driver);
    class_destroy(led_class);
    pr_info("myboard_led: driver unloaded\n");
}

module_init(myboard_led_init);
module_exit(myboard_led_exit);

MODULE_AUTHOR("Embedded Developer <dev@example.com>");
MODULE_DESCRIPTION("myboard LED driver");
MODULE_LICENSE("GPL v2");
```

### 9.2 The Corresponding Kconfig and Makefile

```kconfig
# drivers/misc/Kconfig (excerpt to add)
config MYBOARD_LED
    tristate "myboard LED controller driver"
    depends on OF && HAS_IOMEM
    help
      Driver for the LED controller on myboard hardware.
      Provides /dev/mbled character device.

      To compile as a module, choose M here; module name: myboard_led.
```

```makefile
# drivers/misc/Makefile (add one line)
obj-$(CONFIG_MYBOARD_LED) += myboard_led.o
```

### 9.3 Userspace Test Application

```c
// test/led_test.c — userspace test for the LED driver

#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

#define LED_DEV "/dev/mbled"

int main(int argc, char *argv[])
{
    int fd;
    char buf[16];
    ssize_t n;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s [on|off|status]\n", argv[0]);
        return EXIT_FAILURE;
    }

    fd = open(LED_DEV, O_RDWR);
    if (fd < 0) {
        perror("open " LED_DEV);
        return EXIT_FAILURE;
    }

    if (strcmp(argv[1], "status") == 0) {
        n = read(fd, buf, sizeof(buf) - 1);
        if (n < 0) {
            perror("read");
            close(fd);
            return EXIT_FAILURE;
        }
        buf[n] = '\0';
        printf("LED status: %s", buf);
    } else {
        n = write(fd, argv[1], strlen(argv[1]));
        if (n < 0) {
            perror("write");
            close(fd);
            return EXIT_FAILURE;
        }
        printf("LED set to: %s\n", argv[1]);
    }

    close(fd);
    return EXIT_SUCCESS;
}
```

---

## 10. C++ Code Examples

While the Linux kernel itself is written in C, userspace tools for interacting with kernel interfaces are often written in C++. The following examples demonstrate C++ patterns for kernel configuration management and device interaction.

### 10.1 Kernel Config Parser (C++)

A C++ utility to parse and validate a kernel `.config` file — useful in CI/CD pipelines to enforce required symbol states.

```cpp
// tools/kconfig_checker.cpp
// Validates a kernel .config against a set of required symbol rules
// Build: g++ -std=c++17 -O2 kconfig_checker.cpp -o kconfig_checker

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <vector>
#include <stdexcept>
#include <algorithm>
#include <filesystem>

namespace fs = std::filesystem;

// Represents the state of a CONFIG_ symbol
enum class SymbolState {
    YES,        // =y   (built-in)
    MODULE,     // =m   (loadable module)
    NO,         // # CONFIG_FOO is not set
    MISSING,    // not present in the file at all
};

std::string to_string(SymbolState s)
{
    switch (s) {
        case SymbolState::YES:     return "y";
        case SymbolState::MODULE:  return "m";
        case SymbolState::NO:      return "n";
        case SymbolState::MISSING: return "(missing)";
    }
    return "?";
}

class KernelConfig {
public:
    explicit KernelConfig(const fs::path &config_path)
    {
        parse(config_path);
    }

    SymbolState get(const std::string &symbol) const
    {
        auto it = symbols_.find(symbol);
        if (it == symbols_.end())
            return SymbolState::MISSING;
        return it->second;
    }

    // Returns true if symbol is enabled (y or m)
    bool is_enabled(const std::string &symbol) const
    {
        auto s = get(symbol);
        return s == SymbolState::YES || s == SymbolState::MODULE;
    }

    void print_summary(std::ostream &out) const
    {
        out << "Loaded " << symbols_.size() << " symbols.\n";
    }

private:
    std::unordered_map<std::string, SymbolState> symbols_;

    void parse(const fs::path &path)
    {
        std::ifstream f(path);
        if (!f)
            throw std::runtime_error("Cannot open: " + path.string());

        std::string line;
        while (std::getline(f, line)) {
            // Trim leading whitespace
            auto start = line.find_first_not_of(" \t");
            if (start == std::string::npos) continue;
            line = line.substr(start);

            // Comment line: "# CONFIG_FOO is not set"
            if (line.rfind("# CONFIG_", 0) == 0) {
                auto end = line.find(" is not set");
                if (end != std::string::npos) {
                    std::string sym = line.substr(2, end - 2);
                    symbols_[sym] = SymbolState::NO;
                }
                continue;
            }

            // Skip other comments and empty lines
            if (line.empty() || line[0] == '#') continue;

            // CONFIG_FOO=y / CONFIG_FOO=m / CONFIG_FOO="string" / CONFIG_FOO=0x123
            auto eq = line.find('=');
            if (eq == std::string::npos) continue;

            std::string sym = line.substr(0, eq);
            std::string val = line.substr(eq + 1);

            if (val == "y")
                symbols_[sym] = SymbolState::YES;
            else if (val == "m")
                symbols_[sym] = SymbolState::MODULE;
            else
                symbols_[sym] = SymbolState::YES; // string/numeric value = present
        }
    }
};

// A rule: a symbol must be in a required state
struct Rule {
    std::string  symbol;
    SymbolState  required;
    std::string  reason;

    bool operator()(const KernelConfig &cfg) const
    {
        return cfg.get(symbol) == required;
    }
};

class RuleChecker {
public:
    void add_rule(std::string sym, SymbolState req, std::string reason)
    {
        rules_.push_back({std::move(sym), req, std::move(reason)});
    }

    // Returns number of failed rules
    int check(const KernelConfig &cfg, std::ostream &out) const
    {
        int failures = 0;
        for (const auto &r : rules_) {
            bool ok = r(cfg);
            out << (ok ? "  [PASS]" : "  [FAIL]")
                << "  " << r.symbol
                << " = " << to_string(cfg.get(r.symbol))
                << "  (required: " << to_string(r.required) << ")"
                << "  -- " << r.reason
                << "\n";
            if (!ok) ++failures;
        }
        return failures;
    }

private:
    std::vector<Rule> rules_;
};

int main(int argc, char *argv[])
{
    if (argc < 2) {
        std::cerr << "Usage: kconfig_checker <.config path>\n";
        return 2;
    }

    try {
        KernelConfig cfg(argv[1]);
        cfg.print_summary(std::cout);
        std::cout << "\n";

        // Define required rules for an embedded networking product
        RuleChecker checker;
        checker.add_rule("CONFIG_NET",          SymbolState::YES,    "networking required");
        checker.add_rule("CONFIG_INET",         SymbolState::YES,    "TCP/IP required");
        checker.add_rule("CONFIG_EXT4_FS",      SymbolState::YES,    "rootfs filesystem");
        checker.add_rule("CONFIG_TMPFS",        SymbolState::YES,    "/tmp support");
        checker.add_rule("CONFIG_PROC_FS",      SymbolState::YES,    "/proc required");
        checker.add_rule("CONFIG_SYSFS",        SymbolState::YES,    "/sys required");
        checker.add_rule("CONFIG_BLK_DEV_SD",   SymbolState::YES,    "SCSI disk required");
        checker.add_rule("CONFIG_SOUND",        SymbolState::NO,     "audio not needed, saves RAM");
        checker.add_rule("CONFIG_DEBUG_KERNEL", SymbolState::NO,     "no debug in release build");
        checker.add_rule("CONFIG_USB_ACM",      SymbolState::MODULE, "USB modem as module");

        std::cout << "Checking rules:\n";
        int failures = checker.check(cfg, std::cout);

        std::cout << "\nResult: " << failures << " failure(s).\n";
        return failures > 0 ? 1 : 0;

    } catch (const std::exception &e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 2;
    }
}
```

### 10.2 Out-of-Tree Module Wrapper with Sysfs (C++ Host Tool)

```cpp
// tools/module_manager.cpp
// C++ tool to load/unload kernel modules and read their sysfs attributes
// Build: g++ -std=c++17 module_manager.cpp -o module_manager

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <vector>
#include <filesystem>
#include <stdexcept>
#include <cstring>

namespace fs = std::filesystem;

class KernelModule {
public:
    explicit KernelModule(std::string name) : name_(std::move(name)) {}

    // Load the module using modprobe
    void load(const std::vector<std::string> &params = {}) const
    {
        std::string cmd = "modprobe " + name_;
        for (const auto &p : params)
            cmd += " " + p;

        int ret = std::system(cmd.c_str());
        if (ret != 0)
            throw std::runtime_error("modprobe " + name_ + " failed: " + std::to_string(ret));
    }

    // Unload the module
    void unload() const
    {
        std::string cmd = "modprobe -r " + name_;
        int ret = std::system(cmd.c_str());
        if (ret != 0)
            throw std::runtime_error("modprobe -r " + name_ + " failed");
    }

    // Check if module is currently loaded
    bool is_loaded() const
    {
        std::ifstream modules("/proc/modules");
        std::string line;
        while (std::getline(modules, line)) {
            if (line.find(name_) == 0)
                return true;
        }
        return false;
    }

    // Read a sysfs attribute under /sys/module/<name>/parameters/
    std::string read_param(const std::string &param) const
    {
        fs::path path = fs::path("/sys/module") / name_ / "parameters" / param;
        std::ifstream f(path);
        if (!f)
            throw std::runtime_error("Cannot read: " + path.string());
        std::string value;
        std::getline(f, value);
        return value;
    }

    // Write a sysfs attribute
    void write_param(const std::string &param, const std::string &value) const
    {
        fs::path path = fs::path("/sys/module") / name_ / "parameters" / param;
        std::ofstream f(path);
        if (!f)
            throw std::runtime_error("Cannot write: " + path.string());
        f << value;
    }

    const std::string &name() const { return name_; }

private:
    std::string name_;
};

int main()
{
    KernelModule mydrv("mydriver");

    std::cout << "Module '" << mydrv.name() << "' loaded: "
              << (mydrv.is_loaded() ? "yes" : "no") << "\n";

    if (!mydrv.is_loaded()) {
        std::cout << "Loading with debug=1...\n";
        mydrv.load({"debug=1"});
    }

    try {
        auto val = mydrv.read_param("debug");
        std::cout << "debug parameter = " << val << "\n";
    } catch (const std::exception &e) {
        std::cerr << "Warning: " << e.what() << "\n";
    }

    return 0;
}
```

---

## 11. Rust Code Examples

Rust support in the Linux kernel (via `CONFIG_RUST=y`) has matured significantly since 6.1. Out-of-tree Rust modules and Rust-based host tools are both common in modern embedded workflows.

### 11.1 Enabling Rust in the Kernel Config

Add to your kernel fragment:

```
# board/myboard/linux-rust.frag
CONFIG_RUST=y
CONFIG_RUSTC_VERSION_TEXT="rustc 1.78.0 (9b00956e5 2024-04-29)"
```

In your Buildroot defconfig, also enable Rust:

```ini
BR2_PACKAGE_HOST_RUSTC=y
BR2_PACKAGE_HOST_RUST=y
```

### 11.2 Rust Kernel Module Skeleton

```rust
// drivers/misc/myboard_led_rust.rs
// Rust equivalent of the C LED driver above
// Requires CONFIG_RUST=y and the kernel Rust bindings

// kernel crate provides safe wrappers around kernel APIs
use kernel::prelude::*;
use kernel::{
    chrdev,
    file::{self, File, Operations},
    io_buffer::{IoBufferReader, IoBufferWriter},
    platform,
    of,
    Module,
};

module! {
    type: MyBoardLed,
    name: "myboard_led_rust",
    author: "Embedded Developer",
    description: "myboard LED driver (Rust)",
    license: "GPL v2",
}

// Per-device state
struct LedState {
    base: usize,   // MMIO base (simplified)
    on:   bool,
}

// The module-level struct — holds class and cdev registrations
struct MyBoardLed {
    _chrdev: Pin<Box<chrdev::Registration<2>>>,
}

// File operations for /dev/mbled
struct LedFileOps;

#[vtable]
impl Operations for LedFileOps {
    type Data = Arc<kernel::sync::Mutex<LedState>>;

    fn open(context: &Self::Data, _file: &File) -> Result<Self::Data> {
        Ok(context.clone())
    }

    fn read(
        data: ArcBorrow<'_, kernel::sync::Mutex<LedState>>,
        _file: &File,
        writer: &mut impl IoBufferWriter,
        _offset: u64,
    ) -> Result<usize> {
        let state = data.lock();
        let msg = if state.on { b"on\n" as &[u8] } else { b"off\n" };
        writer.write_slice(msg)?;
        Ok(msg.len())
    }

    fn write(
        data: ArcBorrow<'_, kernel::sync::Mutex<LedState>>,
        _file: &File,
        reader: &mut impl IoBufferReader,
        _offset: u64,
    ) -> Result<usize> {
        let mut buf = [0u8; 8];
        let len = reader.read_slice(&mut buf)?;
        let cmd = core::str::from_utf8(&buf[..len])
            .map_err(|_| EINVAL)?
            .trim();

        let mut state = data.lock();
        match cmd {
            "on"  => state.on = true,
            "off" => state.on = false,
            _     => return Err(EINVAL),
        }
        // In a real driver: iowrite32 to state.base + LED_CTRL_REG
        pr_info!("LED set to: {}\n", cmd);
        Ok(len)
    }
}

impl Module for MyBoardLed {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("myboard_led_rust: init\n");

        let chrdev_reg = chrdev::Registration::new_pinned(
            c_str!("mbled"),
            0,
            _module,
        )?;

        Ok(MyBoardLed {
            _chrdev: chrdev_reg,
        })
    }
}

impl Drop for MyBoardLed {
    fn drop(&mut self) {
        pr_info!("myboard_led_rust: exit\n");
    }
}
```

### 11.3 Rust Host Tool — defconfig Fragment Generator

This Rust binary runs on the host and generates Buildroot-ready kernel config fragments from a TOML specification — a clean, type-safe alternative to maintaining raw text files.

```toml
# board/myboard/features.toml — describes desired kernel features
[features]
networking  = true
usb_host    = true
bluetooth   = false
debug       = false
sound       = false
```

```rust
// tools/gen_kfrag/src/main.rs
// Generates a kernel .frag file from a TOML feature spec.
// Usage: gen_kfrag features.toml > board/myboard/linux-auto.frag
//
// Add to Cargo.toml:
//   [dependencies]
//   toml = "0.8"
//   serde = { version = "1", features = ["derive"] }

use std::collections::HashMap;
use std::env;
use std::fs;
use std::io::{self, Write};

use serde::Deserialize;

#[derive(Deserialize)]
struct FeatureSpec {
    features: HashMap<String, bool>,
}

/// Maps a feature name to the list of CONFIG_ symbols it requires
fn feature_to_symbols(feature: &str, enabled: bool) -> Vec<(String, &'static str)> {
    let y_or_n = |b: bool| if b { "y" } else { "n" };

    match feature {
        "networking" => vec![
            (format!("CONFIG_NET={}", y_or_n(enabled)), ""),
            (format!("CONFIG_INET={}", y_or_n(enabled)), ""),
            (format!("CONFIG_IPV6={}", y_or_n(enabled)), ""),
        ],
        "usb_host" => vec![
            (format!("CONFIG_USB={}", y_or_n(enabled)), ""),
            (format!("CONFIG_USB_XHCI_HCD={}", y_or_n(enabled)), ""),
            (format!("CONFIG_USB_EHCI_HCD={}", y_or_n(enabled)), ""),
            (format!("CONFIG_USB_ACM=m"), ""), // always module when USB enabled
        ],
        "bluetooth" => vec![
            (format!("CONFIG_BT={}", y_or_n(enabled)), ""),
            (format!("CONFIG_BT_HCIBTUSB={}", y_or_n(enabled)), ""),
        ],
        "debug" => vec![
            (format!("CONFIG_DEBUG_INFO={}", y_or_n(enabled)), ""),
            (format!("CONFIG_KGDB={}", y_or_n(enabled)), ""),
            (format!("CONFIG_MAGIC_SYSRQ={}", y_or_n(enabled)), ""),
            (format!("CONFIG_FTRACE={}", y_or_n(enabled)), ""),
        ],
        "sound" => vec![
            (format!("CONFIG_SOUND={}", y_or_n(enabled)), ""),
            (format!("CONFIG_SND={}", y_or_n(enabled)), ""),
        ],
        other => {
            eprintln!("Warning: unknown feature '{}', skipping.", other);
            vec![]
        }
    }
}

fn main() -> io::Result<()> {
    let path = env::args().nth(1).unwrap_or_else(|| {
        eprintln!("Usage: gen_kfrag <features.toml>");
        std::process::exit(1);
    });

    let content = fs::read_to_string(&path)?;
    let spec: FeatureSpec = toml::from_str(&content).unwrap_or_else(|e| {
        eprintln!("TOML parse error: {}", e);
        std::process::exit(1);
    });

    let stdout = io::stdout();
    let mut out = stdout.lock();

    writeln!(out, "# Auto-generated kernel fragment — do not edit manually")?;
    writeln!(out, "# Source: {}", path)?;
    writeln!(out, "#")?;

    for (feature, enabled) in &spec.features {
        writeln!(out, "# Feature: {} = {}", feature, enabled)?;
        for (symbol, _comment) in feature_to_symbols(feature, *enabled) {
            // Emit in proper Kconfig format
            if symbol.ends_with("=n") {
                // Kconfig disabled syntax
                let sym_name = &symbol[..symbol.len() - 2];
                writeln!(out, "# {} is not set", sym_name)?;
            } else {
                writeln!(out, "{}", symbol)?;
            }
        }
        writeln!(out)?;
    }

    Ok(())
}
```

### 11.4 Rust Build Script for Buildroot Integration

```rust
// package/mydriver-rs/src/build.rs
// Buildroot out-of-tree Rust module build script
// Sets KERNEL_SRC for the kernel module build step

use std::env;
use std::path::PathBuf;

fn main() {
    // Buildroot sets LINUX_DIR to the kernel build directory
    let kernel_src = env::var("LINUX_DIR")
        .unwrap_or_else(|_| "/usr/src/linux".to_string());

    let kernel_path = PathBuf::from(&kernel_src);

    println!("cargo:rerun-if-env-changed=LINUX_DIR");
    println!("cargo:rerun-if-changed={}", kernel_path.join("Makefile").display());

    // Pass kernel headers location for bindgen (if generating bindings)
    println!("cargo:rustc-env=KERNEL_INCLUDE={}/include",
             kernel_src);
}
```

---

## 12. Summary

Linux kernel configuration in Buildroot is a multi-layered, well-tooled process. The diagram below captures the full workflow end-to-end:

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                 Complete Kernel Config Workflow Summary                  │
  │                                                                          │
  │  STEP 1: Choose a starting point                                         │
  │  ─────────────────────────────                                           │
  │  $ cd output/build/linux-x.y.z                                           │
  │  $ make ARCH=arm versatile_defconfig   ← upstream arch defconfig         │
  │  $ make ARCH=arm savedefconfig                                           │
  │  $ cp defconfig ../../../board/myboard/linux.config                      │
  │                                                                          │
  │  STEP 2: Set BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE                         │
  │  ────────────────────────────────────────────────                        │
  │  In configs/myboard_defconfig:                                           │
  │  BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="board/myboard/linux.config"        │
  │                                                                          │
  │  STEP 3: Add fragments for variants/features                             │
  │  ────────────────────────────────────────────                            │
  │  BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES=                                 │
  │      "board/myboard/linux-usb.frag board/myboard/linux-debug.frag"       │
  │                                                                          │
  │  STEP 4: Interactively tune                                              │
  │  ──────────────────────────                                              │
  │  $ make linux-menuconfig                                                 │
  │    (navigate, search with /, toggle with Y/N/M)                          │
  │                                                                          │
  │  STEP 5: Save your changes back                                          │
  │  ────────────────────────────                                            │
  │  $ make linux-update-defconfig                                           │
  │    (or: make linux-savedefconfig && cp …/defconfig board/myboard/        │
  │         linux.config)                                                    │
  │                                                                          │
  │  STEP 6: Build                                                           │
  │  ─────────                                                               │
  │  $ make                                                                  │
  │                                                                          │
  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                  │
  │  │  linux.config│   │  *.frag      │   │ Out-of-tree  │                  │
  │  │  (base)      │ + │  (overlays)  │ + │ modules (=m) │                  │
  │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                  │
  │         └──────────────────┴──────────────────┘                          │
  │                             │                                            │
  │                             ▼                                            │
  │                  merge_config.sh (kernel)                                │
  │                             │                                            │
  │                             ▼                                            │
  │              output/build/linux-x.y.z/.config                            │
  │                             │                                            │
  │                    ┌────────┴────────┐                                   │
  │                    ▼                 ▼                                   │
  │              vmlinuz / zImage    *.ko modules                            │
  │                    │                 │                                   │
  │                    └────────┬────────┘                                   │
  │                             ▼                                            │
  │                    target rootfs image                                   │
  └──────────────────────────────────────────────────────────────────────────┘
```

### Key Concepts — Quick Reference

| Concept | What It Does | When To Use |
|---|---|---|
| `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE` | Points to your board's kernel defconfig | Always — your primary kernel config |
| `BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES` | Overlays additional `CONFIG_` changes | Feature toggles, board variants, debug builds |
| `make linux-menuconfig` | Interactive kernel configuration TUI | Exploring options, one-off changes |
| `make linux-update-defconfig` | Saves minimal defconfig back to source | After any menuconfig session |
| `CONFIG_FOO=y` | Builds driver into kernel image | Boot-critical drivers |
| `CONFIG_FOO=m` | Builds driver as loadable module | Optional drivers, saves image size |
| In-tree module | Driver inside kernel source tree | Mainline drivers, standard hardware |
| Out-of-tree module | Driver as a Buildroot package | Proprietary or custom hardware |
| Config fragment | Partial override `.frag` file | Clean, composable config management |

### Design Principles for Production Embedded Systems

Start from the closest upstream defconfig and trim aggressively — every enabled symbol costs code size and boot time. Use fragments to keep variant-specific changes separate from the board base. Always round-trip test your defconfig (save → apply → save again → diff) to confirm it is stable. Prefer built-in (`=y`) for anything required before the rootfs is mounted, and modules (`=m`) for everything else. Commit only the minimal defconfig and fragments — never commit the full `.config` generated in `output/`.

---

*End of Chapter 11 — Linux Kernel Configuration & Custom defconfig*

*Next: Chapter 12 — Root Filesystem Customization & Overlays*