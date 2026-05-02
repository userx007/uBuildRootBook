Now let me create the comprehensive markdown document:The document covers the full topic across 12 sections with ASCII diagrams throughout:

**Structure at a glance:**

1. **Overview & Architecture** — ASCII diagram of the full Buildroot→cargo→rootfs pipeline
2. **Prerequisites** — `menuconfig`, `BR2_PACKAGE_RUST`, arch-to-triple mapping table
3. **`cargo-package` Infrastructure** — lifecycle phases, auto-generated `.cargo/config.toml`
4. **Package Directory Layout** — ASCII tree of the required file structure
5. **Writing the `.mk` File** — four real variants: app, host tool, staging library, feature flags
6. **Vendoring** — step-by-step workflow diagram, `Cargo.toml`, `.cargo/config.toml`
7. **Cross-Compilation Targets** — triple anatomy diagram, full tables for embedded and Linux targets, bare-metal linker script with memory map ASCII art
8. **`Cargo.lock` Reproducibility** — side-by-side "without/with" diagram, annotated lock file anatomy, hash file format
9. **C and C++ Interop** — FFI (C calling Rust and Rust calling C), `bindgen` in `build.rs`, `cxx` crate for C++ bidirectional interop
10. **Real-World Examples** — Cortex-M3 blinky (bare-metal), async Tokio echo server (musl), and a C-extension mixed package
11. **Debugging** — common error messages with fixes, environment variable reference table
12. **Summary** — large ASCII overview diagram + key takeaway bullets

# 10. Rust Packages with `cargo-package` Infrastructure

