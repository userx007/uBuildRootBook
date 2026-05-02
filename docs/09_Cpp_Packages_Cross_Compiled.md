# 09. C++ Packages — Cross-Compiled Libraries & Applications

**Structure overview:**
- **Cross-compilation pipeline** — ASCII diagram showing host→target flow with sysroot layout
- **Package anatomy** — `Config.in` with all C++ guards (`BR2_INSTALL_LIBSTDCPP`, `BR2_TOOLCHAIN_GCC_AT_LEAST_7`, etc.) and the `.mk` file skeleton
- **CMake integration** — the auto-generated `toolchainfile.cmake` Buildroot produces, with explanation of `FIND_ROOT_PATH_MODE ONLY` preventing host library leakage
- **`CMAKE_CXX_FLAGS`** — three methods: `.mk` conf opts, project-side cache variables, and post-patch hooks for upstream projects
- **C++ standard selection** — `CMAKE_CXX_STANDARD=17` + `EXTENSIONS=OFF` difference (`-std=c++17` vs `-std=gnu++17`), compatibility table C++11→C++23
- **Sysroot linking** — ASCII showing correct vs wrong linker invocation, `pkg-config` wrapper usage, `FindPackage` patterns
- **Stripping** — ASCII size comparison table (~93% reduction), four stripping methods, ELF section table showing what gets removed
- **Rust** — `cargo-package` `.mk`, `Cargo.toml` with `strip=true`/`panic=abort`, full sensor reader in Rust
- **Full C++17 example** — complete `CMakeLists.txt`, `sensor.hpp`, `sensor.cpp` (using `std::optional`, `std::from_chars`, structured bindings, `std::filesystem`), and `main.cpp`
- **Troubleshooting** — common ABI/linking errors with causes and fixes
- **Summary checklist** — ASCII box with full build checklist and key `make` commands

## Buildroot: Packaging, CMake Integration, and Cross-Compilation

---

## Table of Contents

