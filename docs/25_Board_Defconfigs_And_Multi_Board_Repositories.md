# 25. Board Defconfigs & Multi-Board Repositories

**Structure & Concepts**
- Full ASCII repository layout diagram for a `BR2_EXTERNAL` multi-board tree
- Defconfig anatomy with a side-by-side ASCII diff map (Lite vs Pro vs Server)
- Fragment merge precedence diagram showing how `common.config → board.config` layering resolves conflicts

**Practical Workflows**
- `merge_config.sh` shell script for assembling configs from fragments
- `make savedefconfig` edit→save→commit cycle with a convenience `Makefile` wrapper
- CI/CD GitLab YAML building all boards in parallel
- Shell script to verify defconfigs are already minimal (for CI gating)

**Code Examples**
- **C** — `board_info.c/h` with compile-time `-DBOARD_NAME` flags from `.mk`, runtime `/etc/board-info` cross-verification, and an ASCII capability bar display
- **C++** — `BoardMatrix` class rendering a Unicode box-drawn feature comparison table across all boards, with a `filter()` method for capability queries
- **Rust** — `build.rs` translating Buildroot env vars into `cfg` flags, then a zero-cost conditional HAL (`gpu`, `storage`, `network`) selected entirely at compile time — with sample output for both the Pro and Server boards

**Common Pitfalls** table covering the most frequent mistakes (non-minimal defconfigs, fragment order, shared output directories, absolute paths, debug config leaking into release).


> **Buildroot Series — Topic 25**
> Storing `*_defconfig` files in `BR2_EXTERNAL/configs/`, using fragments,
> sharing common config snippets across boards, and `make savedefconfig`.

---

## Table of Contents