> **Buildroot Topic · Deep-Dive Reference**
> Covers `cargo-package` (Buildroot ≥ 2023.02), `BR2_PACKAGE_RUST`, vendored dependencies,
> cross-compilation targets, and `Cargo.lock` reproducibility.

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites & Buildroot Configuration](#2-prerequisites--buildroot-configuration)
3. [The `cargo-package` Infrastructure](#3-the-cargo-package-infrastructure)
4. [Package Directory Layout](#4-package-directory-layout)
5. [Writing the `.mk` File](#5-writing-the-mk-file)
6. [Vendoring Dependencies with `cargo vendor`](#6-vendoring-dependencies-with-cargo-vendor)
7. [Cross-Compilation Targets](#7-cross-compilation-targets)
8. [`Cargo.lock` Reproducibility](#8-cargolock-reproducibility)
9. [C and C++ Interop via `cargo-package`](#9-c-and-c-interop-via-cargo-package)
10. [Real-World Rust Package Examples](#10-real-world-rust-package-examples)
11. [Debugging & Common Pitfalls](#11-debugging--common-pitfalls)
12. [Summary](#12-summary)

---

## 1. Overview & Architecture

Buildroot's `cargo-package` infrastructure integrates the Rust/Cargo ecosystem into the
Buildroot build system in a fully reproducible, offline, and cross-compilation-capable way.

```
 BUILDROOT BUILD SYSTEM
 ══════════════════════════════════════════════════════════════════════

  Host machine (x86_64)
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │   menuconfig                                                    │
  │       │                                                         │
  │       ▼                                                         │
  │   BR2_PACKAGE_RUST ──► host/bin/rustc   host/bin/cargo         │
  │                              │                 │                │
  │                    ┌─────────┘                 │                │
  │                    │   Cross-toolchain         │                │
  │                    ▼                           ▼                │
  │   CARGO_TARGET  ──► aarch64-unknown-linux-musl                 │
  │   (triple)          thumbv7m-none-eabi                          │
  │                    x86_64-unknown-linux-gnu                     │
  │                              │                                  │
  │   vendor/  ◄── cargo vendor  │  (offline, reproducible)        │
  │       │                      │                                  │
  │       └──────────────────────┼──► cargo build --release        │
  │                              │         │                        │
  └──────────────────────────────┼─────────┼────────────────────────┘
                                 │         │
                     ════════════▼═════════▼════════════
                       TARGET rootfs  /usr/bin/myapp
                     ══════════════════════════════════
```

### Key design decisions

- **No network access during build** — all crates are vendored (downloaded once, stored in
  the package source tree or a separate tarball).
- **Hermetic Rust toolchain** — Buildroot downloads a pinned `rustc`/`cargo` binary for the
  host, independent of any system-installed Rust.
- **Standard Buildroot lifecycle** — `download`, `extract`, `patch`, `configure`, `build`,
  `install` phases all work as with any other package.
- **`Cargo.lock` is committed** — guarantees bit-for-bit reproducibility across builds.

---

## 2. Prerequisites & Buildroot Configuration

### 2.1 Enabling Rust Support

Open `menuconfig` and navigate:

```
Target packages  --->
  [*] Rust  --->
        Rust compiler version (1.70 / 1.75 / ...)
```

This sets `BR2_PACKAGE_RUST=y` in `.config` and causes Buildroot to:

1. Download the Rust toolchain tarball for the *host* architecture.
2. Unpack it into `$(HOST_DIR)`.
3. Make `rustc`, `cargo`, and `rustup` available under `$(HOST_DIR)/bin/`.

```
# .config (excerpt after menuconfig)
BR2_PACKAGE_RUST=y
BR2_PACKAGE_RUST_TARGET_DEFAULT=y          # auto-derive triple from BR2_ARCH
# BR2_PACKAGE_RUST_TARGET_CUSTOM is not set
```

### 2.2 Architecture-to-Triple Mapping

Buildroot automatically derives the Rust target triple from `BR2_ARCH` and related options:

```
  BR2_ARCH        BR2_LIBC   BR2_ARM_EABI   →  Rust triple
  ──────────────  ─────────  ─────────────     ──────────────────────────────
  aarch64         glibc      N/A            →  aarch64-unknown-linux-gnu
  aarch64         musl       N/A            →  aarch64-unknown-linux-musl
  arm (v7)        glibc      gnueabihf      →  armv7-unknown-linux-gnueabihf
  arm (v7)        musl       musleabihf     →  armv7-unknown-linux-musleabihf
  arm (Cortex-M3) none       none-eabi      →  thumbv7m-none-eabi
  x86_64          glibc      N/A            →  x86_64-unknown-linux-gnu
  x86_64          musl       N/A            →  x86_64-unknown-linux-musl
  riscv64         glibc      N/A            →  riscv64gc-unknown-linux-gnu
```

---

## 3. The `cargo-package` Infrastructure

### 3.1 Infrastructure Declaration

Every Rust package ends its `.mk` file with:

```makefile
$(eval $(cargo-package))
```

This hooks into the same generic infrastructure as `generic-package`, but overrides the
`configure`, `build`, and `install` steps to invoke `cargo`.

### 3.2 What `cargo-package` Does Internally

```
 cargo-package lifecycle
 ══════════════════════════════════════════════════════════
 Phase          Buildroot target       What happens
 ─────────────  ─────────────────────  ──────────────────────────────────────
 download       $(PKG)_SOURCE          wget/git, verify hash
 extract        $(PKG)_EXTRACT_CMDS    unpack tarball into build/
 patch          $(PKG)_PATCH_CMDS      apply *.patch files
 configure      $(PKG)_CONFIGURE_CMDS  write .cargo/config.toml  (vendoring,
                                       target, linker overrides)
 build          $(PKG)_BUILD_CMDS      cargo build --release
                                       --target $(RUST_TARGET_NAME)
                                       --offline
 install        $(PKG)_INSTALL_*_CMDS  cp binary/library to staging/target
 ══════════════════════════════════════════════════════════
```

### 3.3 Generated `.cargo/config.toml`

During the *configure* phase, Buildroot generates a `.cargo/config.toml` in the build
directory:

```toml
# Auto-generated by Buildroot cargo-package infrastructure
# Do NOT edit manually — regenerated on every build

[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "/path/to/build/mypkg-1.0/vendor"

[target.aarch64-unknown-linux-musl]
linker = "aarch64-linux-gnu-gcc"
ar     = "aarch64-linux-gnu-ar"

[profile.release]
opt-level = 3
lto       = true
codegen-units = 1
```

---

## 4. Package Directory Layout

```
 package/
 └── my-rust-app/
     ├── Config.in                  ← menuconfig entry
     ├── my-rust-app.mk             ← Buildroot .mk file
     ├── my-rust-app.hash           ← SHA256 of source tarball
     │
     └── (optional patches)
         └── 0001-fix-libc-link.patch
```

The upstream source tree (downloaded & extracted) looks like:

```
 build/my-rust-app-1.2.0/
 ├── Cargo.toml
 ├── Cargo.lock                     ← MUST be committed upstream
 ├── src/
 │   └── main.rs
 └── vendor/                        ← produced by `cargo vendor`
     ├── libc-0.2.147/
     │   ├── Cargo.toml
     │   └── src/
     ├── serde-1.0.193/
     └── ...
```

---

## 5. Writing the `.mk` File

### 5.1 Minimal Application Package

```makefile
################################################################################
#
# my-rust-app
#
################################################################################

MY_RUST_APP_VERSION  = 1.2.0
MY_RUST_APP_SOURCE   = my-rust-app-$(MY_RUST_APP_VERSION).tar.gz
MY_RUST_APP_SITE     = https://github.com/example/my-rust-app/releases/download/v$(MY_RUST_APP_VERSION)
MY_RUST_APP_LICENSE  = MIT
MY_RUST_APP_LICENSE_FILES = LICENSE

# This package provides a target binary
MY_RUST_APP_INSTALL_TARGET = YES

$(eval $(cargo-package))
```

### 5.2 Package with Host Tool

```makefile
################################################################################
#
# my-rust-codegen  — runs on the host, generates code for target
#
################################################################################

MY_RUST_CODEGEN_VERSION = 0.5.1
MY_RUST_CODEGEN_SOURCE  = my-rust-codegen-$(MY_RUST_CODEGEN_VERSION).tar.gz
MY_RUST_CODEGEN_SITE    = https://crates.io/api/v1/crates/my-rust-codegen/$(MY_RUST_CODEGEN_VERSION)/download
MY_RUST_CODEGEN_SITE_METHOD = http
MY_RUST_CODEGEN_LICENSE = Apache-2.0

# Do NOT install to target rootfs — host tool only
MY_RUST_CODEGEN_INSTALL_TARGET = NO
MY_RUST_CODEGEN_INSTALL_STAGING = NO

$(eval $(cargo-package))
```

### 5.3 Library Package (Staging Only)

```makefile
################################################################################
#
# libmyrust — Rust library installed to staging for linking
#
################################################################################

LIBMYRUST_VERSION = 2.0.0
LIBMYRUST_SOURCE  = libmyrust-$(LIBMYRUST_VERSION).tar.gz
LIBMYRUST_SITE    = https://github.com/example/libmyrust/archive/refs/tags/v$(LIBMYRUST_VERSION)
LIBMYRUST_LICENSE = MIT OR Apache-2.0

LIBMYRUST_INSTALL_STAGING = YES
LIBMYRUST_INSTALL_TARGET  = NO

# Custom install step for the static library
define LIBMYRUST_INSTALL_STAGING_CMDS
    $(INSTALL) -D -m 0644 \
        $(@D)/target/$(RUST_TARGET_NAME)/release/libmyrust.a \
        $(STAGING_DIR)/usr/lib/libmyrust.a
    $(INSTALL) -D -m 0644 $(@D)/include/myrust.h \
        $(STAGING_DIR)/usr/include/myrust.h
endef

$(eval $(cargo-package))
```

### 5.4 Package with Extra Cargo Features

```makefile
################################################################################
#
# ripgrep — with custom feature flags
#
################################################################################

RIPGREP_VERSION  = 14.0.3
RIPGREP_SOURCE   = ripgrep-$(RIPGREP_VERSION).tar.gz
RIPGREP_SITE     = https://github.com/BurntSushi/ripgrep/archive/refs/tags/$(RIPGREP_VERSION)
RIPGREP_LICENSE  = MIT OR UNLICENSE

# Additional cargo flags passed verbatim to `cargo build`
RIPGREP_CARGO_OPTS = --features pcre2

$(eval $(cargo-package))
```

---

## 6. Vendoring Dependencies with `cargo vendor`

Vendoring is the process of bundling all crate dependencies into the source tree so the
build can proceed **offline** (no network required during CI or release builds).

### 6.1 Workflow

```
 Developer workstation (online)
 ══════════════════════════════════════════════════════════════

  1. Write / update Cargo.toml
         │
         ▼
  2. cargo update            ← resolves versions → Cargo.lock
         │
         ▼
  3. cargo vendor vendor/    ← downloads all crates into vendor/
         │
         ├── vendor/serde-1.0.193/
         ├── vendor/tokio-1.35.1/
         └── ...
         │
         ▼
  4. git add Cargo.lock vendor/
     git commit -m "chore: vendor dependencies"
         │
         ▼
  5. tar czf my-rust-app-1.2.0.tar.gz .
     sha256sum *.tar.gz > my-rust-app.hash

 ══════════════════════════════════════════════════════════════
 CI build machine (offline — no internet)

  6. Buildroot downloads tarball (cached in DL_DIR)
  7. cargo build --offline ← reads vendor/ only
```

### 6.2 Cargo.toml with Vendor Config

```toml
[package]
name    = "my-rust-app"
version = "1.2.0"
edition = "2021"

[dependencies]
serde       = { version = "1",  features = ["derive"] }
serde_json  = "1"
log         = "0.4"
env_logger  = "0.10"
tokio       = { version = "1", features = ["rt-multi-thread", "macros", "net"] }
nix         = { version = "0.27", features = ["socket", "uio"] }

[profile.release]
opt-level      = "z"        # minimise binary size (good for embedded)
lto            = true
codegen-units  = 1
strip          = true
panic          = "abort"    # saves ~30 KB on musl targets
```

### 6.3 `.cargo/config.toml` (committed alongside vendor/)

```toml
[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"
```

---

## 7. Cross-Compilation Targets

### 7.1 Common Target Triples

```
 Rust target triple anatomy
 ══════════════════════════════════════════════════════════════

  thumbv7m  -  none  -  eabi
  ────────    ──────   ─────
      │          │       │
      │          │       └─  ABI  (eabi / eabihf / gnu / musl / android)
      │          └──────────  OS   (none / linux / windows / macos)
      └─────────────────────  CPU  (thumb = Cortex-M, aarch64, x86_64 …)

 Common embedded targets:
 ┌──────────────────────────────┬───────────────────────────────────┐
 │  Triple                      │  Typical MCU / SBC                │
 ├──────────────────────────────┼───────────────────────────────────┤
 │  thumbv6m-none-eabi          │  Cortex-M0 / M0+                  │
 │  thumbv7m-none-eabi          │  Cortex-M3                        │
 │  thumbv7em-none-eabi         │  Cortex-M4 (no FPU)               │
 │  thumbv7em-none-eabihf       │  Cortex-M4F / M7 (with FPU)       │
 │  thumbv8m.main-none-eabihf   │  Cortex-M33 / M55                 │
 │  riscv32imac-unknown-none-elf│  ESP32-C3, GD32VF103              │
 └──────────────────────────────┴───────────────────────────────────┘

 Common Linux targets:
 ┌──────────────────────────────┬───────────────────────────────────┐
 │  Triple                      │  Typical use                      │
 ├──────────────────────────────┼───────────────────────────────────┤
 │  aarch64-unknown-linux-gnu   │  RPi 4/5, NVIDIA Jetson, etc.     │
 │  aarch64-unknown-linux-musl  │  Static musl binaries (portable)  │
 │  armv7-unknown-linux-gnueabihf│ RPi 2/3, i.MX6, AM335x          │
 │  x86_64-unknown-linux-musl   │  Static x86 (containers)         │
 │  riscv64gc-unknown-linux-gnu │  SiFive HiFive, StarFive VisionFive│
 └──────────────────────────────┴───────────────────────────────────┘
```

### 7.2 Forcing a Custom Target in the `.mk` File

```makefile
# Override the auto-derived Rust target triple
MY_RUST_APP_CARGO_TARGET = thumbv7m-none-eabi

# For bare-metal targets, no std — set panic handler
MY_RUST_APP_CARGO_OPTS = --no-default-features
```

### 7.3 `.cargo/config.toml` for Bare-Metal

```toml
[build]
target = "thumbv7m-none-eabi"

[target.thumbv7m-none-eabi]
runner  = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
rustflags = [
    "-C", "link-arg=-Tlink.x",
    "-C", "linker=arm-none-eabi-ld",
]
```

### 7.4 Linker Script for Cortex-M3

```
 Memory map  (STM32F103 — 64 KB RAM, 128 KB Flash)
 ══════════════════════════════════════════════════

  0x0800_0000  ┌──────────────┐  ← FLASH origin
               │   .text      │    code / rodata
               │   .rodata    │
               │              │
  0x0802_0000  └──────────────┘  ← FLASH end (128 KB)

  0x2000_0000  ┌──────────────┐  ← RAM origin
               │   .bss       │    zero-init
               │   .data      │    initialised data
               │   .uninit    │
               │   ──────     │
               │   Stack ↓    │
  0x2001_0000  └──────────────┘  ← RAM end (64 KB)
```

```ld
/* memory.x  — included by cortex-m-rt */
MEMORY {
    FLASH : ORIGIN = 0x08000000, LENGTH = 128K
    RAM   : ORIGIN = 0x20000000, LENGTH = 64K
}
```

---

## 8. `Cargo.lock` Reproducibility

### 8.1 Why `Cargo.lock` Must Be Committed

```
 Without Cargo.lock                    With Cargo.lock
 ══════════════════════════           ══════════════════════════════

  Build A  ──► serde 1.0.190          Build A  ──► serde 1.0.193
  Build B  ──► serde 1.0.193          Build B  ──► serde 1.0.193
  Build C  ──► serde 1.0.195  ✗       Build C  ──► serde 1.0.193  ✓

  Different binaries!                 Identical binaries!
  Non-reproducible!                   Reproducible!
```

### 8.2 Anatomy of `Cargo.lock`

```toml
# This file is automatically @generated by Cargo.
# It is not intended for manual editing.
version = 3

[[package]]
name       = "my-rust-app"
version    = "1.2.0"

[[package]]
name       = "serde"
version    = "1.0.193"
source     = "registry+https://github.com/rust-lang/crates.io-index"
checksum   = "25dd0975d2c2f38107e2ba73ec4df3df6fec5fd25e18a91cceb1c1af61870e3b"
dependencies = [
    "serde_derive",
]

[[package]]
name       = "serde_derive"
version    = "1.0.193"
source     = "registry+https://github.com/rust-lang/crates.io-index"
checksum   = "43576ca501357b9b071ac53cdc7da8ef0cbd9493d8df096cd74d7d1e5a2f85a"
dependencies = [
    "proc-macro2",
    "quote",
    "syn 2.0.39",
]
```

### 8.3 Buildroot Hash File

```
# my-rust-app.hash
# sha256  <hash>  <filename>
sha256  a3b2c1d4e5f6...  my-rust-app-1.2.0.tar.gz
sha256  0a1b2c3d4e5f...  LICENSE
```

---

## 9. C and C++ Interop via `cargo-package`

A common scenario is a Rust package that links against a C/C++ library already present in
the Buildroot staging directory, or a Rust library that exposes a C API.

### 9.1 Rust Calling a C Library (FFI)

```c
/* include/sensor.h  — a C library already in staging */
#ifndef SENSOR_H
#define SENSOR_H

#include <stdint.h>

typedef struct {
    float temperature;
    float humidity;
    uint32_t timestamp_ms;
} SensorReading;

int  sensor_open(const char *device_path);
int  sensor_read(int fd, SensorReading *out);
void sensor_close(int fd);

#endif /* SENSOR_H */
```

```rust
// src/ffi.rs  — unsafe FFI bindings (hand-written)
use std::ffi::CStr;
use std::os::raw::{c_char, c_int, c_uint, c_float};

#[repr(C)]
pub struct SensorReading {
    pub temperature:  c_float,
    pub humidity:     c_float,
    pub timestamp_ms: c_uint,
}

#[link(name = "sensor")]
extern "C" {
    pub fn sensor_open(device_path: *const c_char) -> c_int;
    pub fn sensor_read(fd: c_int, out: *mut SensorReading) -> c_int;
    pub fn sensor_close(fd: c_int);
}
```

```rust
// src/main.rs
mod ffi;

use std::ffi::CString;
use ffi::SensorReading;

fn main() {
    let path = CString::new("/dev/i2c-1").expect("CString failed");

    let fd = unsafe { ffi::sensor_open(path.as_ptr()) };
    if fd < 0 {
        eprintln!("Failed to open sensor");
        std::process::exit(1);
    }

    let mut reading = SensorReading {
        temperature: 0.0,
        humidity:    0.0,
        timestamp_ms: 0,
    };

    let rc = unsafe { ffi::sensor_read(fd, &mut reading) };
    if rc == 0 {
        println!("Temperature: {:.1} °C", reading.temperature);
        println!("Humidity:    {:.1} %",  reading.humidity);
    } else {
        eprintln!("Read error: {}", rc);
    }

    unsafe { ffi::sensor_close(fd) };
}
```

```makefile
# package/rust-sensor-logger/rust-sensor-logger.mk
RUST_SENSOR_LOGGER_VERSION = 1.0.0
RUST_SENSOR_LOGGER_SOURCE  = rust-sensor-logger-$(RUST_SENSOR_LOGGER_VERSION).tar.gz
RUST_SENSOR_LOGGER_SITE    = https://github.com/example/rust-sensor-logger/releases/download/v$(RUST_SENSOR_LOGGER_VERSION)
RUST_SENSOR_LOGGER_LICENSE = MIT

# Depend on the C library package
RUST_SENSOR_LOGGER_DEPENDENCIES = libsensor

$(eval $(cargo-package))
```

### 9.2 Using `bindgen` in a `build.rs`

```rust
// build.rs  — runs on the HOST during compilation
fn main() {
    // Tell cargo to re-run if header changes
    println!("cargo:rerun-if-changed=include/sensor.h");

    // Tell cargo to link against libsensor
    println!("cargo:rustc-link-lib=sensor");
    println!("cargo:rustc-link-search={}/usr/lib",
             std::env::var("STAGING_DIR").unwrap_or_default());

    // Generate bindings (bindgen must be in [build-dependencies])
    let bindings = bindgen::Builder::default()
        .header("include/sensor.h")
        .clang_arg(format!(
            "--sysroot={}",
            std::env::var("STAGING_DIR").unwrap_or_default()
        ))
        .parse_callbacks(Box::new(bindgen::CargoCallbacks))
        .generate()
        .expect("Unable to generate bindings");

    let out_path = std::path::PathBuf::from(
        std::env::var("OUT_DIR").unwrap()
    );
    bindings
        .write_to_file(out_path.join("sensor_bindings.rs"))
        .expect("Couldn't write bindings");
}
```

```toml
# Cargo.toml
[build-dependencies]
bindgen = "0.69"
```

### 9.3 Exposing a Rust Library to C

```rust
// src/lib.rs  — Rust library with C-compatible API
use std::ffi::CStr;
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn compute_crc32(data: *const u8, len: usize) -> u32 {
    if data.is_null() || len == 0 {
        return 0;
    }
    let slice = unsafe { std::slice::from_raw_parts(data, len) };
    crc32fast::hash(slice)
}

#[no_mangle]
pub extern "C" fn rust_greet(name: *const c_char) -> *mut c_char {
    let c_str = unsafe { CStr::from_ptr(name) };
    let greeting = format!("Hello, {}! (from Rust)", c_str.to_str().unwrap_or("?"));
    // Caller must free with rust_free_string()
    std::ffi::CString::new(greeting)
        .unwrap()
        .into_raw()
}

#[no_mangle]
pub extern "C" fn rust_free_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe { drop(std::ffi::CString::from_raw(s)) };
    }
}
```

```c
/* example C consumer */
#include <stdio.h>
#include <stdint.h>

extern uint32_t compute_crc32(const uint8_t *data, size_t len);
extern char    *rust_greet(const char *name);
extern void     rust_free_string(char *s);

int main(void) {
    const char payload[] = "Hello, Buildroot!";
    uint32_t crc = compute_crc32((const uint8_t *)payload, sizeof(payload) - 1);
    printf("CRC32: 0x%08X\n", crc);

    char *msg = rust_greet("Buildroot");
    puts(msg);
    rust_free_string(msg);
    return 0;
}
```

```makefile
# Cargo.toml crate-type for a C-compatible static library
# [lib]
# crate-type = ["staticlib", "cdylib"]
```

### 9.4 C++ Interop via `cxx` Crate

```rust
// src/bridge.rs  — using the `cxx` crate for safe C++ interop
#[cxx::bridge]
mod ffi {
    // C++ types exposed to Rust
    unsafe extern "C++" {
        include!("myapp/include/image_processor.h");

        type ImageProcessor;

        fn new_image_processor() -> UniquePtr<ImageProcessor>;
        fn process(&self, width: u32, height: u32) -> bool;
        fn get_output_path(&self) -> &CxxString;
    }

    // Rust functions exposed to C++
    extern "Rust" {
        fn rust_status_callback(code: i32, message: &str);
    }
}

pub fn rust_status_callback(code: i32, message: &str) {
    eprintln!("[Rust callback] code={} message={}", code, message);
}
```

```cpp
// include/image_processor.h
#pragma once
#include <memory>
#include <string>

class ImageProcessor {
public:
    bool        process(uint32_t width, uint32_t height);
    std::string get_output_path() const;
private:
    std::string output_path_;
};

std::unique_ptr<ImageProcessor> new_image_processor();
```

```cpp
// src/image_processor.cpp
#include "myapp/include/image_processor.h"
#include "myapp/src/bridge.rs.h"  // generated by cxx

bool ImageProcessor::process(uint32_t width, uint32_t height) {
    output_path_ = "/tmp/output_" +
                   std::to_string(width) + "x" +
                   std::to_string(height) + ".png";
    // call back into Rust
    rust_status_callback(0, "Processing complete");
    return true;
}

std::string ImageProcessor::get_output_path() const {
    return output_path_;
}

std::unique_ptr<ImageProcessor> new_image_processor() {
    return std::make_unique<ImageProcessor>();
}
```

---

## 10. Real-World Rust Package Examples

### 10.1 Bare-Metal Cortex-M3 Application (`thumbv7m-none-eabi`)

```toml
# Cargo.toml
[package]
name    = "blinky-m3"
version = "0.1.0"
edition = "2021"

[dependencies]
cortex-m     = "0.7"
cortex-m-rt  = "0.7"
panic-halt   = "0.2"

# STM32F103 peripheral access crate
stm32f1xx-hal = { version = "0.10", features = ["stm32f103", "rt"] }

[profile.release]
opt-level     = "z"
lto           = true
codegen-units = 1
debug         = false
```

```rust
// src/main.rs  — LED blink on STM32F103 Cortex-M3
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _;

use stm32f1xx_hal::{
    pac,
    prelude::*,
    gpio::PinState,
    delay::Delay,
};

#[entry]
fn main() -> ! {
    // Take ownership of peripherals
    let cp  = cortex_m::Peripherals::take().unwrap();
    let dp  = pac::Peripherals::take().unwrap();

    let mut flash = dp.FLASH.constrain();
    let rcc       = dp.RCC.constrain();
    let clocks    = rcc.cfgr.freeze(&mut flash.acr);

    let mut delay = Delay::new(cp.SYST, &clocks);

    // Configure PC13 as push-pull output (onboard LED on Blue Pill)
    let mut gpioc = dp.GPIOC.split();
    let mut led   = gpioc.pc13.into_push_pull_output_with_state(
        &mut gpioc.crh,
        PinState::High,   // LED off (active-low on Blue Pill)
    );

    loop {
        led.set_low();          // LED on
        delay.delay_ms(500u32);
        led.set_high();         // LED off
        delay.delay_ms(500u32);
    }
}
```

```makefile
# package/blinky-m3/blinky-m3.mk
BLINKY_M3_VERSION  = 0.1.0
BLINKY_M3_SOURCE   = blinky-m3-$(BLINKY_M3_VERSION).tar.gz
BLINKY_M3_SITE     = https://github.com/example/blinky-m3/archive/refs/tags/v$(BLINKY_M3_VERSION)
BLINKY_M3_LICENSE  = MIT

# Bare-metal: override target triple and install nothing to rootfs
BLINKY_M3_CARGO_TARGET       = thumbv7m-none-eabi
BLINKY_M3_INSTALL_TARGET     = NO
BLINKY_M3_INSTALL_STAGING    = NO

define BLINKY_M3_INSTALL_IMAGES_CMDS
    $(INSTALL) -D -m 0644 \
        $(@D)/target/thumbv7m-none-eabi/release/blinky-m3 \
        $(BINARIES_DIR)/blinky-m3.elf
    $(TARGET_CROSS)objcopy -O binary \
        $(BINARIES_DIR)/blinky-m3.elf \
        $(BINARIES_DIR)/blinky-m3.bin
endef

$(eval $(cargo-package))
```

### 10.2 Async Network Service (`aarch64-unknown-linux-musl`)

```rust
// src/main.rs  — async TCP echo server, statically linked with musl
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;
use std::error::Error;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("0.0.0.0:7878").await?;
    println!("Echo server listening on :7878");

    loop {
        let (mut socket, addr) = listener.accept().await?;
        println!("Connection from {}", addr);

        tokio::spawn(async move {
            let mut buf = vec![0u8; 4096];
            loop {
                let n = match socket.read(&mut buf).await {
                    Ok(0) => return,          // client disconnected
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("Read error: {}", e);
                        return;
                    }
                };
                if let Err(e) = socket.write_all(&buf[..n]).await {
                    eprintln!("Write error: {}", e);
                    return;
                }
            }
        });
    }
}
```

```makefile
# package/echo-server/echo-server.mk
ECHO_SERVER_VERSION  = 1.0.0
ECHO_SERVER_SOURCE   = echo-server-$(ECHO_SERVER_VERSION).tar.gz
ECHO_SERVER_SITE     = https://github.com/example/echo-server/archive/refs/tags/v$(ECHO_SERVER_VERSION)
ECHO_SERVER_LICENSE  = MIT OR Apache-2.0

# Statically linked musl binary — no libc dependency on target
ECHO_SERVER_CARGO_TARGET = aarch64-unknown-linux-musl

$(eval $(cargo-package))
```

```
 Resulting binary characteristics (musl + lto + opt-z)
 ══════════════════════════════════════════════════════

  Architecture : AArch64
  Linking      : static (musl libc)
  Size (strip) : ~2.8 MB  (tokio async runtime included)
  ldd output   : "not a dynamic executable"  ← fully portable!
  libc dep     : none
```

### 10.3 C/Rust Mixed Package (`ripgrep`-style)

```
 Build flow for a C-extension Rust package
 ══════════════════════════════════════════════════════════════════

   Buildroot
      │
      ├─► cargo build ──────────────────────────────────┐
      │       │                                         │
      │       ├─► build.rs (runs on HOST)               │
      │       │       │                                 │
      │       │       ├─► pkg-config libpcre2            │
      │       │       │       │                         │
      │       │       │       └─► STAGING_DIR/usr/lib/  │
      │       │       │               libpcre2-8.a ──── ┤
      │       │       │                                 │
      │       │       └─► emit: cargo:rustc-link-lib=pcre2-8
      │       │                 cargo:rustc-link-search=...
      │       │
      │       └─► rustc ──► target binary
      │                         │
      └─────────────────────────┴─► $(TARGET_DIR)/usr/bin/
```

---

## 11. Debugging & Common Pitfalls

### 11.1 Diagnosing Build Failures

```bash
# Enable verbose cargo output in Buildroot
make my-rust-app V=1

# Inspect the generated .cargo/config.toml
cat $(find output/build/my-rust-app-* -name config.toml -path '*/.cargo/*')

# Check which Rust toolchain is being used
output/host/bin/rustc --version
output/host/bin/cargo --version

# Test vendoring manually
cd output/build/my-rust-app-1.2.0
CARGO_HOME=$(pwd)/.cargo-test output/host/bin/cargo build \
    --offline \
    --target aarch64-unknown-linux-musl \
    --release
```

### 11.2 Common Error Messages

```
 ERROR: vendor/ directory missing
 ═══════════════════════════════════════════════════════════════════
  Symptom:  error: no registry sources are supported when
            `--frozen` is used, but crates.io was found

  Fix:      Run `cargo vendor vendor/` in the package source,
            commit vendor/ and Cargo.lock, then rebuild the tarball.

 ═══════════════════════════════════════════════════════════════════
 ERROR: linker not found
 ═══════════════════════════════════════════════════════════════════
  Symptom:  error: linker `aarch64-linux-gnu-gcc` not found

  Fix:      Ensure BR2_PACKAGE_HOST_BINUTILS and the correct
            cross-toolchain package are enabled in menuconfig.

 ═══════════════════════════════════════════════════════════════════
 ERROR: proc-macro crate can't cross-compile
 ═══════════════════════════════════════════════════════════════════
  Symptom:  error[E0463]: can't find crate for `proc_macro`
            for target thumbv7m-none-eabi

  Fix:      proc-macro crates must compile for the HOST.
            Buildroot's cargo-package handles this automatically
            via CARGO_BUILD_TARGET / --target separation.
            Ensure build.rs build-dependencies do NOT appear in
            [dependencies] for no_std targets.

 ═══════════════════════════════════════════════════════════════════
 ERROR: hash mismatch
 ═══════════════════════════════════════════════════════════════════
  Symptom:  my-rust-app-1.2.0.tar.gz: sha256 mismatch

  Fix:      Recompute with:
              sha256sum my-rust-app-1.2.0.tar.gz
            and update my-rust-app.hash.
```

### 11.3 Cargo Environment Variables in Buildroot

| Variable | Set by Buildroot | Purpose |
|---|---|---|
| `CARGO_HOME` | `$(BUILD_DIR)/.cargo` | Shared cache directory |
| `CARGO_TARGET_DIR` | `$(@D)/target` | Build artefacts |
| `STAGING_DIR` | `$(STAGING_DIR)` | Sysroot for pkg-config |
| `PKG_CONFIG_SYSROOT_DIR` | `$(STAGING_DIR)` | pkg-config sysroot |
| `PKG_CONFIG_PATH` | `$(STAGING_DIR)/usr/lib/pkgconfig` | Library discovery |
| `HOST_DIR` | `$(HOST_DIR)` | Host tools (rustc, cc, …) |
| `RUST_TARGET_NAME` | auto-derived | Target triple string |

### 11.4 Checking for `std` Availability

```rust
// Check at compile time if std is available
// useful for crates that support both std and no_std

#![cfg_attr(not(feature = "std"), no_std)]

#[cfg(feature = "std")]
use std::collections::HashMap;

#[cfg(not(feature = "std"))]
use core::cell::UnsafeCell;  // no_std fallback
```

```toml
# Cargo.toml
[features]
default = ["std"]
std     = []
```

---

## 12. Summary

```
 RUST PACKAGES WITH cargo-package — SUMMARY DIAGRAM
 ══════════════════════════════════════════════════════════════════════════════

  CONFIGURATION          PACKAGE DEFINITION          BUILD PIPELINE
  ═════════════          ══════════════════          ══════════════

  menuconfig             Config.in                   1. Download tarball
      │                      │                           (with vendor/)
  BR2_PACKAGE_RUST=y     package/foo/                    │
      │                  ├── Config.in               2. Verify .hash
      │                  ├── foo.mk                      │
      ▼                  └── foo.hash               3. Extract source
  host/bin/rustc                │                        │
  host/bin/cargo            $(eval $(cargo-package)) 4. Apply patches
      │                                                  │
      └─────────────────────────────────────────────►5. Configure
                                                         Write .cargo/config.toml
  TARGET TRIPLES                                         (vendor sources,
  ══════════════                                          linker override,
                                                          target triple)
  Linux:                                                 │
  aarch64-unknown-linux-musl ◄────────────────────────►6. Build
  aarch64-unknown-linux-gnu                              cargo build --release
  armv7-unknown-linux-gnueabihf                          --target $TRIPLE
  x86_64-unknown-linux-musl                              --offline
                                                         │
  Bare-metal:                                        7. Install
  thumbv7m-none-eabi                                     → TARGET_DIR/usr/bin/
  thumbv7em-none-eabihf                                  → STAGING_DIR/usr/lib/
  thumbv6m-none-eabi                                     → BINARIES_DIR/*.elf
  riscv32imac-unknown-none-elf

  KEY CONSTRAINTS
  ═══════════════
  ✓  vendor/ must be in source tarball (offline build)
  ✓  Cargo.lock must be committed (reproducibility)
  ✓  .hash file must match tarball SHA256
  ✓  Rust toolchain pinned per Buildroot release
  ✓  No network access during `make`
  ✓  Cross-compilation via CARGO_TARGET and linker override
  ✓  C/C++ interop via FFI, bindgen, or cxx crate
  ✓  Works with Buildroot ≥ 2023.02
```

### Key Takeaways

**`cargo-package` infrastructure** (Buildroot ≥ 2023.02) provides a first-class, reproducible
Rust integration into Buildroot's build system. It follows the same package lifecycle as all
other Buildroot packages while transparently handling the unique requirements of the Cargo
ecosystem.

**Vendoring** (`cargo vendor`) is mandatory for offline/reproducible builds. All crate
dependencies must be bundled into the source tarball before release. The generated
`.cargo/config.toml` redirects Cargo to read from the `vendor/` directory instead of the
network.

**`Cargo.lock` must be committed** to the upstream repository. Without it, dependency
resolution is non-deterministic and builds will differ between machines and over time.

**Cross-compilation** is handled by setting `CARGO_TARGET` (either auto-derived from
`BR2_ARCH`/`BR2_LIBC` or explicitly overridden in the `.mk` file) and injecting the
Buildroot cross-linker path into `.cargo/config.toml`. Both Linux targets (musl/glibc) and
bare-metal targets (Cortex-M, RISC-V) are supported.

**Interoperability** with C and C++ is achieved via Rust's FFI (`extern "C"` blocks),
`bindgen` for automatic header-to-binding generation, and the `cxx` crate for safe,
bidirectional C++ interop — all of which work transparently within the `cargo-package`
infrastructure.

---

*Buildroot version context: cargo-package introduced in Buildroot 2023.02 · Rust toolchain
pinned per Buildroot release · `BR2_PACKAGE_RUST` required for all Rust packages*