1. [Overview](#overview)
2. [The Cross-Compilation Pipeline](#the-cross-compilation-pipeline)
3. [Buildroot Package Anatomy for C++](#buildroot-package-anatomy-for-c++)
4. [CMake Integration in Buildroot](#cmake-integration-in-buildroot)
5. [Passing CMAKE_CXX_FLAGS](#passing-cmake_cxx_flags)
6. [C++ Standard Selection](#c++-standard-selection)
7. [Linking Against Target Sysroot Libraries](#linking-against-target-sysroot-libraries)
8. [Stripping Debug Symbols](#stripping-debug-symbols)
9. [Rust Cross-Compilation in Buildroot](#rust-cross-compilation-in-buildroot)
10. [Full Working Examples](#full-working-examples)
11. [Troubleshooting](#troubleshooting)
12. [Summary](#summary)

---

## Overview

Buildroot is a simple, efficient, and easy-to-use tool to generate embedded Linux systems through cross-compilation. When dealing with C++ packages, the challenges multiply: name mangling, ABI compatibility, standard library selection (`libstdc++` vs `libc++`), and proper sysroot linking all require careful attention.

This chapter covers the full workflow for packaging a CMake-based C++ project in Buildroot, including:

- Writing `.mk` and `Config.in` files for a C++ package
- Controlling `CMAKE_CXX_FLAGS` and C++ standard selection
- Resolving and linking against target sysroot libraries
- Stripping debug symbols to reduce binary size on the target

---

## The Cross-Compilation Pipeline

```
  HOST MACHINE                         TARGET (e.g., ARM Cortex-A9)
  ┌──────────────────────────┐         ┌──────────────────────────┐
  │                          │         │                          │
  │  Source Code (.cpp/.hpp) │         │  Embedded Linux System   │
  │          │               │         │                          │
  │          ▼               │         │  ┌────────────────────┐  │
  │  CMake Configure         │         │  │  /usr/lib/         │  │
  │  (toolchain file)        │         │  │  libstdc++.so      │  │
  │          │               │         │  │  libfoo.so         │  │
  │          ▼               │  SSH /  │  └────────────────────┘  │
  │  Cross-Compiler          │  NFS /  │                          │
  │  arm-linux-gnueabihf-g++ │─ TFTP ─▶│  ┌────────────────────┐  │
  │          │               │         │  │  /usr/bin/         │  │
  │          ▼               │         │  │  myapp  (stripped) │  │
  │  Linker: ld (cross)      │         │  └────────────────────┘  │
  │    → links vs SYSROOT    │         │                          │
  │          │               │         └──────────────────────────┘
  │          ▼               │
  │  strip (cross)           │
  │  → removes debug info    │
  │          │               │
  │          ▼               │
  │  Buildroot staging/      │
  │  target/ directories     │
  └──────────────────────────┘

  SYSROOT layout:
  $(STAGING_DIR)/
  ├── usr/
  │   ├── include/       ← headers for compilation
  │   └── lib/           ← .so / .a for linking
  └── lib/               ← runtime libs (libc, libstdc++)
```

---

## Buildroot Package Anatomy for C++

Every Buildroot package lives under `package/<pkgname>/` and requires at minimum two files:

### `Config.in`

```kconfig
config BR2_PACKAGE_MYCPPAPP
    bool "mycppapp"
    depends on BR2_INSTALL_LIBSTDCPP
    depends on BR2_USE_WCHAR
    depends on BR2_TOOLCHAIN_HAS_THREADS
    select BR2_PACKAGE_LIBFOO
    help
      A sample C++17 application demonstrating Buildroot
      cross-compilation with CMake.

      https://example.com/mycppapp
```

**Key `depends on` guards for C++:**

| Guard | Meaning |
|---|---|
| `BR2_INSTALL_LIBSTDCPP` | Target has `libstdc++` installed |
| `BR2_USE_WCHAR` | Wide-character support present |
| `BR2_TOOLCHAIN_HAS_THREADS` | pthreads available |
| `BR2_TOOLCHAIN_GCC_AT_LEAST_7` | GCC 7+ for C++17 |
| `BR2_TOOLCHAIN_HAS_CXX` | Toolchain supports C++ at all |

---

### `mycppapp.mk`

```makefile
################################################################################
#
# mycppapp — sample C++17 CMake application
#
################################################################################

MYCPPAPP_VERSION       = 1.2.0
MYCPPAPP_SITE          = https://github.com/example/mycppapp/archive/refs/tags
MYCPPAPP_SITE_METHOD   = wget
MYCPPAPP_SOURCE        = v$(MYCPPAPP_VERSION).tar.gz

# Declare dependencies (host tools + target libs)
MYCPPAPP_DEPENDENCIES  = libfoo zlib

# Tell Buildroot this is a CMake package
MYCPPAPP_INSTALL_STAGING = YES    # install headers+libs to staging_dir
MYCPPAPP_INSTALL_TARGET  = YES    # install binaries to target_dir

# Extra CMake arguments (see dedicated section below)
MYCPPAPP_CONF_OPTS = \
    -DCMAKE_BUILD_TYPE=Release         \
    -DBUILD_SHARED_LIBS=OFF            \
    -DMYCPPAPP_ENABLE_TESTS=OFF        \
    -DCMAKE_CXX_STANDARD=17            \
    -DCMAKE_CXX_STANDARD_REQUIRED=ON

$(eval $(cmake-package))
```

The final line `$(eval $(cmake-package))` invokes Buildroot's CMake package infrastructure, which automatically:

- Sets `CMAKE_TOOLCHAIN_FILE` to the Buildroot-generated toolchain file
- Sets `CMAKE_INSTALL_PREFIX` to the staging directory
- Runs `cmake`, `make`, and `make install`

---

## CMake Integration in Buildroot

Buildroot generates a CMake toolchain file at:

```
$(BUILD_DIR)/toolchainfile.cmake
```

It looks roughly like this (auto-generated, do not edit manually):

```cmake
# Auto-generated by Buildroot — toolchainfile.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER   /path/to/arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER /path/to/arm-linux-gnueabihf-g++)
set(CMAKE_Fortran_COMPILER /path/to/arm-linux-gnueabihf-gfortran)

set(CMAKE_SYSROOT /path/to/buildroot/output/host/arm-linux-gnueabihf/sysroot)

set(CMAKE_FIND_ROOT_PATH  /path/to/buildroot/output/host/arm-linux-gnueabihf/sysroot)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)   # don't look for host programs
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)    # only target libs
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)    # only target headers
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

The `ONLY` modes for `LIBRARY` and `INCLUDE` are critical — they prevent CMake from accidentally picking up host libraries instead of target ones.

---

## Passing CMAKE_CXX_FLAGS

### Method 1: Directly in the `.mk` file (recommended)

```makefile
MYCPPAPP_CONF_OPTS += \
    -DCMAKE_CXX_FLAGS="-Wall -Wextra -fstack-protector-strong -D_FORTIFY_SOURCE=2"
```

For flags that include spaces or special characters, use `MYCPPAPP_CONF_ENV`:

```makefile
MYCPPAPP_CONF_ENV += \
    CXXFLAGS="-march=armv7-a -mfpu=neon -mfloat-abi=hard -O2"
```

### Method 2: Override via `CMakeLists.txt` in the package

If you maintain the upstream CMakeLists.txt, add a cache variable:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(mycppapp CXX)

# Accept external flags injection (set by Buildroot)
set(MYCPPAPP_EXTRA_CXX_FLAGS "" CACHE STRING "Extra C++ compile flags")

target_compile_options(mycppapp PRIVATE
    -Wall
    -Wextra
    -Wno-unused-parameter
    ${MYCPPAPP_EXTRA_CXX_FLAGS}
)
```

Then in the `.mk`:

```makefile
MYCPPAPP_CONF_OPTS += \
    -DMYCPPAPP_EXTRA_CXX_FLAGS="-fomit-frame-pointer -funwind-tables"
```

### Method 3: Post-patch hook (for upstream projects you can't modify)

```makefile
define MYCPPAPP_INJECT_FLAGS
    $(SED) 's/target_compile_options(mycppapp/target_compile_options(mycppapp PRIVATE -fstack-protector-strong #/' \
        $(@D)/CMakeLists.txt
endef

MYCPPAPP_POST_PATCH_HOOKS += MYCPPAPP_INJECT_FLAGS
```

---

## C++ Standard Selection

### In the `.mk` file

```makefile
# Prefer CMake's built-in standard handling over raw -std= flags
MYCPPAPP_CONF_OPTS += \
    -DCMAKE_CXX_STANDARD=17            \
    -DCMAKE_CXX_STANDARD_REQUIRED=ON   \
    -DCMAKE_CXX_EXTENSIONS=OFF
```

`CMAKE_CXX_EXTENSIONS=OFF` enforces `-std=c++17` instead of `-std=gnu++17`, which is important for strict standards compliance on embedded targets.

### In `CMakeLists.txt` (project-side control)

```cmake
cmake_minimum_required(VERSION 3.16)
project(mycppapp LANGUAGES CXX)

# ── C++ Standard ──────────────────────────────────────────────
set(CMAKE_CXX_STANDARD          17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS        OFF)   # -std=c++17, not gnu++17

# ── Source files ──────────────────────────────────────────────
add_executable(mycppapp
    src/main.cpp
    src/sensor.cpp
    src/protocol.cpp
)
```

### Checking GCC version in `Config.in`

C++17 requires GCC 7+, C++20 requires GCC 10+:

```kconfig
config BR2_PACKAGE_MYCPPAPP
    bool "mycppapp"
    depends on BR2_INSTALL_LIBSTDCPP
    depends on BR2_TOOLCHAIN_GCC_AT_LEAST_7
    comment "mycppapp needs a toolchain with C++17 support (GCC >= 7)"
        depends on !BR2_TOOLCHAIN_GCC_AT_LEAST_7
```

### C++ Standard Feature Compatibility Table

```
  C++ Standard  │ GCC Min │ Key Features
  ──────────────┼─────────┼──────────────────────────────────────────
  C++11         │  4.8.1  │ auto, lambdas, move semantics, threads
  C++14         │  5.0    │ generic lambdas, constexpr relaxed
  C++17         │  7.0    │ if constexpr, structured bindings, std::optional
  C++20         │  10.0   │ concepts, ranges, coroutines, modules
  C++23         │  13.0   │ std::print, std::expected, stacktrace
  ──────────────┴─────────┴──────────────────────────────────────────
  Buildroot default: determined by BR2_GCC_VERSION_*
```

---

## Linking Against Target Sysroot Libraries

### The Problem

When building for a target like ARM, you must link against the *target* variant of every library — not the host's `/usr/lib`. Accidentally linking against the host is a common pitfall.

```
  WRONG ─────────────────────────────────────────────────────────
  arm-linux-gnueabihf-g++ main.o -o myapp -L/usr/lib -lzlib
                                            ↑ HOST library!
                                            ABI mismatch → CRASH

  CORRECT ────────────────────────────────────────────────────────
  arm-linux-gnueabihf-g++ main.o -o myapp \
      --sysroot=$(STAGING_DIR)             \
      -L$(STAGING_DIR)/usr/lib             \
      -lz
                ↑ TARGET sysroot library ✓
```

### How Buildroot handles it automatically

Buildroot injects the sysroot through the toolchain wrapper scripts at:

```
$(HOST_DIR)/bin/arm-linux-gnueabihf-g++
```

These wrapper scripts prepend `--sysroot=$(STAGING_DIR)` to every compiler invocation. You can inspect a wrapper:

```bash
cat output/host/bin/arm-linux-gnueabihf-g++
# #!/bin/sh
# exec /path/to/real/gcc --sysroot=/path/to/staging "$@"
```

### Using `pkg-config` with target sysroot

Buildroot provides a cross `pkg-config` wrapper that points to the target's `.pc` files:

```makefile
# In your .mk file — Buildroot sets this automatically
# PKG_CONFIG_SYSROOT_DIR = $(STAGING_DIR)
# PKG_CONFIG_LIBDIR      = $(STAGING_DIR)/usr/lib/pkgconfig

# CMake finds it via toolchain file; but for manual use:
MYCPPAPP_CONF_ENV += \
    PKG_CONFIG="$(PKG_CONFIG_HOST_BINARY)"   \
    PKG_CONFIG_SYSROOT_DIR="$(STAGING_DIR)"  \
    PKG_CONFIG_LIBDIR="$(STAGING_DIR)/usr/lib/pkgconfig:$(STAGING_DIR)/usr/share/pkgconfig"
```

### `CMakeLists.txt` — Finding and linking a sysroot library

```cmake
# ── Find zlib (target sysroot version) ───────────────────────
find_package(ZLIB REQUIRED)    # CMake's FindZLIB module
                                # CMAKE_FIND_ROOT_PATH ensures target is found

# ── Find a pkg-config based library ──────────────────────────
find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBFOO REQUIRED IMPORTED_TARGET libfoo>=1.0)

# ── Link everything together ──────────────────────────────────
target_link_libraries(mycppapp
    PRIVATE
        ZLIB::ZLIB
        PkgConfig::LIBFOO
)
```

### Manual linking example (when CMake find modules don't exist)

```cmake
# Fallback: manually specify paths (Buildroot sysroot is CMAKE_SYSROOT)
target_include_directories(mycppapp PRIVATE
    ${CMAKE_SYSROOT}/usr/include/mylib
)
target_link_directories(mycppapp PRIVATE
    ${CMAKE_SYSROOT}/usr/lib
)
target_link_libraries(mycppapp PRIVATE mylib)
```

---

## Stripping Debug Symbols

Debug symbols can increase binary size by 10–50×. On embedded targets with limited flash/RAM, stripping is essential.

### Size comparison (ASCII)

```
  Binary size (example mycppapp):
  ┌──────────────────────────────────────────────────────┐
  │  With debug symbols:  2,847 KB  ████████████████████ │
  │  After strip:           184 KB  █▌                   │
  │  After strip + UPX:      98 KB  ▊                    │
  └──────────────────────────────────────────────────────┘
  Reduction: ~93% with strip alone
```

### Method 1: CMake Release build (automatic)

```makefile
# In .mk — Release build removes debug info and enables optimizations
MYCPPAPP_CONF_OPTS += -DCMAKE_BUILD_TYPE=Release
```

This sets `-O3 -DNDEBUG` and omits `-g`. However, it does NOT strip the binary of its symbol table.

### Method 2: Buildroot automatic stripping

Buildroot strips all installed binaries automatically as a post-install step. It uses:

```bash
$(TARGET_STRIP) --strip-unneeded <binary>
```

Where `$(TARGET_STRIP)` resolves to e.g. `arm-linux-gnueabihf-strip`.

**To opt out (rare — e.g., for gdbserver debugging):**

```makefile
MYCPPAPP_INSTALL_TARGET_OPTS = DESTDIR=$(TARGET_DIR) STRIP=""
```

Or globally in `BR2_STRIP_strip` (menuconfig → "Build options → strip target binaries").

### Method 3: CMake `install/strip` target

```cmake
# CMakeLists.txt — configure install with optional stripping
install(TARGETS mycppapp
    RUNTIME DESTINATION bin
)

# Alternatively, use CMake's STRIP_DURING_INSTALL
set_target_properties(mycppapp PROPERTIES
    INSTALL_STRIP_BINARIES ON
)
```

Or as a CMake option at configure time:

```makefile
MYCPPAPP_CONF_OPTS += -DCMAKE_INSTALL_DO_STRIP=1
```

### Method 4: Custom post-install hook in `.mk`

```makefile
define MYCPPAPP_STRIP_BINARY
    $(TARGET_STRIP) --strip-unneeded \
        $(TARGET_DIR)/usr/bin/mycppapp
endef

MYCPPAPP_POST_INSTALL_TARGET_HOOKS += MYCPPAPP_STRIP_BINARY
```

### What `--strip-unneeded` removes

```
  ELF Binary Sections:
  ┌────────────────────┬──────────────┬───────────────┐
  │ Section            │ strip-debug  │ strip-unneeded│
  ├────────────────────┼──────────────┼───────────────┤
  │ .text (code)       │ kept         │ kept          │
  │ .data / .bss       │ kept         │ kept          │
  │ .dynsym (dyn link) │ kept         │ kept          │
  │ .symtab (all syms) │ removed      │ removed       │
  │ .debug_info        │ removed      │ removed       │
  │ .debug_line        │ removed      │ removed       │
  │ .strtab            │ partial      │ partial       │
  └────────────────────┴──────────────┴───────────────┘
  Note: .dynsym kept → dynamic linking still works
```

---

## Rust Cross-Compilation in Buildroot

Since Buildroot 2020.02, Rust packages are supported natively via the `cargo-package` infrastructure. The workflow is analogous to CMake packages.

### `Config.in` for a Rust package

```kconfig
config BR2_PACKAGE_MYRUST_SENSOR
    bool "myrust-sensor"
    depends on BR2_PACKAGE_HOST_RUSTC
    depends on BR2_TOOLCHAIN_HAS_THREADS
    help
      A sensor data collector written in Rust.
```

### `myrust-sensor.mk`

```makefile
################################################################################
#
# myrust-sensor — Rust sensor application
#
################################################################################

MYRUST_SENSOR_VERSION       = 0.3.1
MYRUST_SENSOR_SITE          = https://github.com/example/myrust-sensor/archive/refs/tags
MYRUST_SENSOR_SITE_METHOD   = wget
MYRUST_SENSOR_SOURCE        = v$(MYRUST_SENSOR_VERSION).tar.gz
MYRUST_SENSOR_LICENSE       = MIT
MYRUST_SENSOR_LICENSE_FILES = LICENSE

# Cargo features to enable
MYRUST_SENSOR_CARGO_BUILD_OPTS += --features="embedded,no_std_compat"

$(eval $(cargo-package))
```

Buildroot automatically sets the Cargo target triple (e.g., `arm-unknown-linux-gnueabihf`), sets `CARGO_HOME`, and handles cross-compilation.

### Rust `Cargo.toml` for embedded cross-compilation

```toml
[package]
name    = "myrust-sensor"
version = "0.3.1"
edition = "2021"

[dependencies]
serde       = { version = "1.0", features = ["derive"], default-features = false }
serde_json  = { version = "1.0", default-features = false, features = ["alloc"] }
embedded-hal = "1.0"

[profile.release]
opt-level   = "z"     # optimize for size
lto         = true    # link-time optimization
codegen-units = 1     # better optimization, slower compile
strip       = true    # strip symbols (Rust 1.59+)
panic       = "abort" # smaller binary, no unwinding
```

### Rust cross-compilation source example

```rust
// src/main.rs — sensor data collector, cross-compiled for ARM Linux

use std::fs;
use std::path::Path;
use std::time::{Duration, Instant};
use std::thread;

/// Read a sysfs temperature sensor (common on embedded Linux)
fn read_temperature(hwmon_path: &Path) -> Result<f64, std::io::Error> {
    let raw = fs::read_to_string(hwmon_path.join("temp1_input"))?;
    let millidegrees: i64 = raw.trim().parse()
        .map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;
    Ok(millidegrees as f64 / 1000.0)
}

/// Exponential moving average filter
struct EmaFilter {
    alpha: f64,
    value: Option<f64>,
}

impl EmaFilter {
    fn new(alpha: f64) -> Self {
        Self { alpha, value: None }
    }

    fn update(&mut self, sample: f64) -> f64 {
        let v = match self.value {
            None    => sample,
            Some(p) => self.alpha * sample + (1.0 - self.alpha) * p,
        };
        self.value = Some(v);
        v
    }
}

fn main() {
    let hwmon = Path::new("/sys/class/hwmon/hwmon0");
    let mut filter = EmaFilter::new(0.1);
    let interval   = Duration::from_secs(1);

    println!("Sensor monitor started (Ctrl+C to stop)");
    println!("{:<10} {:<12} {:<12}", "Time(s)", "Raw °C", "Filtered °C");
    println!("{}", "-".repeat(36));

    let start = Instant::now();
    loop {
        match read_temperature(hwmon) {
            Ok(temp) => {
                let filtered = filter.update(temp);
                let elapsed  = start.elapsed().as_secs();
                println!("{:<10} {:<12.2} {:<12.2}", elapsed, temp, filtered);
            }
            Err(e) => eprintln!("Read error: {}", e),
        }
        thread::sleep(interval);
    }
}
```

---

## Full Working Examples

### C Example: Sensor utility with sysroot-linked zlib

```c
/* sensor_log.c — compress sensor data with zlib (target sysroot) */
#include <stdio.h>
#include <string.h>
#include <zlib.h>     /* from TARGET sysroot, linked by Buildroot */

int compress_log(const char *input, size_t in_len,
                 unsigned char *output, size_t *out_len)
{
    z_stream zs;
    memset(&zs, 0, sizeof(zs));

    if (deflateInit(&zs, Z_BEST_COMPRESSION) != Z_OK)
        return -1;

    zs.next_in   = (Bytef *)input;
    zs.avail_in  = (uInt)in_len;
    zs.next_out  = output;
    zs.avail_out = (uInt)*out_len;

    int ret = deflate(&zs, Z_FINISH);
    deflateEnd(&zs);

    if (ret != Z_STREAM_END)
        return -2;

    *out_len = zs.total_out;
    return 0;
}

int main(void) {
    const char *data    = "temp=23.5;hum=61;press=1013\n";
    unsigned char buf[256];
    size_t out_len      = sizeof(buf);

    printf("zlib version: %s\n", zlibVersion());

    if (compress_log(data, strlen(data), buf, &out_len) == 0)
        printf("Compressed %zu bytes → %zu bytes\n", strlen(data), out_len);
    else
        fprintf(stderr, "Compression failed\n");

    return 0;
}
```

### C++ Example: Full CMake-based application

#### `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.16)
project(mycppapp VERSION 1.2.0 LANGUAGES CXX)

# ── Standards ─────────────────────────────────────────────────
set(CMAKE_CXX_STANDARD          17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS        OFF)

# ── Dependencies ──────────────────────────────────────────────
find_package(ZLIB   REQUIRED)
find_package(Threads REQUIRED)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBFOO REQUIRED IMPORTED_TARGET libfoo>=1.0)

# ── Compile options ───────────────────────────────────────────
add_compile_options(
    -Wall -Wextra -Wno-unused-parameter
    -fstack-protector-strong
    $<$<CONFIG:Release>:-O2>
    $<$<CONFIG:Debug>:-g3 -O0>
)

# ── Library target ────────────────────────────────────────────
add_library(sensor_lib STATIC
    src/sensor.cpp
    src/protocol.cpp
    src/filter.cpp
)
target_include_directories(sensor_lib PUBLIC include/)
target_link_libraries(sensor_lib
    PUBLIC  Threads::Threads
    PRIVATE ZLIB::ZLIB PkgConfig::LIBFOO
)

# ── Executable ────────────────────────────────────────────────
add_executable(mycppapp src/main.cpp)
target_link_libraries(mycppapp PRIVATE sensor_lib)

# ── Install rules ─────────────────────────────────────────────
include(GNUInstallDirs)
install(TARGETS mycppapp RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(TARGETS sensor_lib
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
```

#### `src/sensor.hpp`

```cpp
// include/sensor.hpp
#pragma once

#include <cstdint>
#include <optional>
#include <string_view>
#include <chrono>

namespace mycppapp {

/// Represents a single sensor reading
struct SensorReading {
    float    temperature_c;
    float    humidity_pct;
    uint32_t pressure_pa;
    std::chrono::system_clock::time_point timestamp;
};

/// Abstract sensor interface (C++17 std::optional return)
class ISensor {
public:
    virtual ~ISensor() = default;

    [[nodiscard]]
    virtual std::optional<SensorReading> read() = 0;

    [[nodiscard]]
    virtual std::string_view name() const noexcept = 0;
};

/// Concrete Linux sysfs sensor implementation
class SysfsSensor final : public ISensor {
public:
    explicit SysfsSensor(std::string_view hwmon_path);

    std::optional<SensorReading> read() override;
    std::string_view name() const noexcept override;

private:
    std::string hwmon_path_;
    std::string name_;
};

} // namespace mycppapp
```

#### `src/sensor.cpp`

```cpp
// src/sensor.cpp — C++17, cross-compiled for target via Buildroot

#include "sensor.hpp"

#include <fstream>
#include <filesystem>
#include <charconv>
#include <stdexcept>

namespace mycppapp {

namespace fs = std::filesystem;

SysfsSensor::SysfsSensor(std::string_view hwmon_path)
    : hwmon_path_(hwmon_path)
{
    // Read sensor name from sysfs (C++17 filesystem)
    auto name_file = fs::path(hwmon_path_) / "name";
    if (std::ifstream f{name_file}) {
        std::getline(f, name_);
    } else {
        name_ = "unknown";
    }
}

std::string_view SysfsSensor::name() const noexcept {
    return name_;
}

/// Read a sysfs integer file and scale it
static std::optional<float> read_sysfs_float(
    const fs::path &base,
    const char     *filename,
    float           scale)
{
    auto path = base / filename;
    std::ifstream f{path};
    if (!f) return std::nullopt;

    std::string line;
    if (!std::getline(f, line)) return std::nullopt;

    int32_t raw{};
    auto [ptr, ec] = std::from_chars(
        line.data(), line.data() + line.size(), raw);

    if (ec != std::errc{}) return std::nullopt;

    return static_cast<float>(raw) * scale;
}

std::optional<SensorReading> SysfsSensor::read()
{
    fs::path base{hwmon_path_};

    auto temp = read_sysfs_float(base, "temp1_input",  0.001f); // milli-°C → °C
    auto hum  = read_sysfs_float(base, "humidity1",    0.001f);
    auto pres = read_sysfs_float(base, "pressure1",    1.0f);

    if (!temp || !hum || !pres) return std::nullopt;

    return SensorReading{
        .temperature_c = *temp,
        .humidity_pct  = *hum,
        .pressure_pa   = static_cast<uint32_t>(*pres),
        .timestamp     = std::chrono::system_clock::now()
    };
}

} // namespace mycppapp
```

#### `src/main.cpp`

```cpp
// src/main.cpp — entry point with C++17 structured bindings

#include "sensor.hpp"

#include <iostream>
#include <iomanip>
#include <thread>
#include <chrono>
#include <cstdlib>

using namespace mycppapp;
using namespace std::chrono_literals;

int main(int argc, char *argv[])
{
    const char *hwmon = (argc > 1) ? argv[1] : "/sys/class/hwmon/hwmon0";

    SysfsSensor sensor{hwmon};
    std::cout << "Monitoring: " << sensor.name() << "\n"
              << std::string(40, '-') << "\n";

    while (true) {
        if (auto reading = sensor.read()) {
            // C++17 structured bindings
            auto& [temp, hum, pres, ts] = *reading;

            auto t = std::chrono::system_clock::to_time_t(ts);
            std::cout
                << std::put_time(std::localtime(&t), "%T")
                << "  T=" << std::fixed << std::setprecision(1) << temp << "°C"
                << "  H=" << hum  << "%"
                << "  P=" << pres << "Pa"
                << "\n";
        } else {
            std::cerr << "Sensor read failed\n";
        }
        std::this_thread::sleep_for(1s);
    }
}
```

### Complete `.mk` File

```makefile
################################################################################
#
# mycppapp — C++17 sensor monitor, CMake, cross-compiled via Buildroot
#
################################################################################

MYCPPAPP_VERSION        = 1.2.0
MYCPPAPP_SITE           = $(call github,example,mycppapp,v$(MYCPPAPP_VERSION))
MYCPPAPP_SITE_METHOD    = wget
MYCPPAPP_SOURCE         = v$(MYCPPAPP_VERSION).tar.gz
MYCPPAPP_LICENSE        = MIT
MYCPPAPP_LICENSE_FILES  = LICENSE

MYCPPAPP_DEPENDENCIES   = zlib libfoo

MYCPPAPP_INSTALL_STAGING = YES
MYCPPAPP_INSTALL_TARGET  = YES

MYCPPAPP_CONF_OPTS = \
    -DCMAKE_BUILD_TYPE=Release              \
    -DCMAKE_CXX_STANDARD=17                 \
    -DCMAKE_CXX_STANDARD_REQUIRED=ON        \
    -DCMAKE_CXX_EXTENSIONS=OFF              \
    -DBUILD_SHARED_LIBS=OFF                 \
    -DMYCPPAPP_ENABLE_TESTS=OFF             \
    -DCMAKE_CXX_FLAGS="-fstack-protector-strong -D_FORTIFY_SOURCE=2"

# Strip the installed binary
define MYCPPAPP_STRIP_BINARY
    $(TARGET_STRIP) --strip-unneeded \
        $(TARGET_DIR)/usr/bin/mycppapp
endef
MYCPPAPP_POST_INSTALL_TARGET_HOOKS += MYCPPAPP_STRIP_BINARY

$(eval $(cmake-package))
```

---

## Troubleshooting

### Common errors and fixes

```
ERROR: Could not find libfoo for target
────────────────────────────────────────────────────────────
Cause:  libfoo not selected in Buildroot config, or .pc file
        not in $(STAGING_DIR)/usr/lib/pkgconfig
Fix:    Add BR2_PACKAGE_LIBFOO=y in .config
        Check: ls $(STAGING_DIR)/usr/lib/pkgconfig/libfoo.pc

ERROR: /usr/lib/x86_64-linux-gnu/libz.so found instead of ARM
────────────────────────────────────────────────────────────
Cause:  CMAKE_FIND_ROOT_PATH_MODE_LIBRARY not set to ONLY
        (missing or broken toolchain file)
Fix:    Verify toolchainfile.cmake contains ONLY mode
        Never pass -DZLIB_LIBRARY=/usr/lib/... manually

ERROR: undefined reference to `__cxa_throw'
────────────────────────────────────────────────────────────
Cause:  libstdc++ not installed on target or not linked
Fix:    Ensure BR2_INSTALL_LIBSTDCPP=y and BR2_STATIC_LIBSTDCPP
        if building fully static

ERROR: GLIBCXX_3.4.26 not found on target
────────────────────────────────────────────────────────────
Cause:  Binary compiled with newer toolchain than target's
        libstdc++.so
Fix:    Match toolchain version or static-link libstdc++:
        MYCPPAPP_CONF_OPTS += -DCMAKE_EXE_LINKER_FLAGS="-static-libstdc++"
```

### Useful debugging commands

```bash
# Inspect what libraries a cross-compiled binary needs
arm-linux-gnueabihf-readelf -d output/target/usr/bin/mycppapp | grep NEEDED

# Check if binary is stripped
arm-linux-gnueabihf-size output/target/usr/bin/mycppapp
file output/target/usr/bin/mycppapp
# "stripped" → good; "not stripped" → debug symbols present

# List all symbols (before stripping)
arm-linux-gnueabihf-nm output/build/mycppapp-1.2.0/mycppapp

# Verify C++ standard used
arm-linux-gnueabihf-strings output/target/usr/bin/mycppapp | grep GLIBCXX
```

---

## Summary

```
  ┌─────────────────────────────────────────────────────────────┐
  │          C++ PACKAGE CROSS-COMPILATION CHECKLIST            │
  ├─────────────────────────────────────────────────────────────┤
  │                                                             │
  │  Config.in                                                  │
  │  ├─ [✓] depends on BR2_INSTALL_LIBSTDCPP                    │
  │  ├─ [✓] depends on BR2_TOOLCHAIN_GCC_AT_LEAST_7 (C++17)     │
  │  └─ [✓] select dependencies (libfoo, zlib, ...)             │
  │                                                             │
  │  .mk file                                                   │
  │  ├─ [✓] $(eval $(cmake-package))                            │
  │  ├─ [✓] CMAKE_CXX_STANDARD=17                               │
  │  ├─ [✓] CMAKE_CXX_EXTENSIONS=OFF                            │
  │  ├─ [✓] CMAKE_BUILD_TYPE=Release                            │
  │  ├─ [✓] MYCPPAPP_DEPENDENCIES = zlib libfoo                 │
  │  └─ [✓] POST_INSTALL strip hook                             │
  │                                                             │
  │  CMakeLists.txt                                             │
  │  ├─ [✓] find_package(ZLIB REQUIRED)                         │
  │  ├─ [✓] CMAKE_FIND_ROOT_PATH_MODE = ONLY                    │
  │  │       (auto-set by Buildroot toolchain file)             │
  │  └─ [✓] install(TARGETS ...)                                │
  │                                                             │
  │  Stripping pipeline                                         │
  │  ├─ CMake Release build     → removes -g debug info         │
  │  ├─ Buildroot auto-strip    → runs TARGET_STRIP             │
  │  └─ --strip-unneeded        → removes .symtab, .debug_*     │
  │                                                             │
  │  Key commands                                               │
  │  ├─ make mycppapp           → build only this package       │
  │  ├─ make mycppapp-rebuild   → force full rebuild            │
  │  ├─ make mycppapp-dirclean  → delete build dir              │
  │  └─ BR2_JLEVEL=4 make      → parallel build (4 jobs)        │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

Cross-compiling C++ with Buildroot requires attention at every layer: the `Config.in` must guard against incompatible toolchains, the `.mk` file must correctly pass CMake options including the C++ standard, the CMake toolchain file enforces sysroot isolation so only target libraries are linked, and the strip pipeline ensures shipping binaries are as small as possible. Following this chapter's patterns enables reliable, reproducible embedded C++ builds with Buildroot's infrastructure doing most of the heavy lifting.

---

*Buildroot documentation: https://buildroot.org/docs.html*
*CMake cross-compilation: https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html*