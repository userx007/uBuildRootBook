# 04. Cross-Compilation Fundamentals in Buildroot

**Structure at a glance:**

1. **What Is Cross-Compilation** — host vs target concept, Buildroot build pipeline diagram
2. **CROSS_COMPILE** — the prefix variable, how it maps to gcc/g++/ld/ar/strip, how Buildroot sets it, and manual use
3. **The Sysroot** — what it is, directory tree layout, compiler flags, library search order
4. **pkg-config Wrappers** — why vanilla pkg-config is wrong, the wrapper script, correct Makefile usage, `.pc` file anatomy
5. **HOST_ vs TARGET_ Make Variables** — full variable reference table, package infrastructure, `host-` vs regular packages
6. **C Examples** — Hello World Makefile, OpenSSL via pkg-config, a full autotools `.mk` file
7. **C++ Examples** — C++17 with `std::filesystem`, CMake `.mk` file, CMake toolchain file
8. **Rust Examples** — target triples explained, `.cargo/config.toml`, Hello World, bindgen FFI, Cargo package `.mk`
9. **Common Pitfalls** — 7 pitfalls with fixes: host contamination, float ABI mismatch, C++ runtime, Rust link errors, autoconf cache poisoning, endianness, PATH ordering
10. **Summary** — big-picture ASCII diagram, key concepts table, cross-compilation checklist, quick-reference env vars


> **Topic:** `CROSS_COMPILE`, sysroot, `pkg-config` wrappers, `HOST_` vs `TARGET_` make variables,
> and common pitfalls when porting C++/Rust code.

---

## Table of Contents

