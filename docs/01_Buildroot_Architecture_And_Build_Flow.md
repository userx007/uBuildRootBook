# 01. Buildroot Architecture & Build Flow

**Structure at a glance:**

- **Architecture overview** — ASCII block diagram of the Buildroot source tree feeding into the `output/` directory
- **8-stage pipeline** — Download → Extract → Patch → Configure → Build → Install Staging → Install Target → FS Image, each with ASCII flow diagrams
- **`output/` layout** — annotated directory tree distinguishing `build/`, `host/`, `staging/`, `target/`, and `images/`
- **`make` internals** — stamp files, infrastructure macros (`generic-package`, `cmake-package`, `cargo-package`), and all key targets
- **`BR2_*` variables** — architecture, toolchain, package, system, and image groups
- **C example** (`my-c-app`) — a polling sysinfo monitor using `utsname`/`sysinfo`, with full `.mk`, `Config.in`, and `Makefile`
- **C++ example** (`my-cpp-app`) — RAII `libcurl` wrapper using C++17 filesystem and smart pointers, with `pkg-config` integration
- **Rust example** (`my-rust-app`) — pure-`std` `/proc` parser with an ASCII memory bar chart, using the `cargo-package` infrastructure
- **Integration** — complete `BR2_EXTERNAL` tree layout, building all three together
- **Advanced hooks** — post-extract/build/install hooks, rootfs overlays, post-build and post-image shell scripts
- **Summary** — a full ASCII box summary of the entire flow, package types, and essential commands


> How Buildroot orchestrates downloads, extraction, patching, configuration,
> compilation and installation in stages; understanding `make` internals,
> `output/` directory layout and `BR2_*` variables.

---

## Table of Contents