1. [Conceptual Overview](#1-conceptual-overview)
2. [Repository Layout](#2-repository-layout)
3. [Defconfig Anatomy](#3-defconfig-anatomy)
4. [Config Fragments](#4-config-fragments)
5. [Sharing Common Snippets — merge_config](#5-sharing-common-snippets--merge_config)
6. [make savedefconfig Workflow](#6-make-savedefconfig-workflow)
7. [C Example — Board Identity at Runtime](#7-c-example--board-identity-at-runtime)
8. [C++ Example — Board Feature Matrix](#8-c-example--board-feature-matrix)
9. [Rust Example — Board HAL Dispatcher](#9-rust-example--board-hal-dispatcher)
10. [CI / Automation Scripts](#10-ci--automation-scripts)
11. [Common Pitfalls](#11-common-pitfalls)
12. [Summary](#12-summary)

---

## 1. Conceptual Overview

In production embedded projects you rarely target a single board.
A typical product line might have:

```
  Product Family "FooBox"
  ========================
  FooBox-Lite   → ARM Cortex-A7,  256 MB RAM, no GPU
  FooBox-Pro    → ARM Cortex-A53, 512 MB RAM, Mali GPU
  FooBox-Server → ARM Cortex-A72,   2 GB RAM, PCIe NVMe
```

All three share a large chunk of package selection, kernel configuration, and
toolchain settings, yet differ in a handful of details.

Buildroot's answer is the **defconfig / fragment / BR2_EXTERNAL** trio:

```
┌─────────────────────────────────────────────────────────────────┐
│                     BR2_EXTERNAL tree                           │
│                                                                 │
│  configs/                  ← defconfigs live here               │
│  ├── foobox_lite_defconfig                                      │
│  ├── foobox_pro_defconfig                                       │
│  └── foobox_server_defconfig                                    │
│                                                                 │
│  configs/fragments/        ← reusable snippets                  │
│  ├── common.config         ← shared by all boards               │
│  ├── gpu.config            ← opt-in GPU packages                │
│  └── debug.config          ← developer/debug overrides          │
│                                                                 │
│  board/                    ← board-specific files               │
│  package/                  ← custom packages                    │
│  Config.in                                                      │
│  external.mk                                                    │
└─────────────────────────────────────────────────────────────────┘
```

**Key principle:** a defconfig is a *minimal diff* from Buildroot defaults.
Only lines that differ from the default need to be stored.

---

## 2. Repository Layout

### 2.1 Minimal BR2_EXTERNAL skeleton

```
foobox-external/
│
├── Config.in               # Kconfig menu entries for custom packages
├── external.mk             # include custom package .mk files
├── external.desc           # name + description of this external tree
│
├── configs/
│   ├── foobox_lite_defconfig
│   ├── foobox_pro_defconfig
│   ├── foobox_server_defconfig
│   └── fragments/
│       ├── common.config
│       ├── gpu.config
│       ├── networking.config
│       └── debug.config
│
├── board/
│   ├── foobox_lite/
│   │   ├── linux.config
│   │   ├── post-build.sh
│   │   └── rootfs-overlay/
│   ├── foobox_pro/
│   │   ├── linux.config
│   │   ├── uboot.config
│   │   └── rootfs-overlay/
│   └── foobox_server/
│       ├── linux.config
│       └── rootfs-overlay/
│
└── package/
    └── foobox-init/
        ├── Config.in
        └── foobox-init.mk
```

### 2.2 external.desc

```
name: FOOBOX
desc: FooBox product family — multi-board external tree
```

### 2.3 Registering the external tree

```bash
# Single external tree
make BR2_EXTERNAL=/path/to/foobox-external foobox_lite_defconfig

# Multiple external trees (colon-separated)
make BR2_EXTERNAL=/path/to/foobox-external:/path/to/company-common \
     foobox_pro_defconfig
```

Once registered, the path is saved in `output/.br2-external.mk` and you can
omit `BR2_EXTERNAL` in subsequent `make` invocations inside that output
directory.

---

## 3. Defconfig Anatomy

### 3.1 foobox_lite_defconfig

```
# Architecture
BR2_arm=y
BR2_cortex_a7=y
BR2_ARM_FPU_VFPV4D16=y

# Toolchain
BR2_TOOLCHAIN_BUILDROOT_CXX=y
BR2_TOOLCHAIN_BUILDROOT_GLIBC=y
BR2_GCC_VERSION_12_X=y

# System
BR2_TARGET_GENERIC_HOSTNAME="foobox-lite"
BR2_TARGET_GENERIC_ISSUE="FooBox Lite 1.0"
BR2_SYSTEM_DHCP="eth0"

# Kernel
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="6.6.28"
BR2_LINUX_KERNEL_DEFCONFIG="custom"
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES=\
  "$(BR2_EXTERNAL_FOOBOX_PATH)/board/foobox_lite/linux.config"

# Bootloader
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="foobox_lite"

# Root filesystem
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y

# Packages (lite — no GPU)
BR2_PACKAGE_BUSYBOX=y
BR2_PACKAGE_DROPBEAR=y
BR2_PACKAGE_FOOBOX_INIT=y

# Post-build
BR2_ROOTFS_POST_BUILD_SCRIPT=\
  "$(BR2_EXTERNAL_FOOBOX_PATH)/board/foobox_lite/post-build.sh"
BR2_ROOTFS_POST_IMAGE_SCRIPT=\
  "support/scripts/genimage.sh"
```

### 3.2 foobox_pro_defconfig — differences only

```
# Architecture (Cortex-A53 instead of A7)
BR2_aarch64=y
BR2_cortex_a53=y

# System
BR2_TARGET_GENERIC_HOSTNAME="foobox-pro"
BR2_TARGET_GENERIC_ISSUE="FooBox Pro 1.0"

# Kernel fragment — adds GPU driver overlay
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES=\
  "$(BR2_EXTERNAL_FOOBOX_PATH)/board/foobox_pro/linux.config"

# Extra packages for Pro
BR2_PACKAGE_LIBMALI=y
BR2_PACKAGE_WESTON=y
BR2_PACKAGE_CHROMIUM_BROWSER=y

# Post-build (different script for pro)
BR2_ROOTFS_POST_BUILD_SCRIPT=\
  "$(BR2_EXTERNAL_FOOBOX_PATH)/board/foobox_pro/post-build.sh"
```

**Visual diff — Lite vs Pro:**

```
                ┌──────────────────────────────────────┐
                │         Defconfig Diff Map           │
                ├────────────────┬─────────────────────┤
  Config Key    │  foobox_lite   │    foobox_pro       │
  ──────────────┼────────────────┼─────────────────────┤
  Architecture  │  ARM A7 32-bit │  AArch64 A53 64-bit │
  Hostname      │  foobox-lite   │  foobox-pro         │
  GPU pkgs      │  (none)        │  libmali + weston   │
  Browser       │  (none)        │  chromium           │
  RAM target    │  256 MB        │  512 MB             │
  Linux frag    │  lite/linux.c  │  pro/linux.config   │
  Post-build    │  lite/post-b   │  pro/post-build.sh  │
                └────────────────┴─────────────────────┘
```

---

## 4. Config Fragments

A **config fragment** is a partial Kconfig file — only the options you want to
override or add. Buildroot's `support/kconfig/merge_config.sh` merges them
in order, later files winning conflicts.

### 4.1 fragments/common.config

```
# ─── common.config ────────────────────────────────────────────
# Shared by ALL FooBox board defconfigs.
# Toolchain hardening, security baseline, common utilities.

# Toolchain hardening
BR2_TOOLCHAIN_BUILDROOT_FORTIFY_SOURCE=y
BR2_TOOLCHAIN_BUILDROOT_SSP_ALL=y
BR2_RELRO_FULL=y

# Locale / timezone
BR2_TARGET_LOCALTIME="Europe/Berlin"
BR2_TARGET_ZONEINFO_POSIXRULES="Europe/Berlin"

# NTP
BR2_PACKAGE_CHRONY=y

# SSH
BR2_PACKAGE_OPENSSH=y
BR2_PACKAGE_OPENSSH_SERVER=y

# Diagnostics
BR2_PACKAGE_HTOP=y
BR2_PACKAGE_STRACE=y
BR2_PACKAGE_LSOF=y

# Logging
BR2_PACKAGE_RSYSLOG=y
```

### 4.2 fragments/gpu.config

```
# ─── gpu.config ───────────────────────────────────────────────
# Enable GPU / display stack. Include in Pro and Server defconfigs.

BR2_PACKAGE_MESA3D=y
BR2_PACKAGE_MESA3D_OPENGL_EGL=y
BR2_PACKAGE_MESA3D_OPENGL_ES=y
BR2_PACKAGE_LIBDRM=y
BR2_PACKAGE_LIBGBM=y
BR2_PACKAGE_WESTON=y
BR2_PACKAGE_WESTON_BACKEND_DRM=y
```

### 4.3 fragments/debug.config

```
# ─── debug.config ─────────────────────────────────────────────
# Developer overlay — NEVER merge into release builds.

BR2_ENABLE_DEBUG=y
BR2_PACKAGE_GDB=y
BR2_PACKAGE_GDB_SERVER=y
BR2_PACKAGE_VALGRIND=y
BR2_PACKAGE_PERF=y
BR2_PACKAGE_OPROFILE=y
BR2_STRIP_strip=n          # keep debug symbols
```

### 4.4 How fragments map to defconfigs

```
                  Defconfig Assembly
                  ==================

  foobox_lite_defconfig
        │
        ▼
  ┌─────────────┐     ┌──────────────────┐
  │   lite base │ ──► │  common.config   │ ──► MERGED .config
  └─────────────┘     └──────────────────┘
         (no gpu.config)

  foobox_pro_defconfig
        │
        ▼
  ┌─────────────┐     ┌──────────────────┐     ┌────────────┐
  │   pro base  │ ──► │  common.config   │ ──► │ gpu.config │ ──► MERGED .config
  └─────────────┘     └──────────────────┘     └────────────┘

  foobox_server_defconfig
        │
        ▼
  ┌──────────────┐     ┌──────────────────┐     ┌────────────┐
  │  server base │ ──► │  common.config   │ ──► │ gpu.config │ ──► MERGED .config
  └──────────────┘     └──────────────────┘     └────────────┘
```

Fragment paths are referenced inside the defconfig via
`BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES` (for the kernel) or via a custom
top-level merge script (for Buildroot's own `.config`).

---

## 5. Sharing Common Snippets — merge_config

### 5.1 Using merge_config.sh directly

```bash
#!/usr/bin/env bash
# scripts/build_pro.sh — assemble and build foobox_pro

set -euo pipefail

BR2_EXTERNAL=$(pwd)/../foobox-external
BUILDROOT=$(pwd)/../buildroot
OUTPUT=$(pwd)/../output/foobox_pro
FRAG=${BR2_EXTERNAL}/configs/fragments

mkdir -p "${OUTPUT}"

# 1. Start from the board defconfig
make -C "${BUILDROOT}" O="${OUTPUT}" \
     BR2_EXTERNAL="${BR2_EXTERNAL}" \
     foobox_pro_defconfig

# 2. Merge additional fragments on top
"${BUILDROOT}/support/kconfig/merge_config.sh" \
    -m -O "${OUTPUT}" \
    "${OUTPUT}/.config" \
    "${FRAG}/common.config" \
    "${FRAG}/gpu.config"

# 3. Resolve any newly introduced symbols to their defaults
make -C "${BUILDROOT}" O="${OUTPUT}" olddefconfig

# 4. Build
make -C "${BUILDROOT}" O="${OUTPUT}" -j"$(nproc)"
```

### 5.2 Using BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES (kernel only)

Inside a defconfig, you can layer multiple kernel fragments:

```
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES=\
  "$(BR2_EXTERNAL_FOOBOX_PATH)/board/common/linux-hardening.config \
   $(BR2_EXTERNAL_FOOBOX_PATH)/board/foobox_pro/linux-pro.config"
```

Buildroot will merge them in left-to-right order when building the kernel.

### 5.3 Fragment merge precedence (ASCII diagram)

```
  Lower priority ◄──────────────────────────────► Higher priority
  ───────────────────────────────────────────────────────────────
  [Buildroot defaults] → [base defconfig] → [common.config] → [board.config]

  Example conflict resolution:
  ┌──────────────┬──────────────┬───────────────┬──────────────┐
  │  Buildroot   │ base defcfg  │ common.config │ board.config │  WINS
  │  default     │              │               │              │
  ├──────────────┼──────────────┼───────────────┼──────────────┤
  │ BR2_PKG_X=n  │      -       │  BR2_PKG_X=y  │      -       │  =y
  │ BR2_PKG_Y=n  │ BR2_PKG_Y=y  │       -       │      -       │  =y
  │ BR2_PKG_Z=y  │      -       │  BR2_PKG_Z=n  │ BR2_PKG_Z=y  │  =y
  └──────────────┴──────────────┴───────────────┴──────────────┘
```

---

## 6. make savedefconfig Workflow

`make savedefconfig` writes the *minimal* representation of the current
`.config` back to the file pointed at by `BR2_DEFCONFIG`.

### 6.1 Typical edit→save cycle

```bash
# Step 1: Load an existing board defconfig
make O=output/foobox_lite \
     BR2_EXTERNAL=/path/to/foobox-external \
     foobox_lite_defconfig

# Step 2: Interactively tweak options
make O=output/foobox_lite menuconfig        # Buildroot packages
make O=output/foobox_lite linux-menuconfig  # Kernel config
make O=output/foobox_lite uboot-menuconfig  # U-Boot config

# Step 3: Save the minimal defconfig back to the external tree
make O=output/foobox_lite \
     BR2_DEFCONFIG=/path/to/foobox-external/configs/foobox_lite_defconfig \
     savedefconfig

# Step 4: Commit
git -C /path/to/foobox-external add configs/foobox_lite_defconfig
git -C /path/to/foobox-external commit -m "foobox_lite: add dropbear package"
```

### 6.2 What savedefconfig strips out

```
  Full .config (thousands of lines)
  ──────────────────────────────────
  # BR2_aarch64 is not set          ← DEFAULT (n) — stripped
  BR2_arm=y                         ← non-default — KEPT
  # BR2_cortex_a5 is not set        ← DEFAULT — stripped
  BR2_cortex_a7=y                   ← non-default — KEPT
  BR2_PACKAGE_BUSYBOX=y             ← non-default — KEPT
  # BR2_PACKAGE_ALSA_LIB is not set ← DEFAULT (n) — stripped
  ...

  Resulting minimal defconfig (tens of lines)
  ────────────────────────────────────────────
  BR2_arm=y
  BR2_cortex_a7=y
  BR2_PACKAGE_BUSYBOX=y
  ...
```

### 6.3 Makefile helper in the external tree

```makefile
# foobox-external/Makefile (convenience wrapper)
BUILDROOT  ?= $(HOME)/src/buildroot
BR2_EXT    := $(CURDIR)
OUTPUT_DIR ?= $(CURDIR)/output

.PHONY: lite pro server save-lite save-pro save-server

lite:
	$(MAKE) -C $(BUILDROOT) O=$(OUTPUT_DIR)/lite \
	        BR2_EXTERNAL=$(BR2_EXT) foobox_lite_defconfig
	$(MAKE) -C $(BUILDROOT) O=$(OUTPUT_DIR)/lite -j$(nproc)

pro:
	$(MAKE) -C $(BUILDROOT) O=$(OUTPUT_DIR)/pro \
	        BR2_EXTERNAL=$(BR2_EXT) foobox_pro_defconfig
	$(MAKE) -C $(BUILDROOT) O=$(OUTPUT_DIR)/pro -j$(nproc)

save-lite:
	$(MAKE) -C $(BUILDROOT) O=$(OUTPUT_DIR)/lite \
	        BR2_DEFCONFIG=$(BR2_EXT)/configs/foobox_lite_defconfig \
	        savedefconfig

save-pro:
	$(MAKE) -C $(BUILDROOT) O=$(OUTPUT_DIR)/pro \
	        BR2_DEFCONFIG=$(BR2_EXT)/configs/foobox_pro_defconfig \
	        savedefconfig
```

---

## 7. C Example — Board Identity at Runtime

The board defconfig bakes `BR2_TARGET_GENERIC_HOSTNAME` and custom symbols
into the system. The following C utility reads the board identity at runtime
from `/etc/board-info` (written by the post-build script) and verifies it
against a compiled-in expected value.

### 7.1 board_info.h

```c
/* board_info.h — compile-time board constants injected by Buildroot */
#ifndef BOARD_INFO_H
#define BOARD_INFO_H

#include <stdint.h>

/* These are set via -D flags in the package .mk file,
 * derived from the active defconfig. */
#ifndef BOARD_NAME
#  define BOARD_NAME    "unknown"
#endif
#ifndef BOARD_VARIANT
#  define BOARD_VARIANT "unknown"
#endif
#ifndef BOARD_VERSION
#  define BOARD_VERSION "0.0.0"
#endif

typedef struct {
    const char *name;
    const char *variant;
    const char *version;
    uint32_t    capabilities;  /* bitmask — see BOARD_CAP_* */
} board_info_t;

/* Capability flags */
#define BOARD_CAP_GPU       (1u << 0)
#define BOARD_CAP_NVME      (1u << 1)
#define BOARD_CAP_WIFI      (1u << 2)
#define BOARD_CAP_BT        (1u << 3)
#define BOARD_CAP_CAN       (1u << 4)

const board_info_t *board_get_info(void);

#endif /* BOARD_INFO_H */
```

### 7.2 board_info.c

```c
/* board_info.c — runtime board identity and capability reporting */

#include "board_info.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define BOARD_FILE  "/etc/board-info"
#define LINE_MAX    256

/* ─── static compiled-in record ─────────────────────────────────── */
static const board_info_t s_info = {
    .name       = BOARD_NAME,
    .variant    = BOARD_VARIANT,
    .version    = BOARD_VERSION,
    .capabilities =
#ifdef BOARD_HAS_GPU
        BOARD_CAP_GPU  |
#endif
#ifdef BOARD_HAS_NVME
        BOARD_CAP_NVME |
#endif
#ifdef BOARD_HAS_WIFI
        BOARD_CAP_WIFI |
#endif
        0u
};

const board_info_t *board_get_info(void)
{
    return &s_info;
}

/* ─── runtime verification ──────────────────────────────────────── */
static int read_runtime_name(char *buf, size_t bufsz)
{
    FILE *fp = fopen(BOARD_FILE, "r");
    if (!fp) {
        fprintf(stderr, "board_info: cannot open %s: %s\n",
                BOARD_FILE, strerror(errno));
        return -1;
    }

    char line[LINE_MAX];
    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "BOARD_NAME=", 11) == 0) {
            /* strip trailing newline */
            char *val = line + 11;
            val[strcspn(val, "\r\n")] = '\0';
            snprintf(buf, bufsz, "%s", val);
            fclose(fp);
            return 0;
        }
    }
    fclose(fp);
    return -1;
}

/* ─── capability banner (ASCII art) ────────────────────────────── */
static void print_capability_bar(uint32_t caps)
{
    printf("\n  Capabilities\n");
    printf("  ============\n");

    struct { uint32_t flag; const char *label; } tbl[] = {
        { BOARD_CAP_GPU,  "GPU  " },
        { BOARD_CAP_NVME, "NVMe " },
        { BOARD_CAP_WIFI, "WiFi " },
        { BOARD_CAP_BT,   "BT   " },
        { BOARD_CAP_CAN,  "CAN  " },
    };

    printf("  ");
    for (size_t i = 0; i < sizeof(tbl)/sizeof(tbl[0]); ++i)
        printf("[%s]", tbl[i].label);
    printf("\n  ");
    for (size_t i = 0; i < sizeof(tbl)/sizeof(tbl[0]); ++i)
        printf("  %s ", (caps & tbl[i].flag) ? " ON " : "off ");
    printf("\n\n");
}

/* ─── main ──────────────────────────────────────────────────────── */
int main(void)
{
    const board_info_t *bi = board_get_info();
    char runtime_name[64] = "(unavailable)";

    (void)read_runtime_name(runtime_name, sizeof(runtime_name));

    printf("+---------------------------------------+\n");
    printf("|        FooBox Board Identity          |\n");
    printf("+---------------------------------------+\n");
    printf("| Compiled-in name  : %-17s |\n", bi->name);
    printf("| Runtime name      : %-17s |\n", runtime_name);
    printf("| Variant           : %-17s |\n", bi->variant);
    printf("| Version           : %-17s |\n", bi->version);
    printf("+---------------------------------------+\n");

    if (strcmp(bi->name, runtime_name) != 0) {
        fprintf(stderr,
            "WARNING: compiled-in name '%s' != runtime name '%s'\n",
            bi->name, runtime_name);
        fprintf(stderr,
            "         Wrong image flashed to this board?\n");
    }

    print_capability_bar(bi->capabilities);
    return 0;
}
```

### 7.3 foobox-init.mk — injecting -D flags from the defconfig

```makefile
################################################################################
# foobox-init — sets compile-time board constants from Buildroot variables
################################################################################

FOOBOX_INIT_VERSION  = 1.0.0
FOOBOX_INIT_SITE     = $(BR2_EXTERNAL_FOOBOX_PATH)/package/foobox-init
FOOBOX_INIT_SITE_METHOD = local

FOOBOX_INIT_CFLAGS  = -DBOARD_NAME=\"$(BR2_TARGET_GENERIC_HOSTNAME)\"
FOOBOX_INIT_CFLAGS += -DBOARD_VERSION=\"$(FOOBOX_INIT_VERSION)\"

# Conditional capabilities derived from selected packages
ifeq ($(BR2_PACKAGE_LIBMALI),y)
FOOBOX_INIT_CFLAGS += -DBOARD_HAS_GPU
endif
ifeq ($(BR2_PACKAGE_NVME_CLI),y)
FOOBOX_INIT_CFLAGS += -DBOARD_HAS_NVME
endif

define FOOBOX_INIT_BUILD_CMDS
	$(TARGET_CC) $(TARGET_CFLAGS) $(FOOBOX_INIT_CFLAGS) \
	    $(@D)/board_info.c -o $(@D)/board_info
endef

define FOOBOX_INIT_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/board_info $(TARGET_DIR)/usr/bin/board_info
endef

$(eval $(generic-package))
```

### 7.4 Sample output on foobox_pro

```
+---------------------------------------+
|        FooBox Board Identity          |
+---------------------------------------+
| Compiled-in name  : foobox-pro        |
| Runtime name      : foobox-pro        |
| Variant           : Pro               |
| Version           : 1.0.0             |
+---------------------------------------+

  Capabilities
  ============
  [GPU  ][NVMe ][WiFi ][BT   ][CAN  ]
    ON    off    off    off    off
```

---

## 8. C++ Example — Board Feature Matrix

This C++ class reads all board configs from a JSON manifest (shipped by the
external tree) and renders an ASCII comparison matrix — useful in factory
tooling or an SDK-side configuration UI.

### 8.1 board_matrix.hpp

```cpp
// board_matrix.hpp — board feature comparison matrix
#pragma once

#include <string>
#include <vector>
#include <map>
#include <functional>
#include <ostream>

namespace foobox {

struct Feature {
    std::string key;        // internal key, e.g. "gpu"
    std::string label;      // display label, e.g. "GPU"
};

struct Board {
    std::string name;
    std::string defconfig;  // e.g. "foobox_pro_defconfig"
    std::string arch;
    uint32_t    ram_mb;
    std::map<std::string, bool> features;
};

class BoardMatrix {
public:
    explicit BoardMatrix(std::vector<Feature> features);

    void add_board(Board b);

    /// Render ASCII comparison table to the given stream.
    void render(std::ostream &out) const;

    /// Return boards that have ALL requested features enabled.
    std::vector<const Board*>
    filter(std::vector<std::string> required_features) const;

private:
    std::vector<Feature> features_;
    std::vector<Board>   boards_;

    static std::string center(const std::string &s, int width);
    static std::string hline(int width, char fill = '─',
                             char left = '├', char mid = '┼',
                             char right = '┤',
                             const std::vector<int> &cols = {});
};

} // namespace foobox
```

### 8.2 board_matrix.cpp

```cpp
// board_matrix.cpp — board feature matrix implementation
#include "board_matrix.hpp"

#include <algorithm>
#include <iomanip>
#include <iostream>
#include <numeric>
#include <sstream>

namespace foobox {

// ─── helpers ──────────────────────────────────────────────────────────────

std::string BoardMatrix::center(const std::string &s, int width)
{
    int pad   = width - static_cast<int>(s.size());
    int left  = pad / 2;
    int right = pad - left;
    return std::string(left, ' ') + s + std::string(right, ' ');
}

std::string BoardMatrix::hline(int /*width*/, char fill,
                               char left, char mid, char right,
                               const std::vector<int> &cols)
{
    std::string out;
    out += left;
    for (size_t i = 0; i < cols.size(); ++i) {
        out += std::string(cols[i], fill);
        out += (i + 1 < cols.size()) ? mid : right;
    }
    return out;
}

// ─── constructor ──────────────────────────────────────────────────────────

BoardMatrix::BoardMatrix(std::vector<Feature> features)
    : features_(std::move(features))
{}

void BoardMatrix::add_board(Board b)
{
    boards_.push_back(std::move(b));
}

// ─── render ───────────────────────────────────────────────────────────────

void BoardMatrix::render(std::ostream &out) const
{
    if (boards_.empty()) { out << "(no boards)\n"; return; }

    // Column widths
    constexpr int COL_FEAT  = 10;  // feature label column
    constexpr int COL_BOARD = 14;  // per-board column

    int total_boards = static_cast<int>(boards_.size());
    std::vector<int> cols;
    cols.push_back(COL_FEAT);
    for (int i = 0; i < total_boards; ++i)
        cols.push_back(COL_BOARD);

    auto top  = hline(0, '─', '┌', '┬', '┐', cols);
    auto sep  = hline(0, '─', '├', '┼', '┤', cols);
    auto bot  = hline(0, '─', '└', '┴', '┘', cols);

    // ── header ──
    out << top << '\n';
    out << "│" << center("Feature", COL_FEAT) << "│";
    for (auto &b : boards_)
        out << center(b.name, COL_BOARD) << "│";
    out << '\n';

    out << "│" << center("", COL_FEAT) << "│";
    for (auto &b : boards_)
        out << center(b.arch, COL_BOARD) << "│";
    out << '\n';

    out << "│" << center("RAM", COL_FEAT) << "│";
    for (auto &b : boards_) {
        out << center(std::to_string(b.ram_mb) + " MB", COL_BOARD) << "│";
    }
    out << '\n' << sep << '\n';

    // ── feature rows ──
    for (auto &feat : features_) {
        out << "│" << center(feat.label, COL_FEAT) << "│";
        for (auto &b : boards_) {
            auto it = b.features.find(feat.key);
            bool on = (it != b.features.end() && it->second);
            out << center(on ? "✔  YES" : "✘  no", COL_BOARD) << "│";
        }
        out << '\n';
    }

    out << bot << '\n';
}

// ─── filter ───────────────────────────────────────────────────────────────

std::vector<const Board*>
BoardMatrix::filter(std::vector<std::string> req) const
{
    std::vector<const Board*> result;
    for (auto &b : boards_) {
        bool match = std::all_of(req.begin(), req.end(),
            [&b](const std::string &key) {
                auto it = b.features.find(key);
                return it != b.features.end() && it->second;
            });
        if (match) result.push_back(&b);
    }
    return result;
}

} // namespace foobox

// ─── demo main ────────────────────────────────────────────────────────────

int main()
{
    using namespace foobox;

    BoardMatrix mx({
        {"gpu",   "GPU"  },
        {"nvme",  "NVMe" },
        {"wifi",  "WiFi" },
        {"bt",    "BT"   },
        {"can",   "CAN"  },
        {"tpm",   "TPM"  },
    });

    mx.add_board({
        "Lite", "foobox_lite_defconfig", "ARMv7", 256,
        {{"gpu",false},{"nvme",false},{"wifi",true},
         {"bt",true},{"can",false},{"tpm",false}}
    });
    mx.add_board({
        "Pro", "foobox_pro_defconfig", "AArch64", 512,
        {{"gpu",true},{"nvme",false},{"wifi",true},
         {"bt",true},{"can",false},{"tpm",true}}
    });
    mx.add_board({
        "Server", "foobox_server_defconfig", "AArch64", 2048,
        {{"gpu",true},{"nvme",true},{"wifi",false},
         {"bt",false},{"can",false},{"tpm",true}}
    });

    std::cout << "\n  FooBox Board Feature Matrix\n";
    std::cout << "  ============================\n\n";
    mx.render(std::cout);

    // Filter example
    std::cout << "\n  Boards with GPU + TPM:\n";
    for (auto *b : mx.filter({"gpu","tpm"}))
        std::cout << "    → " << b->name
                  << " (" << b->defconfig << ")\n";

    return 0;
}
```

### 8.3 Sample output

```
  FooBox Board Feature Matrix
  ============================

┌──────────┬──────────────┬──────────────┬──────────────┐
│ Feature  │    Lite      │     Pro      │    Server    │
│          │    ARMv7     │   AArch64    │   AArch64    │
│   RAM    │   256 MB     │   512 MB     │   2048 MB    │
├──────────┼──────────────┼──────────────┼──────────────┤
│  GPU     │   -  no      │   +  YES     │   +  YES     │
│  NVMe    │   -  no      │   -  no      │   +  YES     │
│  WiFi    │   +  YES     │   +  YES     │   -  no      │
│  BT      │   +  YES     │   +  YES     │   -  no      │
│  CAN     │   -  no      │   -  no      │   -  no      │
│  TPM     │   -  no      │   +  YES     │   +  YES     │
└──────────┴──────────────┴──────────────┴──────────────┘

  Boards with GPU + TPM:
    → Pro (foobox_pro_defconfig)
    → Server (foobox_server_defconfig)
```

---

## 9. Rust Example — Board HAL Dispatcher

A Rust library crate that selects the correct Hardware Abstraction Layer (HAL)
implementation at compile time based on board features injected from the
Buildroot defconfig via a `build.rs` script.

### 9.1 build.rs — reading Buildroot environment

Buildroot sets environment variables for packages via the `.mk` file.
The `build.rs` script translates them into Cargo `cfg` flags.

```rust
// build.rs — translate Buildroot env vars → Cargo cfg flags

fn main() {
    // Buildroot exports BR2_TARGET_GENERIC_HOSTNAME via the environment
    // when building packages that call cargo.
    let hostname = std::env::var("BR2_TARGET_GENERIC_HOSTNAME")
        .unwrap_or_else(|_| "unknown".to_string());

    println!("cargo:rustc-env=BOARD_NAME={}", hostname);

    // Feature flags from defconfig, passed via .mk as env vars
    if env_is_set("BOARD_HAS_GPU")  { println!("cargo:rustc-cfg=board_gpu");  }
    if env_is_set("BOARD_HAS_NVME") { println!("cargo:rustc-cfg=board_nvme"); }
    if env_is_set("BOARD_HAS_WIFI") { println!("cargo:rustc-cfg=board_wifi"); }
    if env_is_set("BOARD_HAS_TPM")  { println!("cargo:rustc-cfg=board_tpm");  }

    // Architecture shorthand
    let arch = std::env::var("ARCH").unwrap_or_else(|_| "unknown".to_string());
    match arch.as_str() {
        "arm"   => println!("cargo:rustc-cfg=board_arch_arm32"),
        "arm64" => println!("cargo:rustc-cfg=board_arch_arm64"),
        _       => {}
    }
}

fn env_is_set(key: &str) -> bool {
    matches!(
        std::env::var(key).as_deref(),
        Ok("1") | Ok("y") | Ok("yes") | Ok("true")
    )
}
```

### 9.2 src/hal/mod.rs — compile-time HAL selection

```rust
// src/hal/mod.rs

//! Hardware Abstraction Layer — board-specific implementation selected
//! at compile time by Cargo cfg flags set from the Buildroot defconfig.

pub mod gpu;
pub mod storage;
pub mod network;

/// Top-level board descriptor, fully resolved at compile time.
pub struct BoardHal {
    pub name:    &'static str,
    pub gpu:     gpu::GpuDriver,
    pub storage: storage::StorageDriver,
    pub network: network::NetworkDriver,
}

impl BoardHal {
    pub const fn new() -> Self {
        BoardHal {
            name:    env!("BOARD_NAME"),
            gpu:     gpu::GpuDriver::new(),
            storage: storage::StorageDriver::new(),
            network: network::NetworkDriver::new(),
        }
    }

    /// Print a human-readable capability summary (ASCII table).
    pub fn print_summary(&self) {
        let g = self.gpu.is_present();
        let s = self.storage.kind();
        let n = self.network.kind();

        println!("┌─────────────────────────────────────┐");
        println!("│  Board HAL — {:>20}    │", self.name);
        println!("├─────────────────────────────────────┤");
        println!("│  GPU driver   : {:>20} │",
                 if g { "Mali (active)" } else { "none" });
        println!("│  Storage      : {:>20} │", s);
        println!("│  Network      : {:>20} │", n);
        println!("└─────────────────────────────────────┘");
    }
}
```

### 9.3 src/hal/gpu.rs

```rust
// src/hal/gpu.rs — GPU HAL, selected by board_gpu cfg flag

pub struct GpuDriver {
    #[cfg(board_gpu)]
    _priv: (),
}

impl GpuDriver {
    pub const fn new() -> Self {
        GpuDriver {
            #[cfg(board_gpu)]
            _priv: (),
        }
    }

    pub fn is_present(&self) -> bool {
        cfg!(board_gpu)
    }

    /// Submit a render command buffer.
    /// On boards without GPU this is a compile-time no-op.
    #[allow(unused_variables)]
    pub fn submit(&self, cmd: &[u8]) -> Result<(), &'static str> {
        #[cfg(board_gpu)]
        {
            // Real Mali GPU submission would go here.
            println!("[GPU] submitting {} bytes via libmali", cmd.len());
            return Ok(());
        }
        #[cfg(not(board_gpu))]
        Err("no GPU on this board")
    }
}
```

### 9.4 src/hal/storage.rs

```rust
// src/hal/storage.rs — storage HAL

pub struct StorageDriver;

impl StorageDriver {
    pub const fn new() -> Self { StorageDriver }

    pub fn kind(&self) -> &'static str {
        #[cfg(board_nvme)]  { return "NVMe (PCIe)"; }
        #[cfg(not(board_nvme))] { "eMMC / SD" }
    }

    pub fn block_size(&self) -> usize {
        #[cfg(board_nvme)]      { 4096 }
        #[cfg(not(board_nvme))] { 512  }
    }
}
```

### 9.5 src/hal/network.rs

```rust
// src/hal/network.rs — network HAL

pub struct NetworkDriver;

impl NetworkDriver {
    pub const fn new() -> Self { NetworkDriver }

    pub fn kind(&self) -> &'static str {
        #[cfg(board_wifi)]      { "WiFi 802.11ac" }
        #[cfg(not(board_wifi))] { "GbE (wired)"   }
    }
}
```

### 9.6 src/main.rs — integration demo

```rust
// src/main.rs

mod hal;
use hal::BoardHal;

fn main() {
    let board = BoardHal::new();
    board.print_summary();

    // GPU path — compile-time conditional
    let dummy_cmd = [0u8; 64];
    match board.gpu.submit(&dummy_cmd) {
        Ok(())   => println!("[OK ] GPU submission succeeded"),
        Err(msg) => println!("[--] GPU skipped: {}", msg),
    }

    println!(
        "\n  Storage block size : {} bytes",
        board.storage.block_size()
    );
    println!(
        "  Network interface  : {}",
        board.network.kind()
    );
}
```

### 9.7 foobox-hal.mk — building the Rust crate with Buildroot

```makefile
################################################################################
# foobox-hal — Rust board HAL crate, built by Buildroot's cargo-package infra
################################################################################

FOOBOX_HAL_VERSION       = 1.0.0
FOOBOX_HAL_SITE          = $(BR2_EXTERNAL_FOOBOX_PATH)/package/foobox-hal
FOOBOX_HAL_SITE_METHOD   = local

# Pass defconfig-derived values as env vars so build.rs can read them
FOOBOX_HAL_CARGO_ENV = \
    BR2_TARGET_GENERIC_HOSTNAME="$(BR2_TARGET_GENERIC_HOSTNAME)" \
    ARCH="$(KERNEL_ARCH)"

ifeq ($(BR2_PACKAGE_LIBMALI),y)
FOOBOX_HAL_CARGO_ENV += BOARD_HAS_GPU=1
endif
ifeq ($(BR2_PACKAGE_NVME_CLI),y)
FOOBOX_HAL_CARGO_ENV += BOARD_HAS_NVME=1
endif
ifeq ($(BR2_PACKAGE_WPA_SUPPLICANT),y)
FOOBOX_HAL_CARGO_ENV += BOARD_HAS_WIFI=1
endif

$(eval $(cargo-package))
```

### 9.8 Output on foobox_pro (GPU + WiFi)

```
┌─────────────────────────────────────┐
│  Board HAL —           foobox-pro   │
├─────────────────────────────────────┤
│  GPU driver   :       Mali (active) │
│  Storage      :          eMMC / SD  │
│  Network      :        WiFi 802.11ac│
└─────────────────────────────────────┘
[OK ] GPU submission succeeded

  Storage block size : 512 bytes
  Network interface  : WiFi 802.11ac
```

### 9.9 Output on foobox_server (NVMe, no GPU, no WiFi)

```
┌─────────────────────────────────────┐
│  Board HAL —       foobox-server    │
├─────────────────────────────────────┤
│  GPU driver   :               none  │
│  Storage      :        NVMe (PCIe)  │
│  Network      :         GbE (wired) │
└─────────────────────────────────────┘
[--] GPU skipped: no GPU on this board

  Storage block size : 4096 bytes
  Network interface  : GbE (wired)
```

---

## 10. CI / Automation Scripts

### 10.1 GitLab CI — build all boards in parallel

```yaml
# .gitlab-ci.yml

variables:
  BUILDROOT_VERSION: "2024.02.3"
  BR2_EXT: "$CI_PROJECT_DIR"

stages:
  - build

.buildroot_template: &br_template
  image: debian:bookworm
  before_script:
    - apt-get update -qq
    - apt-get install -y --no-install-recommends
        make gcc g++ unzip bc libssl-dev
        python3 rsync wget cpio file
    - wget -q
        "https://buildroot.org/downloads/buildroot-${BUILDROOT_VERSION}.tar.gz"
    - tar -xf "buildroot-${BUILDROOT_VERSION}.tar.gz"
  artifacts:
    paths:
      - output/*/images/
    expire_in: 1 week

build:lite:
  <<: *br_template
  script:
    - make -C buildroot-${BUILDROOT_VERSION}
        O=$PWD/output/lite
        BR2_EXTERNAL=$BR2_EXT
        foobox_lite_defconfig
    - make -C buildroot-${BUILDROOT_VERSION}
        O=$PWD/output/lite -j$(nproc)

build:pro:
  <<: *br_template
  script:
    - make -C buildroot-${BUILDROOT_VERSION}
        O=$PWD/output/pro
        BR2_EXTERNAL=$BR2_EXT
        foobox_pro_defconfig
    - make -C buildroot-${BUILDROOT_VERSION}
        O=$PWD/output/pro -j$(nproc)

build:server:
  <<: *br_template
  script:
    - make -C buildroot-${BUILDROOT_VERSION}
        O=$PWD/output/server
        BR2_EXTERNAL=$BR2_EXT
        foobox_server_defconfig
    - make -C buildroot-${BUILDROOT_VERSION}
        O=$PWD/output/server -j$(nproc)
```

### 10.2 Shell helper — verify defconfigs are minimal

```bash
#!/usr/bin/env bash
# scripts/check_defconfigs.sh
# Ensures every defconfig in configs/ is already minimal (savedefconfig
# produces no diff). Run in CI after any config change.

set -euo pipefail

BUILDROOT=${BUILDROOT:-"$HOME/src/buildroot"}
BR2_EXT=$(cd "$(dirname "$0")/.." && pwd)
TMPOUT=$(mktemp -d)
trap 'rm -rf "$TMPOUT"' EXIT

EXIT_CODE=0

for defconfig in "$BR2_EXT"/configs/*_defconfig; do
    name=$(basename "$defconfig")
    echo "Checking $name ..."

    make -C "$BUILDROOT" O="$TMPOUT" BR2_EXTERNAL="$BR2_EXT" "$name" \
         >/dev/null 2>&1
    make -C "$BUILDROOT" O="$TMPOUT" \
         BR2_DEFCONFIG="$TMPOUT/.saved_defconfig" savedefconfig \
         >/dev/null 2>&1

    if ! diff -u "$defconfig" "$TMPOUT/.saved_defconfig" >/dev/null 2>&1; then
        echo "  FAIL: $name is not minimal!"
        echo "  Run: make O=<output> BR2_DEFCONFIG=$defconfig savedefconfig"
        diff -u "$defconfig" "$TMPOUT/.saved_defconfig" || true
        EXIT_CODE=1
    else
        echo "  OK"
    fi
done

exit "$EXIT_CODE"
```

---

## 11. Common Pitfalls

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │                       Common Pitfalls & Fixes                          │
  ├──────────────────────────────┬─────────────────────────────────────────┤
  │  Pitfall                     │  Fix                                    │
  ├──────────────────────────────┼─────────────────────────────────────────┤
  │ Committing full .config      │ Always use `make savedefconfig` first;  │
  │ instead of minimal defconfig │ commit only the _defconfig file.        │
  ├──────────────────────────────┼─────────────────────────────────────────┤
  │ Fragment order wrong         │ Merge from least- to most-specific;     │
  │ (board loses to common)      │ board.config must come LAST.            │
  ├──────────────────────────────┼─────────────────────────────────────────┤
  │ BR2_EXTERNAL not set in CI   │ Always pass BR2_EXTERNAL= explicitly;   │
  │                              │ the .br2-external.mk is not in git.     │
  ├──────────────────────────────┼─────────────────────────────────────────┤
  │ Absolute paths in defconfig  │ Use $(BR2_EXTERNAL_<NAME>_PATH) macros  │
  │                              │ so the tree is relocatable.             │
  ├──────────────────────────────┼─────────────────────────────────────────┤
  │ Sharing output/ between      │ Always use separate O= directories per  │
  │ multiple boards              │ board; never reuse output/.             │
  ├──────────────────────────────┼─────────────────────────────────────────┤
  │ debug.config leaks into      │ Never reference debug.config from any   │
  │ release image                │ _defconfig; merge it manually only.     │
  ├──────────────────────────────┼─────────────────────────────────────────┤
  │ External tree name clash     │ Use a unique NAME in external.desc;     │
  │ (two trees, same name)       │ Buildroot prefixes macros with it.      │
  └──────────────────────────────┴─────────────────────────────────────────┘
```

---

## 12. Summary

```
  ╔══════════════════════════════════════════════════════════════════════╗
  ║          Board Defconfigs & Multi-Board Repositories                 ║
  ║                       — At a Glance —                                ║
  ╠══════════════════════════════════════════════════════════════════════╣
  ║                                                                      ║
  ║  WHERE configs live                                                  ║
  ║  ─────────────────                                                   ║
  ║  BR2_EXTERNAL/configs/<board>_defconfig                              ║
  ║  Minimal diff from Buildroot defaults; committed to version control. ║
  ║                                                                      ║
  ║  FRAGMENTS                                                           ║
  ║  ─────────                                                           ║
  ║  Partial Kconfig files merged left-to-right (last wins).             ║
  ║  common.config  →  shared by all boards                              ║
  ║  gpu.config     →  opt-in for boards with GPU                        ║
  ║  debug.config   →  developer overlay, never in production            ║
  ║                                                                      ║
  ║  KEY COMMANDS                                                        ║
  ║  ────────────                                                        ║
  ║  make <board>_defconfig      Load a board config                     ║
  ║  make menuconfig             Tweak options interactively             ║
  ║  make savedefconfig          Write minimal diff back to _defconfig   ║
  ║  make linux-menuconfig       Edit kernel config                      ║
  ║  make olddefconfig           Resolve new symbols to defaults         ║
  ║                                                                      ║
  ║  PROGRAMMING PATTERNS                                                ║
  ║  ────────────────────                                                ║
  ║  C     → -D flags from .mk + runtime /etc/board-info verification    ║
  ║  C++   → Board feature matrix from compile-time constants            ║
  ║  Rust  → build.rs + cfg flags → zero-cost conditional HAL            ║
  ║                                                                      ║
  ║  GOLDEN RULES                                                        ║
  ║  ────────────                                                        ║
  ║  1. One output directory per board — never share O=                  ║
  ║  2. Use $(BR2_EXTERNAL_<NAME>_PATH) — never hardcode paths           ║
  ║  3. Run savedefconfig before every commit                            ║
  ║  4. Fragments: common first, board-specific last                     ║
  ║  5. debug.config: manual merge only, never in defconfigs             ║
  ╚══════════════════════════════════════════════════════════════════════╝
```

**Board Defconfigs & Multi-Board Repositories** is the cornerstone of
maintainable embedded Linux product families built with Buildroot. By keeping
each board's defconfig as a *minimal delta* stored in `BR2_EXTERNAL/configs/`,
layering reusable config fragments for shared settings, and consistently using
`make savedefconfig` to keep configs lean, a team can manage dozens of board
variants from a single external tree without configuration drift.

The C code example shows how compile-time `-D` flags from the `.mk` file and
runtime `/etc/board-info` cross-check can guard against flashing the wrong
image to the wrong hardware. The C++ feature matrix demonstrates a clean
object-oriented approach to visualising and querying board capabilities from
a shared manifest. The Rust HAL dispatcher illustrates how `build.rs` can
bridge Buildroot's Kconfig world into Cargo's `cfg` system, enabling
zero-cost, zero-dead-code conditional compilation across an entire product
family.

---

*End of Topic 25 — Board Defconfigs & Multi-Board Repositories*