1. [What Is Cross-Compilation?](#1-what-is-cross-compilation)
2. [The Toolchain and CROSS_COMPILE](#2-the-toolchain-and-cross_compile)
3. [The Sysroot](#3-the-sysroot)
4. [pkg-config Wrappers](#4-pkg-config-wrappers)
5. [HOST_ vs TARGET_ Make Variables](#5-host_-vs-target_-make-variables)
6. [C Code Examples](#6-c-code-examples)
7. [C++ Code Examples](#7-c-code-examples-1)
8. [Rust Cross-Compilation Examples](#8-rust-cross-compilation-examples)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Summary](#10-summary)

---

## 1. What Is Cross-Compilation?

Cross-compilation means **building executable code on one machine (the host) that runs on a different machine (the target)**. In Buildroot, the host is typically an x86-64 Linux workstation; the target is usually an embedded system running an ARM, MIPS, RISC-V, or similar processor.

```
  HOST MACHINE                          TARGET DEVICE
  ┌─────────────────────────┐           ┌──────────────────────┐
  │  x86-64 Linux           │           │  ARM Cortex-A53      │
  │  (your workstation)     │  compile  │  (Raspberry Pi, etc.)│
  │                         │ ───────►  │                      │
  │  gcc (host)             │  transfer │  ./my_app            │
  │  arm-linux-gnueabihf-gcc│           │  (runs here)         │
  └─────────────────────────┘           └──────────────────────┘
```

Buildroot automates the entire cross-compilation pipeline:

```
  Buildroot Build Pipeline
  ════════════════════════════════════════════════════════════════

  [Source Fetch]──►[Patch]──►[Configure]──►[Build]──►[Install]
       │                          │             │
       │                    (cross tools)  (cross tools)
       │
       ▼
  dl/          output/host/      output/target/    output/images/
  (tarballs)   (host tools)      (rootfs tree)     (final images)
```

---

## 2. The Toolchain and CROSS_COMPILE

### 2.1 The CROSS_COMPILE Variable

`CROSS_COMPILE` is a **prefix string** prepended to every cross-tool invocation. It tells the build system which compiler, linker, archiver, etc. to use for the target architecture.

```
  CROSS_COMPILE = "arm-linux-gnueabihf-"

  arm-linux-gnueabihf- gcc       → C compiler
  arm-linux-gnueabihf- g++       → C++ compiler
  arm-linux-gnueabihf- ld        → Linker
  arm-linux-gnueabihf- ar        → Archiver
  arm-linux-gnueabihf- strip     → Strip debuginfo
  arm-linux-gnueabihf- objdump   → Object dump
  arm-linux-gnueabihf- nm        → Symbol listing
  arm-linux-gnueabihf- ranlib    → Library indexer
```

### 2.2 Buildroot's Toolchain Architecture

```
  output/host/
  ├── bin/
  │   ├── arm-buildroot-linux-gnueabihf-gcc      ← cross-compiler
  │   ├── arm-buildroot-linux-gnueabihf-g++
  │   ├── arm-buildroot-linux-gnueabihf-ld
  │   ├── arm-buildroot-linux-gnueabihf-strip
  │   └── ... (other cross-tools)
  ├── arm-buildroot-linux-gnueabihf/
  │   └── sysroot/                               ← sysroot (see §3)
  │       ├── usr/include/
  │       ├── usr/lib/
  │       └── lib/
  └── usr/
      └── bin/                                   ← host-native tools
          ├── gcc  (host gcc, NOT cross)
          └── pkg-config (wrapper, see §4)
```

### 2.3 How Buildroot Sets CROSS_COMPILE

In `output/build/<package>-<version>/`, Buildroot sets the environment before invoking `make` or `cmake`:

```makefile
# Simplified excerpt from Buildroot's package infrastructure
CROSS_COMPILE = $(HOST_DIR)/bin/$(GNU_TARGET_NAME)-

# Exported to every package build:
export CC  = $(CROSS_COMPILE)gcc
export CXX = $(CROSS_COMPILE)g++
export LD  = $(CROSS_COMPILE)ld
export AR  = $(CROSS_COMPILE)ar
export AS  = $(CROSS_COMPILE)as
export STRIP = $(CROSS_COMPILE)strip
export RANLIB = $(CROSS_COMPILE)ranlib
export OBJCOPY = $(CROSS_COMPILE)objcopy
```

### 2.4 Manual Cross-Compile (outside Buildroot)

```bash
# Set variables manually for a standalone cross-compile
export CROSS_COMPILE="arm-linux-gnueabihf-"
export CC="${CROSS_COMPILE}gcc"
export CXX="${CROSS_COMPILE}g++"
export SYSROOT="/path/to/buildroot/output/host/arm-buildroot-linux-gnueabihf/sysroot"

# Compile with explicit sysroot
${CC} --sysroot="${SYSROOT}" -o hello hello.c

# Verify the output architecture
file hello
# hello: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), ...
```

---

## 3. The Sysroot

### 3.1 What Is a Sysroot?

The sysroot is a **directory tree that mirrors the target's root filesystem**, containing headers and libraries the cross-compiler uses when compiling code for the target. Without a correct sysroot, the compiler would find the host's `/usr/include` and `/usr/lib` — wrong architecture, wrong ABI.

```
  WRONG (no sysroot)           CORRECT (with sysroot)
  ══════════════════           ══════════════════════
  Host compiler looks at:      Cross-compiler looks at:

  /usr/include/stdio.h         /sysroot/usr/include/stdio.h
  (x86-64 headers)             (ARM headers, correct ABI)

  /usr/lib/libc.so             /sysroot/usr/lib/libc.so
  (x86-64 library)             (ARM library, correct ABI)
```

### 3.2 Sysroot Directory Structure

```
  output/host/arm-buildroot-linux-gnueabihf/sysroot/
  │
  ├── usr/
  │   ├── include/          ← Target system headers
  │   │   ├── stdio.h
  │   │   ├── stdlib.h
  │   │   ├── sys/
  │   │   └── ...
  │   ├── lib/              ← Target shared/static libraries
  │   │   ├── libc.so.6
  │   │   ├── libpthread.so.0
  │   │   ├── libstdc++.so.6
  │   │   └── pkgconfig/    ← .pc files for pkg-config
  │   └── bin/              ← Target binaries (if any)
  │
  ├── lib/                  ← Essential target libraries
  │   ├── ld-linux-armhf.so.3
  │   └── libc.so.6 -> ...
  │
  └── etc/                  ← Target configuration files
```

### 3.3 Compiler Flags for Sysroot

```bash
# Explicit sysroot flag
${CC} --sysroot=/path/to/sysroot -o output input.c

# Buildroot's cross-gcc is usually configured with sysroot baked in:
arm-buildroot-linux-gnueabihf-gcc -print-sysroot
# /home/user/buildroot/output/host/arm-buildroot-linux-gnueabihf/sysroot

# Check where the compiler looks for headers/libs
arm-buildroot-linux-gnueabihf-gcc -v 2>&1 | grep -E "SYSROOT|--with-sysroot"
```

### 3.4 Library Search Paths

```
  Cross-linker search order (simplified):

  1. -L flags on command line
         │
         ▼
  2. LIBRARY_PATH environment variable
         │
         ▼
  3. Sysroot lib directories:
     <sysroot>/lib
     <sysroot>/usr/lib
         │
         ▼
  4. Built-in linker paths (compiled into ld)

  ⚠  Host /usr/lib is NEVER searched when using --sysroot
```

---

## 4. pkg-config Wrappers

### 4.1 The Problem with Vanilla pkg-config

`pkg-config` is a tool that outputs compiler flags for a library (include paths, library names, linker flags). In cross-compilation, the system `pkg-config` reads `.pc` files from the **host** paths — giving completely wrong flags for the target.

```
  ┌─────────────────────────────────────────────────────────┐
  │  pkg-config --cflags libssl         (system pkg-config) │
  │                                                         │
  │  Returns: -I/usr/include/openssl    ← HOST path! WRONG  │
  │           -L/usr/lib/x86_64-...    ← HOST arch! WRONG   │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │  pkg-config --cflags libssl         (Buildroot wrapper) │
  │                                                         │
  │  Returns: -I/sysroot/usr/include    ← TARGET path! OK   │
  │           -L/sysroot/usr/lib        ← TARGET arch! OK   │
  └─────────────────────────────────────────────────────────┘
```

### 4.2 Buildroot's pkg-config Wrapper

Buildroot installs a wrapper script at `output/host/bin/pkg-config`:

```bash
#!/bin/sh
# Buildroot's pkg-config wrapper (simplified)

SYSROOT="/path/to/output/host/arm-buildroot-linux-gnueabihf/sysroot"

# Override search path to target's .pc files only
export PKG_CONFIG_SYSROOT_DIR="${SYSROOT}"
export PKG_CONFIG_LIBDIR="${SYSROOT}/usr/lib/pkgconfig:${SYSROOT}/usr/share/pkgconfig"

# Unset any host paths that would pollute results
unset PKG_CONFIG_PATH

exec /usr/bin/pkg-config "$@"
```

### 4.3 Using pkg-config Correctly in Package Makefiles

```makefile
# In a Buildroot .mk file — CORRECT usage
MY_PACKAGE_CONF_ENV = \
    PKG_CONFIG="$(PKG_CONFIG_HOST_BINARY)" \
    PKG_CONFIG_SYSROOT_DIR="$(STAGING_DIR)"

# In your own Makefile outside Buildroot — set these:
export PKG_CONFIG_SYSROOT_DIR = $(SYSROOT)
export PKG_CONFIG_LIBDIR      = $(SYSROOT)/usr/lib/pkgconfig
export PKG_CONFIG_PATH        =   # clear it!

CFLAGS  += $(shell pkg-config --cflags openssl)
LDFLAGS += $(shell pkg-config --libs   openssl)
```

### 4.4 pkg-config .pc File Anatomy

```
  File: /sysroot/usr/lib/pkgconfig/openssl.pc
  ────────────────────────────────────────────
  prefix=/usr                    ← will be prefixed with sysroot
  exec_prefix=${prefix}
  libdir=${exec_prefix}/lib
  includedir=${prefix}/include

  Name: OpenSSL
  Version: 3.0.8
  Requires: libssl libcrypto
  Libs: -L${libdir} -lssl -lcrypto
  Cflags: -I${includedir}

  ┌────────────────────────────────────────────────────────┐
  │ With PKG_CONFIG_SYSROOT_DIR=/sysroot, pkg-config       │
  │ prepends the sysroot to ${libdir} and ${includedir},   │
  │ yielding correct cross-compile flags automatically.    │
  └────────────────────────────────────────────────────────┘
```

---

## 5. HOST_ vs TARGET_ Make Variables

### 5.1 The Fundamental Distinction

Buildroot distinguishes between software built **for the host** (tools that run during the build) and software built **for the target** (code that runs on the embedded device).

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   Buildroot Variable Namespaces             │
  │                                                             │
  │  HOST_*                          TARGET_*                   │
  │  ──────────────────              ────────────────────────   │
  │  Runs on: build machine          Runs on: embedded device   │
  │  Compiler: host gcc              Compiler: cross-gcc        │
  │  Output: output/host/            Output: output/target/     │
  │  Example: HOST_OPENSSL           Example: TARGET_OPENSSL    │
  │                                                             │
  │  Used by: code generators,       Used by: final rootfs,     │
  │  build tools, cmake,             application code,          │
  │  meson, dtc, mkimage             libraries, daemons         │
  └─────────────────────────────────────────────────────────────┘
```

### 5.2 Core Make Variables Reference

```makefile
# ── Compilers ───────────────────────────────────────────────────
HOST_CC          # Host C compiler   (gcc for build machine)
HOST_CXX         # Host C++ compiler (g++ for build machine)
TARGET_CC        # Target C compiler   (arm-linux-...-gcc)
TARGET_CXX       # Target C++ compiler (arm-linux-...-g++)

# ── Flags ───────────────────────────────────────────────────────
HOST_CFLAGS      # Flags for host compilation
HOST_CXXFLAGS    # Flags for host C++ compilation
HOST_LDFLAGS     # Flags for host linking
TARGET_CFLAGS    # Flags for target compilation
TARGET_CXXFLAGS  # Flags for target C++ compilation
TARGET_LDFLAGS   # Flags for target linking

# ── Directories ─────────────────────────────────────────────────
HOST_DIR         # output/host   — host tools live here
STAGING_DIR      # output/staging (symlink to sysroot)
TARGET_DIR       # output/target  — the rootfs being built
BINARIES_DIR     # output/images  — final images

# ── Paths ───────────────────────────────────────────────────────
HOST_MAKE_ENV    # Environment variables for host builds
TARGET_MAKE_ENV  # Environment variables for target builds
```

### 5.3 Package Infrastructure Variables

```makefile
# In a package .mk file (e.g., package/myapp/myapp.mk):

MY_APP_VERSION  = 1.2.3
MY_APP_SOURCE   = myapp-$(MY_APP_VERSION).tar.gz
MY_APP_SITE     = https://example.com/releases

# ── Configure-phase environment ─────────────────────────────────
MY_APP_CONF_ENV = \
    CC="$(TARGET_CC)"             \
    CXX="$(TARGET_CXX)"          \
    LD="$(TARGET_LD)"            \
    AR="$(TARGET_AR)"            \
    STRIP="$(TARGET_STRIP)"      \
    CFLAGS="$(TARGET_CFLAGS)"    \
    CXXFLAGS="$(TARGET_CXXFLAGS)"\
    LDFLAGS="$(TARGET_LDFLAGS)"  \
    PKG_CONFIG_SYSROOT_DIR="$(STAGING_DIR)"

# ── Build-phase Make options ────────────────────────────────────
MY_APP_MAKE_OPTS = \
    CROSS_COMPILE="$(TARGET_CROSS)" \
    PREFIX="/usr"

$(eval $(autotools-package))   # or cmake-package, generic-package, etc.
```

### 5.4 HOST package vs TARGET package

```
  HOST package (host-mypkg)            TARGET package (mypkg)
  ─────────────────────────            ──────────────────────

  $(eval $(host-autotools-package))    $(eval $(autotools-package))
        │                                     │
        ▼                                     ▼
  Uses HOST_CC, HOST_CXX            Uses TARGET_CC, TARGET_CXX
  Installs to $(HOST_DIR)           Installs to $(STAGING_DIR)
                                    and $(TARGET_DIR)

  Example: host-cmake, host-pkgconf  Example: busybox, openssl, ffmpeg
```

---

## 6. C Code Examples

### 6.1 Simple Hello World — Makefile for Cross-Compilation

```c
/* src/hello.c */
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    printf("Hello from target architecture!\n");

#if defined(__ARM_ARCH)
    printf("Running on ARM (version %d)\n", __ARM_ARCH);
#elif defined(__riscv)
    printf("Running on RISC-V (%d-bit)\n", __riscv_xlen);
#elif defined(__mips__)
    printf("Running on MIPS\n");
#else
    printf("Running on unknown architecture\n");
#endif

    return EXIT_SUCCESS;
}
```

```makefile
# Makefile — cross-compilation aware
CROSS_COMPILE ?= arm-linux-gnueabihf-
CC             = $(CROSS_COMPILE)gcc
STRIP          = $(CROSS_COMPILE)strip

SYSROOT       ?= /path/to/buildroot/output/staging
CFLAGS        += --sysroot=$(SYSROOT) -Wall -Wextra -O2
LDFLAGS       += --sysroot=$(SYSROOT)

TARGET = hello
SRC    = src/hello.c

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(SRC)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

strip: $(TARGET)
	$(STRIP) $(TARGET)

clean:
	rm -f $(TARGET)
```

### 6.2 Using a Library via pkg-config

```c
/* src/tls_client.c — uses OpenSSL, found via pkg-config */
#include <stdio.h>
#include <openssl/ssl.h>
#include <openssl/err.h>

int main(void)
{
    SSL_CTX *ctx;

    /* OpenSSL 3.x: OPENSSL_init_ssl replaces SSL_library_init */
    OPENSSL_init_ssl(0, NULL);
    OPENSSL_init_crypto(OPENSSL_INIT_LOAD_CRYPTO_STRINGS, NULL);

    ctx = SSL_CTX_new(TLS_client_method());
    if (!ctx) {
        ERR_print_errors_fp(stderr);
        return 1;
    }

    printf("OpenSSL version: %s\n", OpenSSL_version(OPENSSL_VERSION));
    SSL_CTX_free(ctx);
    return 0;
}
```

```makefile
# Makefile using pkg-config wrapper correctly
CROSS_COMPILE ?= arm-linux-gnueabihf-
CC             = $(CROSS_COMPILE)gcc
SYSROOT        = /path/to/buildroot/output/staging

# Point pkg-config at the TARGET's .pc files
export PKG_CONFIG_SYSROOT_DIR = $(SYSROOT)
export PKG_CONFIG_LIBDIR      = $(SYSROOT)/usr/lib/pkgconfig
export PKG_CONFIG_PATH        =

CFLAGS  += --sysroot=$(SYSROOT) $(shell pkg-config --cflags openssl)
LDFLAGS += --sysroot=$(SYSROOT) $(shell pkg-config --libs openssl)

all: tls_client

tls_client: src/tls_client.c
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)
```

### 6.3 Buildroot Package .mk File (autotools)

```makefile
# package/myapp/myapp.mk
################################################################################
#
# myapp
#
################################################################################

MYAPP_VERSION = 2.1.0
MYAPP_SITE    = https://example.com/releases
MYAPP_SOURCE  = myapp-$(MYAPP_VERSION).tar.xz

MYAPP_LICENSE        = MIT
MYAPP_LICENSE_FILES  = COPYING

# Dependencies on other Buildroot packages
MYAPP_DEPENDENCIES = libcurl openssl zlib

# Pass extra flags to configure
MYAPP_CONF_OPTS = \
    --enable-tls       \
    --disable-tests    \
    --with-zlib

# Extra environment for the configure step
MYAPP_CONF_ENV = \
    ac_cv_func_malloc_0_nonnull=yes  \
    ac_cv_func_realloc_0_nonnull=yes

# Install target binary only (no dev files to staging)
define MYAPP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/src/myapp \
        $(TARGET_DIR)/usr/bin/myapp
endef

$(eval $(autotools-package))
```

---

## 7. C++ Code Examples

### 7.1 Cross-Compiling C++ with the Standard Library

```cpp
// src/app.cpp — C++17 application
#include <iostream>
#include <vector>
#include <algorithm>
#include <string_view>
#include <filesystem>   // C++17 — requires libstdc++ with fs support

namespace fs = std::filesystem;

class DeviceInfo {
public:
    explicit DeviceInfo(std::string_view path) : root_(path) {}

    void listProcs() const {
        std::vector<std::string> procs;
        for (auto& entry : fs::directory_iterator("/proc")) {
            if (fs::is_directory(entry) &&
                std::isdigit(entry.path().filename().string()[0])) {
                procs.push_back(entry.path().filename().string());
            }
        }
        std::sort(procs.begin(), procs.end(),
                  [](const auto& a, const auto& b) {
                      return std::stoi(a) < std::stoi(b);
                  });
        std::cout << "Running PIDs: ";
        for (const auto& p : procs) std::cout << p << " ";
        std::cout << "\n";
    }

private:
    std::string root_;
};

int main() {
    std::cout << "C++17 cross-compiled application\n";
    DeviceInfo dev("/");
    dev.listProcs();
    return 0;
}
```

```makefile
# Makefile for C++17 cross-compilation
CROSS_COMPILE ?= arm-linux-gnueabihf-
CXX            = $(CROSS_COMPILE)g++
STRIP          = $(CROSS_COMPILE)strip
SYSROOT        = /path/to/buildroot/output/staging

CXXFLAGS  = --sysroot=$(SYSROOT) -std=c++17 -Wall -Wextra -O2
CXXFLAGS += -march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard
LDFLAGS   = --sysroot=$(SYSROOT) -lstdc++fs   # link filesystem on older gcc

TARGET = app

all: $(TARGET)

$(TARGET): src/app.cpp
	$(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS)
	$(STRIP) $@

clean:
	rm -f $(TARGET)
```

### 7.2 Buildroot .mk for a CMake C++ Package

```makefile
# package/mycpplib/mycpplib.mk
MYCPPLIB_VERSION    = 1.0.0
MYCPPLIB_SITE       = https://github.com/example/mycpplib/archive
MYCPPLIB_SOURCE     = v$(MYCPPLIB_VERSION).tar.gz
MYCPPLIB_LICENSE    = Apache-2.0
MYCPPLIB_INSTALL_STAGING = YES   # install headers + libs to staging

MYCPPLIB_CONF_OPTS = \
    -DCMAKE_BUILD_TYPE=Release    \
    -DBUILD_SHARED_LIBS=ON        \
    -DBUILD_TESTS=OFF             \
    -DCMAKE_CXX_STANDARD=17

# Buildroot's cmake-package infrastructure automatically sets:
#   CMAKE_TOOLCHAIN_FILE (with cross-compiler, sysroot, etc.)
#   CMAKE_INSTALL_PREFIX=/usr
#   CMAKE_STAGING_PREFIX=$(STAGING_DIR)/usr

$(eval $(cmake-package))
```

### 7.3 CMake Toolchain File (for use outside Buildroot)

```cmake
# arm-linux-gnueabihf.cmake — CMake toolchain file
# Usage: cmake -DCMAKE_TOOLCHAIN_FILE=arm-linux-gnueabihf.cmake ..

set(CMAKE_SYSTEM_NAME      Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

# The cross-compiler prefix
set(CROSS_COMPILE "arm-linux-gnueabihf-")

set(CMAKE_C_COMPILER   "${CROSS_COMPILE}gcc")
set(CMAKE_CXX_COMPILER "${CROSS_COMPILE}g++")
set(CMAKE_STRIP        "${CROSS_COMPILE}strip")
set(CMAKE_AR           "${CROSS_COMPILE}ar")

# Sysroot — point to Buildroot's staging directory
set(SYSROOT "/path/to/buildroot/output/staging")
set(CMAKE_SYSROOT "${SYSROOT}")

# Search paths: target headers/libs only, no host contamination
set(CMAKE_FIND_ROOT_PATH "${SYSROOT}")
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)   # find_program → host
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)    # find_library → target
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)    # find_path    → target
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)    # find_package → target

# pkg-config
set(ENV{PKG_CONFIG_SYSROOT_DIR} "${SYSROOT}")
set(ENV{PKG_CONFIG_LIBDIR}
    "${SYSROOT}/usr/lib/pkgconfig:${SYSROOT}/usr/share/pkgconfig")
```

---

## 8. Rust Cross-Compilation Examples

### 8.1 Rust Target Triples vs GNU Triplets

```
  GNU toolchain prefix:        arm-linux-gnueabihf-
  Rust target triple:          armv7-unknown-linux-gnueabihf

  Mapping:
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Architecture   OS      ABI/Vendor      Rust Triple                  │
  │  ───────────    ──      ──────────      ──────────────────────────   │
  │  armv7          linux   gnueabihf  →  armv7-unknown-linux-gnueabihf  │
  │  aarch64        linux   gnu        →  aarch64-unknown-linux-gnu      │
  │  riscv64gc      linux   gnu        →  riscv64gc-unknown-linux-gnu    │
  │  mipsel         linux   gnu        →  mipsel-unknown-linux-gnu       │
  │  x86_64         linux   gnu        →  x86_64-unknown-linux-gnu (host)│
  └──────────────────────────────────────────────────────────────────────┘
```

### 8.2 Configuring Cargo for Cross-Compilation

```toml
# .cargo/config.toml  (project-level or ~/.cargo/config.toml)

[target.armv7-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc"
ar     = "arm-linux-gnueabihf-ar"

# Environment for the target
[target.armv7-unknown-linux-gnueabihf.env]
PKG_CONFIG_SYSROOT_DIR  = "/path/to/buildroot/output/staging"
PKG_CONFIG_LIBDIR       = "/path/to/buildroot/output/staging/usr/lib/pkgconfig"
PKG_CONFIG_ALLOW_CROSS  = "1"

[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
```

### 8.3 Hello World in Rust — Cross-Compiled

```rust
// src/main.rs
use std::env;

fn main() {
    // Detect architecture at compile time
    let arch = if cfg!(target_arch = "arm") {
        "ARM 32-bit"
    } else if cfg!(target_arch = "aarch64") {
        "ARM 64-bit (AArch64)"
    } else if cfg!(target_arch = "riscv64") {
        "RISC-V 64-bit"
    } else {
        "Unknown"
    };

    println!("Hello from Rust on {}!", arch);
    println!("Target OS: {}", env::consts::OS);
    println!("Target family: {}", env::consts::FAMILY);
}
```

```bash
# Install the cross-compilation target
rustup target add armv7-unknown-linux-gnueabihf

# Build for target
cargo build --release --target armv7-unknown-linux-gnueabihf

# Verify
file target/armv7-unknown-linux-gnueabihf/release/hello
# hello: ELF 32-bit LSB pie executable, ARM, EABI5 ...
```

### 8.4 Rust with C Library Bindings (bindgen + cross)

```rust
// build.rs — build script for FFI bindings
use std::env;
use std::path::PathBuf;

fn main() {
    let sysroot = env::var("SYSROOT")
        .unwrap_or_else(|_| "/path/to/buildroot/output/staging".to_string());

    // Tell cargo to link against libssl
    println!("cargo:rustc-link-lib=ssl");
    println!("cargo:rustc-link-lib=crypto");
    println!("cargo:rustc-link-search=native={}/usr/lib", sysroot);
    println!("cargo:rerun-if-changed=wrapper.h");

    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        // Point bindgen at the sysroot headers
        .clang_arg(format!("--sysroot={}", sysroot))
        .clang_arg(format!("-I{}/usr/include", sysroot))
        .clang_arg("--target=arm-linux-gnueabihf")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks))
        .generate()
        .expect("Unable to generate bindings");

    let out_path = PathBuf::from(env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings!");
}
```

```rust
// src/lib.rs — using the generated FFI bindings
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));

pub fn openssl_version() -> String {
    unsafe {
        let ver_ptr = OpenSSL_version(OPENSSL_VERSION as i32);
        std::ffi::CStr::from_ptr(ver_ptr)
            .to_string_lossy()
            .into_owned()
    }
}
```

### 8.5 Buildroot Package for a Rust Binary

```makefile
# package/myrust/myrust.mk
################################################################################
#
# myrust — a Rust application built by Buildroot
#
################################################################################

MYRUST_VERSION  = 1.0.0
MYRUST_SITE     = https://github.com/example/myrust
MYRUST_SITE_METHOD = git

MYRUST_LICENSE       = MIT
MYRUST_LICENSE_FILES = LICENSE

# Tell the Rust package infrastructure which Cargo features to enable
MYRUST_CARGO_ENV = \
    PKG_CONFIG_ALLOW_CROSS=1 \
    PKG_CONFIG_SYSROOT_DIR="$(STAGING_DIR)" \
    PKG_CONFIG_LIBDIR="$(STAGING_DIR)/usr/lib/pkgconfig" \
    OPENSSL_DIR="$(STAGING_DIR)/usr" \
    OPENSSL_LIB_DIR="$(STAGING_DIR)/usr/lib" \
    OPENSSL_INCLUDE_DIR="$(STAGING_DIR)/usr/include"

MYRUST_DEPENDENCIES = openssl

$(eval $(cargo-package))
```

---

## 9. Common Pitfalls

### 9.1 Pitfall Map Overview

```
  Cross-Compilation Common Pitfalls
  ═══════════════════════════════════════════════════════════════

  ┌─────────────────────┐  ┌──────────────────────┐  ┌─────────────────────┐
  │  HOST contamination │  │  ABI mismatch        │  │  Hard-coded paths   │
  │  ─────────────────  │  │  ──────────────────  │  │  ─────────────────  │
  │  Wrong pkg-config   │  │  float-abi mismatch  │  │  /usr/lib in .pc    │
  │  Host headers used  │  │  soft vs hard FP     │  │  Absolute sysroot   │
  │  Host libs linked   │  │  endianness error    │  │  paths in Makefile  │
  └─────────────────────┘  └──────────────────────┘  └─────────────────────┘
         │                          │                          │
         └──────────────────────────┼──────────────────────────┘
                                    │
                     ┌──────────────▼───────────────┐
                     │  Symptoms / Errors           │
                     │  ─────────────────────────   │
                     │  "cannot execute binary"     │
                     │  "Exec format error"         │
                     │  Segfaults at runtime        │
                     │  Linker: wrong architecture  │
                     │  "undefined reference"       │
                     └──────────────────────────────┘
```

### 9.2 Pitfall 1 — Host Library Contamination

```bash
# ❌ WRONG: Host pkg-config returns host paths
$ pkg-config --libs openssl
-L/usr/lib/x86_64-linux-gnu -lssl -lcrypto    # host x86-64! 

# ❌ WRONG: Mixing host and target flags
CC=arm-linux-gnueabihf-gcc \
    $(pkg-config --cflags openssl) \   # ← host include path!
    -o myapp myapp.c

# ✅ CORRECT: Use the wrapper or set env vars
export PKG_CONFIG_SYSROOT_DIR=/path/to/staging
export PKG_CONFIG_LIBDIR=/path/to/staging/usr/lib/pkgconfig
export PKG_CONFIG_PATH=   # must be empty!
$(pkg-config --cflags openssl)   # now returns target paths
```

### 9.3 Pitfall 2 — Float ABI Mismatch (ARM)

```
  ARM Floating-Point ABI Options
  ═══════════════════════════════

  -mfloat-abi=soft      Pure software FP (slowest, most compatible)
  -mfloat-abi=softfp    FP instructions, software calling convention
  -mfloat-abi=hard      FP instructions + hardware calling convention

  Toolchain expects: gnueabihf  →  hard float ABI
  Toolchain expects: gnueabi    →  soft float ABI

  ⚠ Mixing toolchains with different float ABIs causes
    "illegal instruction" crashes at runtime — NOT at link time!
```

```bash
# Check what ABI a binary uses
arm-linux-gnueabihf-readelf -A myapp | grep Tag_ABI_VFP_args
# Tag_ABI_VFP_args: VFP registers  ← hard float ✓
# (absence of this tag means soft float — wrong for gnueabihf!)

# Ensure your CFLAGS match the toolchain
CFLAGS="-march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard"
```

### 9.4 Pitfall 3 — C++ Standard Library Issues

```
  C++ Runtime Options
  ═══════════════════

  libstdc++  (GNU standard library — default in Buildroot gcc toolchains)
  libc++     (LLVM standard library — used with clang)
  uClibc++   (minimal — some C++ features missing)

  ┌───────────────────────────────────────────────────────────────┐
  │ Common error: undefined reference to `std::__throw_bad_alloc' │
  │ Cause: linking against wrong or missing libstdc++             │
  │ Fix: ensure $(STAGING_DIR)/usr/lib/libstdc++.so.* exists      │
  │      and LDFLAGS includes the sysroot                         │
  └───────────────────────────────────────────────────────────────┘
```

```bash
# Check which C++ symbols a binary needs
arm-linux-gnueabihf-nm -D myapp | grep GLIBCXX
# Tells you the minimum libstdc++ version required

# Check that libstdc++ is in staging
ls /path/to/staging/usr/lib/libstdc++.so*
# If missing: enable BR2_INSTALL_LIBSTDCPP in Buildroot config
```

### 9.5 Pitfall 4 — Rust Link Errors with C Libraries

```bash
# ❌ WRONG: Cargo can't find libssl for ARM
$ cargo build --target armv7-unknown-linux-gnueabihf
error: could not find system library 'openssl' required by the 'openssl-sys' crate

# ✅ CORRECT: Set environment variables before building
export OPENSSL_DIR=/path/to/staging/usr
export OPENSSL_LIB_DIR=/path/to/staging/usr/lib
export OPENSSL_INCLUDE_DIR=/path/to/staging/usr/include
export PKG_CONFIG_ALLOW_CROSS=1
export PKG_CONFIG_SYSROOT_DIR=/path/to/staging
export PKG_CONFIG_LIBDIR=/path/to/staging/usr/lib/pkgconfig
export PKG_CONFIG_PATH=

cargo build --target armv7-unknown-linux-gnueabihf
```

### 9.6 Pitfall 5 — autoconf Cache Poisoning

```bash
# ❌ PROBLEM: autoconf caches "can execute" tests using host binary
# If a previous native build left a config.cache, cross-compile fails

# Symptom in configure output:
# checking whether the C compiler works... yes   ← WRONG! Used cache!
# checking for working fork... (cached) yes      ← may be wrong for target

# ✅ FIX: always pass cache overrides for cross-compile
MYAPP_CONF_ENV += \
    ac_cv_func_fork=yes          \
    ac_cv_func_fork_works=yes    \
    ac_cv_func_malloc_0_nonnull=yes \
    ac_cv_func_realloc_0_nonnull=yes

# Or simply delete the cache
rm -f config.cache
```

### 9.7 Pitfall 6 — Architecture Detection Macros

```c
/* ❌ WRONG: Assuming x86 in portable code */
#if defined(__x86_64__)
typedef unsigned long size_t;  /* 8 bytes on x86-64 */
#endif
/* Breaks on 32-bit ARM! */

/* ✅ CORRECT: Use standard types */
#include <stddef.h>   /* size_t */
#include <stdint.h>   /* uint32_t, uint64_t, etc. */
#include <stdbool.h>  /* bool */

/* ✅ CORRECT: Endian-safe integer reading */
#include <endian.h>
uint32_t value = le32toh(*(uint32_t*)buf);  /* little-endian from buffer */
```

```
  Endianness at a Glance:

  Little-Endian (ARM, x86, RISC-V):   Big-Endian (some MIPS, PowerPC):
  ─────────────────────────────────    ─────────────────────────────────
  Value 0x12345678 in memory:          Value 0x12345678 in memory:

  addr+0: 0x78  ← LSB first           addr+0: 0x12  ← MSB first
  addr+1: 0x56                         addr+1: 0x34
  addr+2: 0x34                         addr+2: 0x56
  addr+3: 0x12  ← MSB last            addr+3: 0x78  ← LSB last
```

### 9.8 Pitfall 7 — PATH Ordering

```bash
# ❌ WRONG: Host tools shadow cross-tools
export PATH=/usr/bin:$PATH:/buildroot/output/host/bin
# Running "gcc" finds /usr/bin/gcc (host) first!

# ✅ CORRECT: Buildroot host tools must come FIRST
export PATH=/buildroot/output/host/bin:$PATH
which gcc
# /buildroot/output/host/bin/arm-buildroot-linux-gnueabihf-gcc
# (Buildroot's host/bin/gcc is actually a wrapper/symlink)
```

---

## 10. Summary

### 10.1 Cross-Compilation Fundamentals — Big Picture

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │               Buildroot Cross-Compilation Architecture               │
  │                                                                      │
  │   CROSS_COMPILE prefix              Sysroot                          │
  │   ════════════════════              ═══════                          │
  │   "arm-linux-gnueabihf-"           output/staging/                   │
  │    │                                │                                │
  │    ├─ gcc    → compile .c           ├─ usr/include/  (target hdrs)   │
  │    ├─ g++    → compile .cpp         ├─ usr/lib/      (target libs)   │
  │    ├─ ld     → link                 └─ lib/          (ld, libc)      │
  │    ├─ ar     → archive                                               │
  │    └─ strip  → remove debug                                          │
  │                                                                      │
  │   pkg-config wrapper                HOST_ vs TARGET_                 │
  │   ══════════════════                ══════════════════               │
  │   Sets:                             HOST_CC  → host gcc              │
  │   PKG_CONFIG_SYSROOT_DIR            TARGET_CC → cross-gcc            │
  │   PKG_CONFIG_LIBDIR                 HOST_DIR → output/host/          │
  │   Unsets PKG_CONFIG_PATH            TARGET_DIR → output/target/      │
  │   → gives TARGET flags only         STAGING_DIR → output/staging/    │
  │                                                                      │
  │   Rust additions                                                     │
  │   ═════════════                                                      │
  │   rustup target add <triple>                                         │
  │   .cargo/config.toml linker = "arm-linux-gnueabihf-gcc"              │
  │   PKG_CONFIG_ALLOW_CROSS=1                                           │
  │   OPENSSL_DIR=$(STAGING_DIR)/usr                                     │
  └──────────────────────────────────────────────────────────────────────┘
```

### 10.2 Key Concepts Summary

| Concept | What it is | Why it matters |
|---|---|---|
| `CROSS_COMPILE` | Toolchain prefix string | Selects the right compiler for the target architecture |
| Sysroot | Mirror of target's root filesystem | Provides correct target headers and libraries |
| `pkg-config` wrapper | Script that overrides PKG_CONFIG paths | Prevents host library paths contaminating target builds |
| `HOST_CC` / `TARGET_CC` | Make variables for host vs target | Ensures build-time tools use host compiler, output uses cross-compiler |
| `STAGING_DIR` | The sysroot in Buildroot terms | Where installed target libraries are staged before rootfs assembly |
| Rust target triple | e.g. `armv7-unknown-linux-gnueabihf` | Rust's way of specifying the target architecture and ABI |

### 10.3 Cross-Compilation Checklist

```
  Before starting a cross-compile, verify:

  [ ] CROSS_COMPILE is set and points to the right toolchain
  [ ] --sysroot (or CC configured with baked-in sysroot) is set
  [ ] PKG_CONFIG_SYSROOT_DIR points to staging/
  [ ] PKG_CONFIG_LIBDIR points to staging/usr/lib/pkgconfig
  [ ] PKG_CONFIG_PATH is empty (unset)
  [ ] PATH has output/host/bin BEFORE system paths
  [ ] Float ABI matches toolchain suffix (gnueabihf = hard)
  [ ] No hard-coded /usr paths in Makefiles or .pc files
  [ ] autoconf cache is clean or has cross-compile overrides
  [ ] For Rust: .cargo/config.toml linker is set, OPENSSL_DIR set
  [ ] For C++: libstdc++ is available in staging
  [ ] Binary verified with `file` command (correct ELF arch)
```

### 10.4 Quick Reference — Environment Variables

```bash
# Minimal cross-compilation environment
export CROSS_COMPILE="arm-linux-gnueabihf-"
export CC="${CROSS_COMPILE}gcc"
export CXX="${CROSS_COMPILE}g++"
export LD="${CROSS_COMPILE}ld"
export AR="${CROSS_COMPILE}ar"
export STRIP="${CROSS_COMPILE}strip"
export RANLIB="${CROSS_COMPILE}ranlib"
export OBJCOPY="${CROSS_COMPILE}objcopy"

export SYSROOT="/path/to/buildroot/output/staging"
export CFLAGS="--sysroot=${SYSROOT} -O2"
export CXXFLAGS="${CFLAGS}"
export LDFLAGS="--sysroot=${SYSROOT}"

export PKG_CONFIG_SYSROOT_DIR="${SYSROOT}"
export PKG_CONFIG_LIBDIR="${SYSROOT}/usr/lib/pkgconfig"
export PKG_CONFIG_PATH=""

# For Rust
export CARGO_TARGET_ARMV7_UNKNOWN_LINUX_GNUEABIHF_LINKER="${CC}"
export PKG_CONFIG_ALLOW_CROSS=1
export OPENSSL_DIR="${SYSROOT}/usr"
```

---

*Document generated for Buildroot Series — Chapter 04: Cross-Compilation Fundamentals*
*Covers: CROSS_COMPILE, sysroot, pkg-config wrappers, HOST_/TARGET_ variables, C/C++/Rust examples, and common pitfalls.*