1. [What Is Buildroot?](#1-what-is-buildroot)
2. [High-Level Architecture Overview](#2-high-level-architecture-overview)
3. [The Build Flow — Stage by Stage](#3-the-build-flow--stage-by-stage)
4. [The `output/` Directory Layout](#4-the-output-directory-layout)
5. [The `make` Internals](#5-the-make-internals)
6. [BR2_* Configuration Variables](#6-br2_-configuration-variables)
7. [C Programming Example — Custom Package](#7-c-programming-example--custom-package)
8. [C++ Programming Example — Custom Application](#8-c-programming-example--custom-application)
9. [Rust Programming Example — Custom Package](#9-rust-programming-example--custom-package)
10. [Integrating All Three into Buildroot](#10-integrating-all-three-into-buildroot)
11. [Advanced: Hooks, Overlays, and Post-Build Scripts](#11-advanced-hooks-overlays-and-post-build-scripts)
12. [Summary](#12-summary)

---

## 1. What Is Buildroot?

Buildroot is an open-source, Make-based framework for generating complete,
minimal **embedded Linux systems** — cross-compilation toolchain, root
filesystem, kernel, and bootloader — from source, entirely configured via a
single `menuconfig`-style interface.

Key design goals:

- **Simplicity** — one configuration file (`.config`), one `make` invocation.
- **Reproducibility** — pinned package versions, hash-checked downloads.
- **Minimalism** — no package manager on the target; everything is baked in.
- **Flexibility** — supports hundreds of target architectures (ARM, RISC-V,
  x86, MIPS, …) and thousands of packages.

Buildroot is **not** a distribution; it is a *build system* that produces a
distribution for you.

---

## 2. High-Level Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BUILDROOT SOURCE TREE                        │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌───────┐    │
│  │  arch/   │  │ package/ │  │  board/  │  │ boot/  │  │  fs/  │    │
│  │ (CPU     │  │ (3000+   │  │ (vendor  │  │(U-Boot │  │(ext4, │    │
│  │ options) │  │ pkgs)    │  │ configs) │  │ GRUB…) │  │ squash│    │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  └───────┘    │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────────────────────┐  │
│  │ toolchain│  │ system/  │  │            Kconfig / .config      │  │
│  │(internal │  │(init,    │  │  (BR2_ARCH, BR2_PACKAGE_*, etc.)  │  │
│  │ or       │  │ skeleton)│  └───────────────────────────────────┘  │
│  │ external)│  └──────────┘                                         │
│  └──────────┘                                                       │
└─────────────────────────────────────────────────────────────────────┘
                              │  make
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          output/                                    │
│                                                                     │
│   build/        host/        staging/      target/      images/     │
│  (per-pkg       (host        (sysroot)     (rootfs      (kernel,    │
│   build dirs)   tools)                    skeleton)     dtb, fs)    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Build Flow — Stage by Stage

Buildroot processes every package through a **fixed pipeline of stages**.
Each stage is represented by a Make target suffix:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    PACKAGE BUILD PIPELINE                                 │
│                                                                           │
│  ┌──────────┐    ┌───────────┐    ┌─────────┐    ┌──────────────────────┐ │
│  │ DOWNLOAD │───▶│  EXTRACT  │───▶│  PATCH  │───▶│   CONFIGURE          │ │
│  │          │    │           │    │         │    │ (cmake/autoconf/     │ │
│  │ wget/hg/ │    │ tar/unzip │    │ *.patch │    │  meson/custom)       │ │
│  │ git/svn  │    │ to build/ │    │ applied │    │                      │ │
│  └──────────┘    └───────────┘    └─────────┘    └──────────────────────┘ │
│        │                                                   │              │
│        ▼                                                   ▼              │
│  dl/pkg.tar.gz                                    build/pkg-ver/          │
│                                                   Makefile (configured)   │
│                                                                           │
│  ┌──────────┐    ┌───────────┐    ┌─────────────────────────────────────┐ │
│  │  BUILD   │───▶│  INSTALL  │───▶│    INSTALL TO TARGET / STAGING      │ │
│  │          │    │  (host    │    │                                     │ │
│  │ cross-   │    │  tools    │    │  staging/ ──▶ sysroot for next pkg  │ │
│  │ compile  │    │  only)    │    │  target/  ──▶ final rootfs          │ │
│  └──────────┘    └───────────┘    └─────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Download (`<pkg>-source`)

```
dl/
├── busybox-1.36.1.tar.bz2          ← tarball cached here
├── busybox-1.36.1.tar.bz2.br2      ← hash check file
├── linux-6.6.tar.xz
└── my-app-1.0.tar.gz
```

Variables controlling download:

| Variable | Meaning |
|---|---|
| `BR2_PRIMARY_SITE` | Primary download mirror |
| `BR2_BACKUP_SITE` | Fallback mirror |
| `BR2_DL_DIR` | Override for `dl/` location |
| `<PKG>_SOURCE` | Tarball filename |
| `<PKG>_SITE` | URL or local path |
| `<PKG>_SITE_METHOD` | `http`, `git`, `svn`, `local`, … |

### 3.2 Extract (`<pkg>-extract`)

Source archives are unpacked into `output/build/<pkg>-<version>/`.
Buildroot checks SHA256/SHA1/MD5 from `<pkg>.hash` before extraction.

```
output/build/
└── busybox-1.36.1/
    ├── Makefile
    ├── include/
    └── ...
```

### 3.3 Patch (`<pkg>-patch`)

Patches from `package/<pkg>/*.patch` are applied in lexicographic order
using `support/scripts/apply-patches.sh`.

```
package/busybox/
├── 0001-fix-musl-compat.patch
├── 0002-disable-suid.patch
└── busybox.mk
```

### 3.4 Configure (`<pkg>-configure`)

Buildroot delegates configuration to the package's own build system:

```
┌──────────────────────────────────────────────────────────┐
│                 CONFIGURE STRATEGIES                     │
│                                                          │
│  Autotools  ──▶  ./configure --host=arm-linux-...        │
│  CMake      ──▶  cmake -DCMAKE_TOOLCHAIN_FILE=...        │
│  Meson      ──▶  meson --cross-file=...                  │
│  Kconfig    ──▶  make ARCH=arm menuconfig (e.g. Linux)   │
│  Generic    ──▶  custom CONFIGURE_CMDS in .mk file       │
└──────────────────────────────────────────────────────────┘
```

### 3.5 Build (`<pkg>-build`)

Cross-compilation using the toolchain in `output/host/bin/`:

```
output/host/bin/
├── arm-buildroot-linux-musleabihf-gcc
├── arm-buildroot-linux-musleabihf-g++
├── arm-buildroot-linux-musleabihf-strip
└── arm-buildroot-linux-musleabihf-readelf
```

### 3.6 Install to Staging (`<pkg>-install-staging`)

Libraries, headers, and `.pc` files are installed into the sysroot at
`output/staging/`, which acts as the include/link root for subsequent packages.

```
output/staging/
├── usr/
│   ├── include/         ← headers for dependent packages
│   ├── lib/             ← .so / .a for linking
│   └── lib/pkgconfig/   ← pkg-config .pc files
```

### 3.7 Install to Target (`<pkg>-install-target`)

Binaries, libraries (stripped), and data files go into `output/target/`,
which becomes the root filesystem skeleton.

```
output/target/
├── bin/
├── etc/
├── lib/
├── usr/
│   ├── bin/
│   └── lib/
└── var/
```

### 3.8 Filesystem Image Generation

After all packages are installed, Buildroot constructs filesystem images
defined by `fs/<fstype>/`:

```
output/images/
├── rootfs.ext4
├── rootfs.squashfs
├── zImage
├── am335x-boneblack.dtb
└── sdcard.img
```

---

## 4. The `output/` Directory Layout

```
output/
│
├── build/                   ← per-package extracted + built source
│   ├── busybox-1.36.1/
│   ├── linux-6.6.0/
│   └── my-c-app-1.0/
│
├── host/                    ← host-side toolchain & tools
│   ├── bin/                 ← cross-compiler wrappers
│   ├── lib/                 ← host libraries
│   ├── share/
│   └── <arch>-buildroot-linux-<libc>/
│       └── sysroot/         ← target sysroot (symlinked to staging/)
│
├── staging/                 ← target sysroot (headers + libs for build)
│   ├── usr/include/
│   └── usr/lib/
│
├── target/                  ← rootfs skeleton (no host artifacts)
│   ├── bin/
│   ├── etc/
│   ├── lib/
│   └── usr/
│
└── images/                  ← final deliverables
    ├── rootfs.ext4
    ├── zImage
    └── u-boot.bin
```

> **Note:** `output/target/` is **not** a sysroot. It has no headers or
> static libraries. Never use it as `--sysroot` during compilation.
> Use `output/staging/` for that.

---

## 5. The `make` Internals

### 5.1 Entry Point

```
buildroot/
├── Makefile             ← top-level entry point
├── package/
│   └── pkg-generic.mk   ← generic package infrastructure
│   └── pkg-cmake.mk     ← CMake package infrastructure
│   └── pkg-autotools.mk ← Autotools infrastructure
└── support/
    └── scripts/
```

The top-level `Makefile` includes all package `.mk` files dynamically:

```makefile
# Simplified excerpt from Buildroot's Makefile
include $(sort $(wildcard package/*/*.mk))
```

### 5.2 Package Infrastructure Macros

Each package uses one of the infrastructure macros, which expand into
hundreds of Make rules:

```makefile
# package/my-c-app/my-c-app.mk
$(eval $(generic-package))      # custom build commands
$(eval $(autotools-package))    # ./configure && make
$(eval $(cmake-package))        # cmake + make
$(eval $(meson-package))        # meson + ninja
$(eval $(cargo-package))        # cargo build (Rust)
$(eval $(python-package))       # pip / setup.py
$(eval $(kernel-module))        # out-of-tree kernel module
```

### 5.3 Stamp Files

Buildroot uses empty **stamp files** to track completed stages and avoid
re-running them unnecessarily:

```
output/build/my-c-app-1.0/
├── .stamp_downloaded
├── .stamp_extracted
├── .stamp_patched
├── .stamp_configured
├── .stamp_built
├── .stamp_staging_installed
└── .stamp_target_installed
```

To force a rebuild of a single package:

```bash
make my-c-app-dirclean   # remove build dir + stamps
make my-c-app            # rebuild from scratch
```

### 5.4 Useful Top-Level Make Targets

| Target | Description |
|---|---|
| `make menuconfig` | Interactive configuration (ncurses) |
| `make` | Full build |
| `make <pkg>` | Build only `<pkg>` and its dependencies |
| `make <pkg>-rebuild` | Force rebuild of `<pkg>` |
| `make <pkg>-dirclean` | Remove `<pkg>` build directory |
| `make savedefconfig` | Save minimal `.config` to `BR2_DEFCONFIG` |
| `make legal-info` | Collect license information |
| `make graph-depends` | Generate dependency graph |
| `make show-targets` | List all enabled targets |

---

## 6. `BR2_*` Configuration Variables

All Buildroot configuration lives in `.config` as `BR2_` prefixed variables,
generated by Kconfig.

### 6.1 Architecture Variables

```kconfig
BR2_ARCH="arm"
BR2_arm=y
BR2_ENDIAN="LITTLE"
BR2_GCC_TARGET_ABI="aapcs-linux"
BR2_GCC_TARGET_CPU="cortex-a8"
BR2_GCC_TARGET_FPU="neon"
BR2_GCC_TARGET_FLOAT_ABI="hard"
```

### 6.2 Toolchain Variables

```kconfig
BR2_TOOLCHAIN_BUILDROOT=y          # use internal toolchain
# or
BR2_TOOLCHAIN_EXTERNAL=y           # use pre-built toolchain
BR2_TOOLCHAIN_EXTERNAL_PATH="/opt/arm-toolchain"

BR2_TOOLCHAIN_BUILDROOT_LIBC="musl"   # musl | glibc | uClibc-ng
BR2_KERNEL_HEADERS_5_15=y
BR2_BINUTILS_VERSION_2_41_X=y
BR2_GCC_VERSION_13_X=y
```

### 6.3 Package Variables

```kconfig
BR2_PACKAGE_BUSYBOX=y
BR2_PACKAGE_BUSYBOX_CONFIG="package/busybox/busybox-minimal.config"
BR2_PACKAGE_OPENSSL=y
BR2_PACKAGE_LIBCURL=y
BR2_PACKAGE_LIBCURL_CURL=y
BR2_PACKAGE_SQLITE=y
BR2_PACKAGE_MY_C_APP=y              # custom package
BR2_PACKAGE_MY_CPP_APP=y
BR2_PACKAGE_MY_RUST_APP=y
```

### 6.4 System Variables

```kconfig
BR2_TARGET_GENERIC_HOSTNAME="embedded-device"
BR2_TARGET_GENERIC_ISSUE="Embedded Linux 1.0"
BR2_TARGET_GENERIC_ROOT_PASSWD="root123"
BR2_SYSTEM_DHCP="eth0"
BR2_TARGET_TIMEZONE="Europe/Berlin"
BR2_ROOTFS_OVERLAY="board/myboard/rootfs-overlay"
BR2_ROOTFS_POST_BUILD_SCRIPT="board/myboard/post-build.sh"
BR2_ROOTFS_POST_IMAGE_SCRIPT="board/myboard/post-image.sh"
```

### 6.5 Image Variables

```kconfig
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
BR2_TARGET_ROOTFS_EXT2_SIZE="256M"
BR2_TARGET_ROOTFS_SQUASHFS=y
BR2_TARGET_ROOTFS_INITRAMFS=y
```

---

## 7. C Programming Example — Custom Package

### 7.1 Package Directory Structure

```
package/my-c-app/
├── Config.in
├── my-c-app.mk
└── my-c-app.hash
```

### 7.2 `Config.in`

```kconfig
config BR2_PACKAGE_MY_C_APP
    bool "my-c-app"
    depends on BR2_USE_WCHAR
    help
      A demonstration C application for Buildroot.

      Reads system information and prints it to stdout.
      Depends on wide-character support (wchar).
```

### 7.3 `my-c-app.mk`

```makefile
################################################################################
#
# my-c-app
#
################################################################################

MY_C_APP_VERSION        = 1.0.0
MY_C_APP_SITE           = $(BR2_EXTERNAL)/src/my-c-app
MY_C_APP_SITE_METHOD    = local
MY_C_APP_LICENSE        = MIT
MY_C_APP_LICENSE_FILES  = LICENSE

define MY_C_APP_BUILD_CMDS
    $(MAKE) CC="$(TARGET_CC)" \
            CFLAGS="$(TARGET_CFLAGS)" \
            LDFLAGS="$(TARGET_LDFLAGS)" \
            -C $(@D)
endef

define MY_C_APP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/my-c-app \
        $(TARGET_DIR)/usr/bin/my-c-app
    $(INSTALL) -D -m 0644 $(@D)/my-c-app.conf \
        $(TARGET_DIR)/etc/my-c-app.conf
endef

$(eval $(generic-package))
```

### 7.4 `my-c-app.hash`

```
sha256  b94d27b9934d3e08a52e52d7da7dabfac484efe04a8a2f046aaecfde3e3975cd  my-c-app-1.0.0.tar.gz
```

### 7.5 C Source: `src/my-c-app/main.c`

```c
/*
 * my-c-app/main.c
 * Buildroot C application example.
 * Demonstrates: sysinfo, environment reading, file I/O.
 */

#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/utsname.h>
#include <sys/sysinfo.h>
#include <time.h>
#include <errno.h>

#define CONFIG_FILE "/etc/my-c-app.conf"
#define VERSION     "1.0.0"

/* ─── configuration ────────────────────────────────────────────────────── */

typedef struct {
    char hostname[64];
    int  interval_sec;
    int  verbose;
} app_config_t;

static int config_load(const char *path, app_config_t *cfg)
{
    FILE *fp = fopen(path, "r");
    if (!fp) {
        fprintf(stderr, "warn: cannot open %s: %s\n", path, strerror(errno));
        /* defaults */
        snprintf(cfg->hostname, sizeof(cfg->hostname), "embedded");
        cfg->interval_sec = 5;
        cfg->verbose       = 0;
        return -1;
    }

    char line[128];
    while (fgets(line, sizeof(line), fp)) {
        /* strip newline */
        line[strcspn(line, "\n")] = '\0';
        if (line[0] == '#' || line[0] == '\0') continue;

        char key[64], val[64];
        if (sscanf(line, "%63[^=]=%63s", key, val) == 2) {
            if (strcmp(key, "hostname") == 0)
                snprintf(cfg->hostname, sizeof(cfg->hostname), "%s", val);
            else if (strcmp(key, "interval") == 0)
                cfg->interval_sec = atoi(val);
            else if (strcmp(key, "verbose") == 0)
                cfg->verbose = atoi(val);
        }
    }
    fclose(fp);
    return 0;
}

/* ─── system information ───────────────────────────────────────────────── */

static void print_sysinfo(const app_config_t *cfg)
{
    struct utsname uts;
    struct sysinfo si;
    time_t now = time(NULL);
    char tbuf[32];

    uname(&uts);
    sysinfo(&si);
    strftime(tbuf, sizeof(tbuf), "%Y-%m-%d %H:%M:%S", localtime(&now));

    printf("┌─────────────────────────────────────────┐\n");
    printf("│  my-c-app v%-30s│\n", VERSION);
    printf("├─────────────────────────────────────────┤\n");
    printf("│  Time     : %-28s│\n", tbuf);
    printf("│  Hostname : %-28s│\n", cfg->hostname);
    printf("│  Kernel   : %-28s│\n", uts.release);
    printf("│  Arch     : %-28s│\n", uts.machine);
    printf("│  Uptime   : %-24ld sec│\n", si.uptime);
    printf("│  RAM free : %-22lu MiB  │\n",
           (si.freeram * (unsigned long)si.mem_unit) / (1024 * 1024));
    printf("│  Load avg : %-5.2f %-5.2f %-5.2f          │\n",
           si.loads[0] / 65536.0,
           si.loads[1] / 65536.0,
           si.loads[2] / 65536.0);
    printf("└─────────────────────────────────────────┘\n");

    if (cfg->verbose) {
        printf("  [verbose] uts.nodename  = %s\n", uts.nodename);
        printf("  [verbose] uts.sysname   = %s\n", uts.sysname);
        printf("  [verbose] uts.version   = %s\n", uts.version);
        printf("  [verbose] si.totalram   = %lu MiB\n",
               (si.totalram * (unsigned long)si.mem_unit) / (1024*1024));
    }
}

/* ─── entry point ──────────────────────────────────────────────────────── */

int main(int argc, char *argv[])
{
    app_config_t cfg = {0};
    int one_shot = 0;

    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "-v") == 0) cfg.verbose = 1;
        if (strcmp(argv[i], "-1") == 0) one_shot = 1;
    }

    config_load(CONFIG_FILE, &cfg);

    if (one_shot) {
        print_sysinfo(&cfg);
        return EXIT_SUCCESS;
    }

    /* continuous monitoring loop */
    while (1) {
        print_sysinfo(&cfg);
        sleep((unsigned)cfg.interval_sec);
    }

    return EXIT_SUCCESS;
}
```

### 7.6 `src/my-c-app/Makefile`

```makefile
CC      ?= gcc
CFLAGS  ?= -Wall -Wextra -O2 -std=c99
TARGET   = my-c-app
SRCS     = main.c

$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^

.PHONY: clean
clean:
	rm -f $(TARGET)
```

---

## 8. C++ Programming Example — Custom Application

### 8.1 `package/my-cpp-app/my-cpp-app.mk`

```makefile
################################################################################
#
# my-cpp-app
#
################################################################################

MY_CPP_APP_VERSION       = 1.0.0
MY_CPP_APP_SITE          = $(BR2_EXTERNAL)/src/my-cpp-app
MY_CPP_APP_SITE_METHOD   = local
MY_CPP_APP_LICENSE       = Apache-2.0
MY_CPP_APP_LICENSE_FILES = LICENSE

MY_CPP_APP_DEPENDENCIES  = libcurl openssl

define MY_CPP_APP_BUILD_CMDS
    $(MAKE) CXX="$(TARGET_CXX)" \
            CXXFLAGS="$(TARGET_CXXFLAGS)" \
            LDFLAGS="$(TARGET_LDFLAGS)" \
            PKG_CONFIG="$(PKG_CONFIG_HOST_BINARY)" \
            PKG_CONFIG_LIBDIR="$(STAGING_DIR)/usr/lib/pkgconfig" \
            -C $(@D)
endef

define MY_CPP_APP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/my-cpp-app \
        $(TARGET_DIR)/usr/bin/my-cpp-app
endef

$(eval $(generic-package))
```

### 8.2 C++ Source: `src/my-cpp-app/main.cpp`

```cpp
/*
 * my-cpp-app/main.cpp
 * Buildroot C++ application example.
 * Demonstrates: RAII, modern C++17, pkg-config deps (libcurl).
 */

#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <vector>
#include <memory>
#include <stdexcept>
#include <algorithm>
#include <filesystem>
#include <curl/curl.h>

namespace fs = std::filesystem;
static constexpr const char *VERSION = "1.0.0";

/* ─── CURL RAII wrapper ────────────────────────────────────────────────── */

class CurlHandle {
public:
    CurlHandle() : handle_(curl_easy_init())
    {
        if (!handle_)
            throw std::runtime_error("curl_easy_init() failed");
    }
    ~CurlHandle() { curl_easy_cleanup(handle_); }

    /* no copy */
    CurlHandle(const CurlHandle &) = delete;
    CurlHandle &operator=(const CurlHandle &) = delete;

    CURL *get() const noexcept { return handle_; }

private:
    CURL *handle_;
};

/* ─── HTTP response buffer ─────────────────────────────────────────────── */

struct ResponseBuffer {
    std::string data;

    static size_t write_cb(char *ptr, size_t size,
                           size_t nmemb, void *userdata)
    {
        auto *buf = static_cast<ResponseBuffer *>(userdata);
        buf->data.append(ptr, size * nmemb);
        return size * nmemb;
    }
};

/* ─── file utilities ───────────────────────────────────────────────────── */

class FileWriter {
public:
    explicit FileWriter(const fs::path &path)
        : stream_(path, std::ios::out | std::ios::trunc)
    {
        if (!stream_.is_open())
            throw std::runtime_error("cannot open: " + path.string());
    }

    void write(std::string_view content) { stream_ << content; }

    ~FileWriter() { stream_.close(); }

private:
    std::ofstream stream_;
};

/* ─── package manifest ─────────────────────────────────────────────────── */

struct PackageEntry {
    std::string name;
    std::string version;
    std::string license;
};

static std::vector<PackageEntry> parse_br2_manifest(const fs::path &path)
{
    std::vector<PackageEntry> packages;
    std::ifstream f(path);
    if (!f.is_open()) return packages;

    std::string line;
    while (std::getline(f, line)) {
        std::istringstream ss(line);
        PackageEntry e;
        if (std::getline(ss, e.name,    ',') &&
            std::getline(ss, e.version, ',') &&
            std::getline(ss, e.license))
        {
            packages.push_back(std::move(e));
        }
    }
    return packages;
}

static void print_packages(const std::vector<PackageEntry> &pkgs)
{
    std::cout << "\n  Installed Packages (" << pkgs.size() << ")\n";
    std::cout << "  ┌────────────────────────┬───────────────┬──────────────┐\n";
    std::cout << "  │ Package                │ Version       │ License      │\n";
    std::cout << "  ├────────────────────────┼───────────────┼──────────────┤\n";
    for (const auto &p : pkgs) {
        std::printf("  │ %-22s │ %-13s │ %-12s │\n",
                    p.name.c_str(), p.version.c_str(), p.license.c_str());
    }
    std::cout << "  └────────────────────────┴───────────────┴──────────────┘\n";
}

/* ─── entry point ──────────────────────────────────────────────────────── */

int main(int argc, char *argv[])
{
    std::cout << "my-cpp-app v" << VERSION << "  (C++17, libcurl)\n";
    std::cout << std::string(40, '─') << "\n";

    /* 1. parse a BR2 manifest CSV if present */
    fs::path manifest = "/etc/br2-manifest.csv";
    if (argc > 1) manifest = argv[1];

    if (fs::exists(manifest)) {
        auto packages = parse_br2_manifest(manifest);
        print_packages(packages);
    } else {
        std::cerr << "  (no manifest at " << manifest << ")\n";
    }

    /* 2. demonstrate libcurl fetch (example only, requires network) */
    bool do_fetch = (argc > 2 && std::string(argv[2]) == "--fetch");
    if (do_fetch) {
        try {
            curl_global_init(CURL_GLOBAL_DEFAULT);
            CurlHandle curl;
            ResponseBuffer buf;

            const char *url = "http://example.com";
            curl_easy_setopt(curl.get(), CURLOPT_URL, url);
            curl_easy_setopt(curl.get(), CURLOPT_WRITEFUNCTION,
                             ResponseBuffer::write_cb);
            curl_easy_setopt(curl.get(), CURLOPT_WRITEDATA, &buf);
            curl_easy_setopt(curl.get(), CURLOPT_TIMEOUT, 10L);

            CURLcode rc = curl_easy_perform(curl.get());
            if (rc != CURLE_OK) {
                std::cerr << "curl error: " << curl_easy_strerror(rc) << "\n";
            } else {
                long http_code = 0;
                curl_easy_getinfo(curl.get(), CURLINFO_RESPONSE_CODE, &http_code);
                std::cout << "\nFetched " << url
                          << " → HTTP " << http_code
                          << " (" << buf.data.size() << " bytes)\n";

                FileWriter fw("/tmp/fetch_result.html");
                fw.write(buf.data);
                std::cout << "  saved to /tmp/fetch_result.html\n";
            }
            curl_global_cleanup();
        } catch (const std::exception &ex) {
            std::cerr << "exception: " << ex.what() << "\n";
            return EXIT_FAILURE;
        }
    }

    return EXIT_SUCCESS;
}
```

### 8.3 `src/my-cpp-app/Makefile`

```makefile
CXX      ?= g++
CXXFLAGS ?= -Wall -Wextra -O2 -std=c++17
LDFLAGS  += $(shell $(PKG_CONFIG) --libs libcurl)
CXXFLAGS += $(shell $(PKG_CONFIG) --cflags libcurl)
TARGET    = my-cpp-app
SRCS      = main.cpp

$(TARGET): $(SRCS)
	$(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS)

.PHONY: clean
clean:
	rm -f $(TARGET)
```

---

## 9. Rust Programming Example — Custom Package

Buildroot supports Rust packages through the `cargo-package` infrastructure,
which uses a bundled copy of Cargo and a precompiled Rust toolchain.

### 9.1 Prerequisites in `.config`

```kconfig
BR2_PACKAGE_HOST_RUSTC=y
BR2_PACKAGE_HOST_RUSTC_ARCH_SUPPORTS=y
BR2_TARGET_ARCH_HAS_ATOMIC=y   # required for most Rust targets
```

### 9.2 `package/my-rust-app/Config.in`

```kconfig
config BR2_PACKAGE_MY_RUST_APP
    bool "my-rust-app"
    depends on BR2_PACKAGE_HOST_RUSTC
    depends on !BR2_STATIC_LIBS   # Rust needs dynamic linking by default
    help
      A demonstration Rust application for Buildroot.
      Reads /proc/cpuinfo and /proc/meminfo and reports them.
```

### 9.3 `package/my-rust-app/my-rust-app.mk`

```makefile
################################################################################
#
# my-rust-app
#
################################################################################

MY_RUST_APP_VERSION       = 1.0.0
MY_RUST_APP_SITE          = $(BR2_EXTERNAL)/src/my-rust-app
MY_RUST_APP_SITE_METHOD   = local
MY_RUST_APP_LICENSE       = MIT
MY_RUST_APP_LICENSE_FILES = LICENSE
MY_RUST_APP_CARGO_ENV     = CARGO_NET_OFFLINE=true

# For offline builds, vendor dependencies into src tree:
# run `cargo vendor` and set .cargo/config.toml accordingly.

$(eval $(cargo-package))
```

> **Note on offline builds:** Buildroot's Cargo integration can work offline
> by pre-vendoring dependencies (`cargo vendor`) and committing them to the
> source tree. Set `CARGO_HOME` and `CARGO_NET_OFFLINE=true` accordingly.

### 9.4 Rust Source: `src/my-rust-app/src/main.rs`

```rust
//! my-rust-app/src/main.rs
//! Buildroot Rust application example.
//! Reads /proc/cpuinfo and /proc/meminfo and presents a summary.

use std::collections::HashMap;
use std::fs;
use std::io::{self, BufRead};
use std::process;

const VERSION: &str = "1.0.0";

// ─── /proc/meminfo parser ────────────────────────────────────────────────

#[derive(Debug, Default)]
struct MemInfo {
    total_kb:     u64,
    free_kb:      u64,
    available_kb: u64,
    buffers_kb:   u64,
    cached_kb:    u64,
}

impl MemInfo {
    fn from_proc() -> io::Result<Self> {
        let file = fs::File::open("/proc/meminfo")?;
        let mut map: HashMap<String, u64> = HashMap::new();

        for line in io::BufReader::new(file).lines() {
            let line = line?;
            let mut parts = line.split_whitespace();
            if let (Some(key), Some(val)) = (parts.next(), parts.next()) {
                let key = key.trim_end_matches(':').to_string();
                if let Ok(n) = val.parse::<u64>() {
                    map.insert(key, n);
                }
            }
        }

        Ok(MemInfo {
            total_kb:     *map.get("MemTotal").unwrap_or(&0),
            free_kb:      *map.get("MemFree").unwrap_or(&0),
            available_kb: *map.get("MemAvailable").unwrap_or(&0),
            buffers_kb:   *map.get("Buffers").unwrap_or(&0),
            cached_kb:    *map.get("Cached").unwrap_or(&0),
        })
    }

    fn used_kb(&self) -> u64 {
        self.total_kb.saturating_sub(self.available_kb)
    }

    /// Returns an ASCII bar chart of memory usage (width = 40 chars).
    fn usage_bar(&self) -> String {
        let width: u64 = 40;
        if self.total_kb == 0 {
            return "[                                        ]".to_string();
        }
        let filled = (self.used_kb() * width) / self.total_kb;
        let empty  = width - filled;
        format!("[{}{}]  {:.1}%",
            "█".repeat(filled as usize),
            "░".repeat(empty  as usize),
            100.0 * self.used_kb() as f64 / self.total_kb as f64)
    }
}

// ─── /proc/cpuinfo parser ────────────────────────────────────────────────

#[derive(Debug)]
struct CpuInfo {
    model_name: String,
    hardware:   String,
    num_cores:  usize,
    bogomips:   f64,
}

impl CpuInfo {
    fn from_proc() -> io::Result<Self> {
        let file = fs::File::open("/proc/cpuinfo")?;
        let mut model_name = String::from("unknown");
        let mut hardware   = String::from("unknown");
        let mut num_cores  = 0usize;
        let mut bogomips   = 0.0f64;

        for line in io::BufReader::new(file).lines() {
            let line = line?;
            if let Some((k, v)) = line.split_once(':') {
                let k = k.trim();
                let v = v.trim();
                match k {
                    "model name" | "Model name" => model_name = v.to_string(),
                    "Hardware"                  => hardware   = v.to_string(),
                    "processor"                 => num_cores += 1,
                    "BogoMIPS"                  => {
                        bogomips = v.parse().unwrap_or(0.0);
                    }
                    _ => {}
                }
            }
        }

        Ok(CpuInfo { model_name, hardware, num_cores, bogomips })
    }
}

// ─── display ─────────────────────────────────────────────────────────────

fn print_report(cpu: &CpuInfo, mem: &MemInfo) {
    println!("┌─────────────────────────────────────────────────────────┐");
    println!("│   my-rust-app v{:<41}│", VERSION);
    println!("├──────────────┬──────────────────────────────────────────┤");
    println!("│ CPU Model    │ {:<42}│", &cpu.model_name[..cpu.model_name.len().min(42)]);
    println!("│ Hardware     │ {:<42}│", &cpu.hardware[..cpu.hardware.len().min(42)]);
    println!("│ Cores        │ {:<42}│", cpu.num_cores);
    println!("│ BogoMIPS     │ {:<42.2}│", cpu.bogomips);
    println!("├──────────────┼──────────────────────────────────────────┤");
    println!("│ RAM Total    │ {:<38} MiB │", cpu_mib(mem.total_kb));
    println!("│ RAM Used     │ {:<38} MiB │", cpu_mib(mem.used_kb()));
    println!("│ RAM Free     │ {:<38} MiB │", cpu_mib(mem.free_kb));
    println!("│ RAM Buffers  │ {:<38} MiB │", cpu_mib(mem.buffers_kb));
    println!("│ RAM Cached   │ {:<38} MiB │", cpu_mib(mem.cached_kb));
    println!("├──────────────┴──────────────────────────────────────────┤");
    println!("│ Usage  {}  │", mem.usage_bar());
    println!("└─────────────────────────────────────────────────────────┘");
}

fn cpu_mib(kb: u64) -> u64 { kb / 1024 }

// ─── main ─────────────────────────────────────────────────────────────────

fn main() {
    let cpu = CpuInfo::from_proc().unwrap_or_else(|e| {
        eprintln!("error reading /proc/cpuinfo: {e}");
        process::exit(1);
    });

    let mem = MemInfo::from_proc().unwrap_or_else(|e| {
        eprintln!("error reading /proc/meminfo: {e}");
        process::exit(1);
    });

    print_report(&cpu, &mem);
}
```

### 9.5 `src/my-rust-app/Cargo.toml`

```toml
[package]
name        = "my-rust-app"
version     = "1.0.0"
edition     = "2021"
authors     = ["Embedded Developer <dev@example.com>"]
description = "Buildroot Rust application example"
license     = "MIT"

[[bin]]
name = "my-rust-app"
path = "src/main.rs"

# No external crate dependencies — pure std only.
# If you add crates, run `cargo vendor` and configure offline builds.
[dependencies]
```

---

## 10. Integrating All Three into Buildroot

### 10.1 `BR2_EXTERNAL` Layout

```
my-br2-external/
├── Config.in                  ← sources all package Config.in files
├── external.mk                ← sources all package .mk files
├── external.desc              ← name + description
│
├── board/
│   └── myboard/
│       ├── linux.config
│       ├── post-build.sh
│       ├── post-image.sh
│       └── rootfs-overlay/
│           └── etc/
│               └── my-c-app.conf
│
├── configs/
│   └── myboard_defconfig      ← saved defconfig
│
└── src/
    ├── my-c-app/
    │   ├── Makefile
    │   └── main.c
    ├── my-cpp-app/
    │   ├── Makefile
    │   └── main.cpp
    └── my-rust-app/
        ├── Cargo.toml
        └── src/main.rs
```

### 10.2 `Config.in`

```kconfig
mainmenu "My BR2 External"

source "$BR2_EXTERNAL_MY_BR2_EXTERNAL_PATH/package/my-c-app/Config.in"
source "$BR2_EXTERNAL_MY_BR2_EXTERNAL_PATH/package/my-cpp-app/Config.in"
source "$BR2_EXTERNAL_MY_BR2_EXTERNAL_PATH/package/my-rust-app/Config.in"
```

### 10.3 `external.mk`

```makefile
include $(sort $(wildcard $(BR2_EXTERNAL_MY_BR2_EXTERNAL_PATH)/package/*/*.mk))
```

### 10.4 `external.desc`

```
name: MY_BR2_EXTERNAL
desc: Custom packages: C, C++ and Rust demos
```

### 10.5 Building

```bash
# Step 1: Configure with the external tree
cd buildroot/
make BR2_EXTERNAL=/path/to/my-br2-external myboard_defconfig

# Step 2: Enable packages interactively (optional)
make menuconfig
#  → External options → [*] my-c-app
#                       [*] my-cpp-app
#                       [*] my-rust-app

# Step 3: Build
make -j$(nproc)

# Step 4: Inspect output
ls -lh output/images/
#  rootfs.ext4   ← contains /usr/bin/my-c-app, my-cpp-app, my-rust-app
#  zImage
#  sdcard.img
```

### 10.6 Rebuilding Only One Package

```bash
# C package
make my-c-app-dirclean && make my-c-app

# C++ package
make my-cpp-app-rebuild

# Rust package (also re-runs cargo build)
make my-rust-app-dirclean && make my-rust-app
```

---

## 11. Advanced: Hooks, Overlays, and Post-Build Scripts

### 11.1 Package Hooks (in `.mk` files)

```makefile
# Run after extraction
define MY_C_APP_POST_EXTRACT_HOOKS
    echo "Extracted to: $(@D)"
endef
MY_C_APP_POST_EXTRACT_HOOKS += MY_C_APP_POST_EXTRACT_HOOKS

# Run after build, before install
define MY_C_APP_POST_BUILD_HOOKS
    $(TARGET_STRIP) $(@D)/my-c-app
endef
MY_C_APP_POST_BUILD_HOOKS += MY_C_APP_POST_BUILD_HOOKS

# Run after target install
define MY_C_APP_POST_INSTALL_TARGET_HOOKS
    ln -sf /usr/bin/my-c-app $(TARGET_DIR)/usr/bin/sysmon
endef
MY_C_APP_POST_INSTALL_TARGET_HOOKS += MY_C_APP_POST_INSTALL_TARGET_HOOKS
```

### 11.2 Rootfs Overlay

Files placed in `BR2_ROOTFS_OVERLAY` are copied verbatim into `target/`
after all package installs, overwriting any existing files:

```
board/myboard/rootfs-overlay/
├── etc/
│   ├── my-c-app.conf        ← custom config
│   ├── inittab              ← custom init
│   └── network/interfaces   ← static IP config
└── usr/
    └── share/
        └── my-c-app/
            └── welcome.txt
```

### 11.3 Post-Build Script (`post-build.sh`)

```bash
#!/bin/bash
# board/myboard/post-build.sh
# Called by Buildroot after all packages are installed to target/,
# before filesystem images are created.
# $1 = path to output/target/

TARGET_DIR="$1"
BUILD_DATE=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
GIT_HASH=$(git -C "$BR2_EXTERNAL" rev-parse --short HEAD 2>/dev/null || echo "unknown")

# Inject build metadata into the target
cat > "${TARGET_DIR}/etc/br2-release" <<EOF
BUILD_DATE=${BUILD_DATE}
GIT_HASH=${GIT_HASH}
BR2_VERSION=$(cat ${BR2_EXTERNAL}/../.br2-version 2>/dev/null || echo "unknown")
PACKAGES=my-c-app,my-cpp-app,my-rust-app
EOF

echo "post-build.sh: wrote /etc/br2-release"
```

### 11.4 Post-Image Script (`post-image.sh`)

```bash
#!/bin/bash
# board/myboard/post-image.sh
# Called after filesystem images are generated.
# $1 = path to output/images/

IMAGES_DIR="$1"

# Create a combined SD card image
if [ -f "${IMAGES_DIR}/u-boot.bin" ] && \
   [ -f "${IMAGES_DIR}/rootfs.ext4" ]; then

    dd if=/dev/zero of="${IMAGES_DIR}/sdcard.img" bs=1M count=256
    # (partition table + u-boot + kernel + rootfs layout omitted for brevity)
    echo "post-image.sh: sdcard.img created"
fi
```

---

## 12. Summary

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              BUILDROOT ARCHITECTURE & BUILD FLOW — SUMMARY                   ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INPUT                          PIPELINE                   OUTPUT            ║
║  ─────                          ────────                   ──────            ║
║                                                                              ║
║  .config ──────────────▶  ┌─────────────────────┐                            ║
║  (BR2_* variables)         │  1. DOWNLOAD        │──▶ dl/                    ║
║                            │  2. EXTRACT         │──▶ build/<pkg>/           ║
║  package/*/                │  3. PATCH           │──▶ build/<pkg>/ (patched) ║
║  ├── Config.in             │  4. CONFIGURE       │──▶ build/<pkg>/Makefile   ║
║  ├── *.mk                  │  5. BUILD           │──▶ build/<pkg>/binary     ║
║  └── *.hash                │  6. INSTALL STAGING │──▶ staging/               ║
║                            │  7. INSTALL TARGET  │──▶ target/                ║
║  board/*/                  │  8. FS IMAGE        │──▶ images/                ║
║  ├── rootfs-overlay/       └─────────────────────┘                           ║
║  ├── post-build.sh                                                           ║
║  └── post-image.sh                                                           ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  KEY CONCEPTS                                                                ║
║                                                                              ║
║  • Stamp files  ──  avoid re-running completed stages                        ║
║  • Kconfig      ──  BR2_* variables drive every build decision               ║
║  • Sysroot      ──  output/staging/ for compilation, target/ for rootfs      ║
║  • Infra macros ──  $(eval $(cmake-package)) generates all Make rules        ║
║  • Hooks        ──  POST_EXTRACT / POST_BUILD / POST_INSTALL customisation   ║
║  • Overlays     ──  rootfs-overlay/ overlays target/ verbatim                ║
║  • BR2_EXTERNAL ──  out-of-tree packages & board configs                     ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  PACKAGE TYPES COVERED                                                       ║
║                                                                              ║
║  Language   Infrastructure      Toolchain var       Example target           ║
║  ────────   ──────────────      ─────────────       ──────────────           ║
║  C          generic-package     TARGET_CC           my-c-app                 ║
║  C++        generic-package     TARGET_CXX          my-cpp-app               ║
║  Rust       cargo-package       (Cargo cross-target) my-rust-app             ║
║  Autotools  autotools-package   TARGET_CC/CXX       openssl, busybox         ║
║  CMake      cmake-package       CMAKE_TOOLCHAIN     qt5, …                   ║
║  Meson      meson-package       --cross-file        glib, …                  ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  ESSENTIAL COMMANDS                                                          ║
║                                                                              ║
║  make menuconfig          — configure everything                             ║
║  make -j$(nproc)          — full build                                       ║
║  make <pkg>-dirclean      — force full rebuild of one package                ║
║  make <pkg>-rebuild       — rebuild without cleaning                         ║
║  make savedefconfig        — save minimal defconfig                          ║
║  make graph-depends        — visualise dependency graph                      ║
║  make legal-info           — collect all license texts                       ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

### Key Takeaways

1. **Every package flows through the same 8-stage pipeline** — download,
   extract, patch, configure, build, install-staging, install-target, image.
   Stamp files ensure idempotency.

2. **`output/staging/` is the sysroot** for cross-compilation. Never confuse
   it with `output/target/`, which is the final rootfs skeleton.

3. **`BR2_*` variables are the single source of truth.** They control
   architecture, toolchain, packages, and image formats — all set once in
   `.config` via `menuconfig`.

4. **Package infrastructure macros** (`cmake-package`, `autotools-package`,
   `cargo-package`, `generic-package`) expand into complete Make rule sets,
   keeping `.mk` files concise.

5. **C, C++, and Rust** are all first-class citizens. C and C++ use
   `TARGET_CC`/`TARGET_CXX` from the cross-toolchain; Rust uses
   `cargo-package` with `HOST_RUSTC` to cross-compile for the target triple.

6. **`BR2_EXTERNAL`** is the correct mechanism for out-of-tree packages and
   board configurations — never modify the Buildroot source tree itself.

7. **Hooks, overlays, and post-build/post-image scripts** provide clean
   extension points for board-specific customisation without modifying
   upstream Buildroot files.

---

*Document generated for the Buildroot Learning Series — Topic 01.*
*Buildroot version reference: 2024.02 LTS.*