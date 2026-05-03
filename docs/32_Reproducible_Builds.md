# 32. Reproducible Builds in Buildroot

**Topic: Buildroot Reproducible Builds** covers all five key areas you listed:

---

**`BR2_REPRODUCIBLE`** — The master Kconfig switch that activates path-stripping compiler flags (`-fdebug-prefix-map`, `-fmacro-prefix-map`), sorted file lists, hostname suppression, and UUID disabling. Shown in both `menuconfig` ASCII navigation and `defconfig` format.

**`SOURCE_DATE_EPOCH`** — How Buildroot derives this timestamp from the Git commit date and exports it to all sub-makes. C, C++, and Rust examples all show the correct pattern: injecting `BUILD_EPOCH` at compile time instead of using `__DATE__`/`__TIME__` or `SystemTime::now()`.

**Deterministic Toolchain Versions** — Two strategies (internal vs. external toolchain), both with exact version pinning. The `.hash` file mechanism is explained with a full `mylib.mk` + `mylib.hash` example. A C++ compile-time assertion template guards against accidental toolchain drift.

**`diffoscope`** — Installation, CLI usage for comparing `rootfs.ext4` images or individual ELF binaries, and an ASCII diagram of the two-build test workflow. Also includes a CI pipeline ASCII diagram with example GitLab YAML.

**Pinning Package Versions** — A layered strategy diagram covering Buildroot itself, external packages, toolchain, and host tools, plus a complete `myapp.mk` snippet for package authors that properly injects `SOURCE_DATE_EPOCH` and sorts `find` output.

The code examples include nine complete, annotated samples across C, C++, and Rust covering timestamps, deterministic iteration, hash verification, `build.rs` integration, `BTreeMap` vs `HashMap`, and toolchain enforcement.


## Introduction

Reproducible builds are a set of practices that ensure a software build process
always produces **bit-for-bit identical output** given the same source inputs,
regardless of when, where, or by whom the build is performed. In embedded Linux
development with Buildroot, reproducibility is critical for:

- **Security auditing**: Verifying that distributed binaries exactly match their
  declared source code, with no hidden modifications or injected malware.
- **Supply chain integrity**: Ensuring that firmware shipped to customers is
  provably the same as what was tested and signed off during development.
- **Regulatory compliance**: Meeting certification requirements (e.g., IEC 61508,
  DO-178C) that demand traceable, repeatable build artefacts.
- **Debugging**: Isolating whether a defect is in the source code or introduced
  by a non-deterministic build environment.

---

## 32.1 The Problem: Non-Determinism in Builds

Before exploring solutions, it is useful to understand the common sources of
non-determinism in a build:

```
  Common Sources of Build Non-Determinism
  =========================================

  +----------------------------+----------------------------------+
  |  Source                    |  Example                         |
  +----------------------------+----------------------------------+
  |  Timestamps                |  __DATE__, __TIME__ macros       |
  |  File system ordering      |  readdir() returns random order  |
  |  Locale / timezone         |  strftime() output varies        |
  |  Build path embedding      |  __FILE__ macro, DWARF paths     |
  |  Random seeds              |  ASLR, UUID generation           |
  |  Concurrency / race conds  |  Parallel link order             |
  |  Tool version drift        |  gcc 12.1 vs gcc 12.2            |
  |  Network fetched deps      |  Upstream tarball changes        |
  +----------------------------+----------------------------------+
```

Buildroot addresses each of these through a combination of configuration
options, conventions, and tooling.

---

## 32.2 `BR2_REPRODUCIBLE`: The Master Switch

The Buildroot Kconfig option `BR2_REPRODUCIBLE` activates a collection of
measures that together aim at bit-reproducible output.

### Enabling in `menuconfig`

```
  menuconfig Navigation
  ======================

  Build options  --->
    [*] Make the build reproducible (experimental)
         (BR2_REPRODUCIBLE)
```

### Effect on the Build System

When `BR2_REPRODUCIBLE=y` is set, Buildroot:

1. Passes `-fdebug-prefix-map` and `-fmacro-prefix-map` to GCC to strip
   absolute build paths from DWARF debug info and macro expansions.
2. Exports `SOURCE_DATE_EPOCH` to all sub-makes and package build scripts.
3. Sorts file lists fed to `tar`, `find`, and similar tools.
4. Disables build-time generation of UUIDs where possible.
5. Prevents embedding the build hostname into artefacts.

### In `defconfig`

```ini
# configs/my_board_defconfig
BR2_REPRODUCIBLE=y
```

---

## 32.3 `SOURCE_DATE_EPOCH`

`SOURCE_DATE_EPOCH` is a UNIX timestamp (seconds since 1970-01-01 00:00:00 UTC)
that build tools should use **instead of the current wall-clock time** when
embedding timestamps in build artefacts. It is defined by the
[reproducible-builds.org](https://reproducible-builds.org/specs/source-date-epoch/)
specification and is now supported by GCC, GNU tar, GNU binutils, Python, Rust,
and many other tools.

### How Buildroot Sets It

Buildroot derives `SOURCE_DATE_EPOCH` from the commit date of the Buildroot Git
repository itself (or from a fixed value you supply):

```makefile
# In Buildroot's Makefile (simplified)
ifndef SOURCE_DATE_EPOCH
  SOURCE_DATE_EPOCH := $(shell git -C $(TOPDIR) log -1 --format=%ct 2>/dev/null)
endif
export SOURCE_DATE_EPOCH
```

You can override it in your environment:

```bash
export SOURCE_DATE_EPOCH=1700000000
make BR2_REPRODUCIBLE=y
```

### Verifying it Propagates

```bash
# After a build, inspect embedded timestamps in a compiled ELF
readelf -p .comment output/target/usr/bin/myapp | grep "GCC\|clang"
# Should not contain wall-clock build dates

# Check an archive's timestamps
tar tvf output/images/rootfs.tar | head -20
# All timestamps should equal SOURCE_DATE_EPOCH
```

---

## 32.4 Deterministic Toolchain Versions

Pinning exact toolchain versions is essential. Even a patch-level compiler
upgrade can change code generation, symbol ordering, or debug info layout,
breaking binary reproducibility.

### Buildroot's External Toolchain Approach

```
  Toolchain Pinning Strategy
  ===========================

  Option A: Buildroot Internal Toolchain (pinned via hash)
  ---------------------------------------------------------
  BR2_TOOLCHAIN_BUILDROOT=y
  BR2_GCC_VERSION_12_X=y        # Pin major version
  BR2_BINUTILS_VERSION_2_40_X=y
  BR2_GLIBC_VERSION_2_37_X=y

  Option B: External Pre-built Toolchain (recommended for production)
  -------------------------------------------------------------------
  BR2_TOOLCHAIN_EXTERNAL=y
  BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
  BR2_TOOLCHAIN_EXTERNAL_PATH="/opt/arm-cortexa9-linux-gnueabihf-12.3"
  # Hash-verify the toolchain tarball in a wrapper script
```

### Locking Package Versions

Every Buildroot package has a version variable and a hash file:

```
  package/
  └── mylib/
      ├── mylib.mk          # Declares MYLIB_VERSION and MYLIB_SOURCE
      └── mylib.hash        # SHA-256 / SHA-1 of the downloaded tarball
```

```makefile
# package/mylib/mylib.mk
MYLIB_VERSION  = 2.4.1
MYLIB_SITE     = https://example.com/releases
MYLIB_SOURCE   = mylib-$(MYLIB_VERSION).tar.gz
MYLIB_LICENSE  = MIT

$(eval $(generic-package))
```

```
# package/mylib/mylib.hash
# Mandatory: sha256 of the tarball, sha256 of the licence
sha256  e3b0c44298fc1c149afb...  mylib-2.4.1.tar.gz
sha256  da9c11a4fc9a7b...        LICENSE
```

Buildroot **verifies hashes before extracting** every downloaded tarball. If the
hash does not match, the build fails immediately — preventing silent upstream
tarball substitution attacks.

---

## 32.5 `diffoscope` for Binary Comparison

`diffoscope` is a powerful open-source tool that performs **deep, recursive
comparison** of build artefacts: ELF binaries, archives, filesystem images,
PDF files, and more. It is the standard tool for diagnosing reproducibility
failures.

### Conceptual Workflow

```
  Two-Build Reproducibility Test
  ================================

    Build #1  (on machine A, time T1)         Build #2  (on machine B, time T2)
    +---------------------------------+        +---------------------------------+
    |  make defconfig                 |        |  make defconfig                 |
    |  make BR2_REPRODUCIBLE=y        |        |  make BR2_REPRODUCIBLE=y        |
    |                                 |        |                                 |
    |  output/images/rootfs.ext4  ----+---.    |  output/images/rootfs.ext4  ----+---.
    +---------------------------------+   |    +---------------------------------+   |
                                          |                                          |
                                          v                                          v
                                     +--------+   +--------+
                                     | Image1 |   | Image2 |
                                     +--------+   +--------+
                                          |             |
                                          +------+------+
                                                 |
                                           diffoscope
                                                 |
                                    +-------------------------+
                                    | IDENTICAL: PASS         |
                                    | or                      |
                                    | DIFFERENCES FOUND: FAIL |
                                    +-------------------------+
```

### Installing `diffoscope`

```bash
# On the host build machine
pip install diffoscope
# or
apt-get install diffoscope   # Debian / Ubuntu
```

### Basic Usage

```bash
# Compare two rootfs images
diffoscope \
  build1/output/images/rootfs.ext4 \
  build2/output/images/rootfs.ext4 \
  --html report.html \
  --text report.txt

# Compare individual ELF binaries
diffoscope \
  build1/output/target/usr/bin/myapp \
  build2/output/target/usr/bin/myapp
```

### Reading `diffoscope` Output

```
  Sample diffoscope Text Output
  ==============================

  --- build1/.../myapp
  +++ build2/.../myapp

  ├── readelf --debug-dump=info {}
  │   ┄ /home/builder1/work/myapp.c        <-- absolute path leaked in DWARF
  │   vs
  │   /home/builder2/work/myapp.c

  ├── strings {}
  │   + __DATE__=Jan 15 2024               <-- timestamp macro embedded
  │   - __DATE__=Jan 16 2024
```

---

## 32.6 Pinning Package Versions: A Complete Strategy

### Snapshot the Entire Package Version Matrix

Use Buildroot's `make savedefconfig` and version pinning to create a fully
reproducible snapshot:

```
  Version Pinning Layers
  =======================

     Layer 1: Buildroot itself
     -------------------------
     Pin to a specific Git tag or commit SHA:
       git clone --branch 2024.02 https://git.buildroot.net/buildroot

     Layer 2: External packages (br2-external)
     ------------------------------------------
     Each package specifies exact VERSION + HASH

     Layer 3: Toolchain
     -------------------
     Use a versioned, hash-verified external toolchain tarball

     Layer 4: Host tools
     --------------------
     Use a reproducible host sysroot or Docker image
```

---

## 32.7 C Code Examples

### Example 1: Avoiding `__DATE__` and `__TIME__`

```c
/* BAD: embeds non-reproducible wall-clock timestamp */
#include <stdio.h>

void print_build_info_bad(void) {
    printf("Built: %s %s\n", __DATE__, __TIME__);
}

/* GOOD: use SOURCE_DATE_EPOCH injected at compile time */
/* In your package Makefile:                                          */
/*   TARGET_CFLAGS += -DBUILD_EPOCH=$(SOURCE_DATE_EPOCH)              */

#include <stdio.h>
#include <time.h>

#ifndef BUILD_EPOCH
#  error "BUILD_EPOCH must be defined via -DBUILD_EPOCH=<unix_ts>"
#endif

void print_build_info_good(void) {
    time_t epoch = (time_t)BUILD_EPOCH;
    char   buf[32];
    struct tm *t = gmtime(&epoch);
    strftime(buf, sizeof(buf), "%Y-%m-%dT%H:%M:%SZ", t);
    printf("Built: %s\n", buf);  /* Deterministic: derived from SOURCE_DATE_EPOCH */
}
```

### Example 2: Deterministic File Enumeration

`readdir()` returns directory entries in file-system order, which is
non-deterministic on many file systems. Sort explicitly:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>

/* Compare function for qsort */
static int cmp_str(const void *a, const void *b) {
    return strcmp(*(const char **)a, *(const char **)b);
}

/**
 * List directory entries in deterministic (sorted) order.
 * Non-reproducible builds often iterate directories unsorted,
 * causing link order or archive membership to vary between runs.
 */
void list_dir_deterministic(const char *path) {
    DIR    *d = opendir(path);
    struct  dirent *e;
    char  **names = NULL;
    size_t  count = 0, capacity = 0;

    if (!d) { perror("opendir"); return; }

    while ((e = readdir(d)) != NULL) {
        if (e->d_name[0] == '.') continue;   /* skip hidden / . .. */

        if (count >= capacity) {
            capacity = capacity ? capacity * 2 : 16;
            names = realloc(names, capacity * sizeof(*names));
        }
        names[count++] = strdup(e->d_name);
    }
    closedir(d);

    /* Sort for determinism */
    qsort(names, count, sizeof(*names), cmp_str);

    for (size_t i = 0; i < count; i++) {
        printf("%s\n", names[i]);
        free(names[i]);
    }
    free(names);
}
```

### Example 3: Hash-Verifying a Downloaded File in C

This illustrates the same logic Buildroot uses internally to verify package
tarballs against their `.hash` file entries.

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <openssl/evp.h>

/**
 * Compute SHA-256 of a file and compare against an expected hex digest.
 * Returns 0 on match, -1 on mismatch or error.
 * Mirrors what Buildroot's support/scripts/pkg-download does.
 */
int verify_sha256(const char *filepath, const char *expected_hex) {
    FILE          *f    = fopen(filepath, "rb");
    EVP_MD_CTX    *ctx  = EVP_MD_CTX_new();
    unsigned char  digest[EVP_MAX_MD_SIZE];
    unsigned int   dlen = 0;
    char           hexbuf[65] = {0};
    uint8_t        buf[65536];
    size_t         n;

    if (!f || !ctx) { return -1; }

    EVP_DigestInit_ex(ctx, EVP_sha256(), NULL);

    while ((n = fread(buf, 1, sizeof(buf), f)) > 0)
        EVP_DigestUpdate(ctx, buf, n);

    EVP_DigestFinal_ex(ctx, digest, &dlen);
    EVP_MD_CTX_free(ctx);
    fclose(f);

    /* Convert digest bytes to hex string */
    for (unsigned int i = 0; i < dlen; i++)
        snprintf(hexbuf + i * 2, 3, "%02x", digest[i]);

    if (strcmp(hexbuf, expected_hex) != 0) {
        fprintf(stderr, "HASH MISMATCH for %s\n"
                        "  expected: %s\n"
                        "  got:      %s\n",
                filepath, expected_hex, hexbuf);
        return -1;
    }

    printf("Hash OK: %s\n", filepath);
    return 0;
}
```

---

## 32.8 C++ Code Examples

### Example 4: Reproducible Version Metadata Class

```cpp
// version_info.hpp
#pragma once
#include <cstdint>
#include <string>
#include <ctime>

/**
 * Encapsulates build metadata derived from SOURCE_DATE_EPOCH.
 * The BUILD_EPOCH macro must be injected by the build system via:
 *   TARGET_CXXFLAGS += -DBUILD_EPOCH=$(SOURCE_DATE_EPOCH)
 *   TARGET_CXXFLAGS += -DGIT_COMMIT=\"$(shell git rev-parse --short HEAD)\"
 */
class VersionInfo {
public:
    static constexpr uint64_t epoch() noexcept {
#ifdef BUILD_EPOCH
        return static_cast<uint64_t>(BUILD_EPOCH);
#else
        static_assert(false, "BUILD_EPOCH must be defined by the build system");
        return 0;
#endif
    }

    static std::string iso8601() {
        time_t t = static_cast<time_t>(epoch());
        char   buf[32];
        struct tm *tm = gmtime(&t);
        strftime(buf, sizeof(buf), "%Y-%m-%dT%H:%M:%SZ", tm);
        return {buf};
    }

    static const char *git_commit() noexcept {
#ifdef GIT_COMMIT
        return GIT_COMMIT;
#else
        return "unknown";
#endif
    }
};
```

```cpp
// main.cpp — usage example
#include <iostream>
#include "version_info.hpp"

int main() {
    std::cout << "Build date (ISO-8601): " << VersionInfo::iso8601() << "\n";
    std::cout << "Git commit:            " << VersionInfo::git_commit() << "\n";
    std::cout << "Epoch:                 " << VersionInfo::epoch() << "\n";
    return 0;
}
```

### Example 5: Deterministic Map Iteration for Config Serialisation

Maps in C++ are ordered, but `unordered_map` is not — using it for serialised
configuration output introduces non-determinism:

```cpp
#include <iostream>
#include <map>          /* ordered  — deterministic output */
#include <string>
#include <sstream>
// #include <unordered_map>  /* AVOID for serialised output */

/**
 * Serialise build configuration to a canonical string.
 * std::map guarantees lexicographic key order, giving the
 * same output regardless of insertion order or platform.
 */
std::string serialise_config(
    const std::map<std::string, std::string> &cfg)
{
    std::ostringstream oss;
    for (const auto &[key, val] : cfg) {
        oss << key << "=" << val << "\n";
    }
    return oss.str();
}

int main() {
    std::map<std::string, std::string> config = {
        {"BR2_ARCH",         "arm"},
        {"BR2_REPRODUCIBLE", "y"},
        {"BR2_TARGET_ARCH",  "cortex-a9"},
    };
    std::cout << serialise_config(config);
    /* Output is always alphabetically sorted — reproducible */
    return 0;
}
```

### Example 6: Compile-Time Assertions for Toolchain Version

```cpp
// toolchain_check.hpp
#pragma once

/**
 * Enforce a minimum and exact GCC version at compile time.
 * Place in a header included by every translation unit.
 * This ensures the build fails visibly if the wrong toolchain
 * is used — preventing silent reproducibility breakage.
 */

#define REQUIRED_GCC_MAJOR 12
#define REQUIRED_GCC_MINOR  3

#if !defined(__GNUC__) && !defined(__clang__)
#  error "Only GCC and Clang are supported for reproducible builds"
#endif

#ifdef __GNUC__
#  if __GNUC__ != REQUIRED_GCC_MAJOR || __GNUC_MINOR__ != REQUIRED_GCC_MINOR
#    error "Toolchain version mismatch! Reproducibility requires exactly GCC 12.3"
#  endif
#endif
```

---

## 32.9 Rust Code Examples

### Example 7: `build.rs` — Injecting `SOURCE_DATE_EPOCH`

Rust's Cargo build script (`build.rs`) is the standard place to forward
`SOURCE_DATE_EPOCH` into compiled code:

```rust
// build.rs
use std::env;

fn main() {
    // Forward SOURCE_DATE_EPOCH from the environment to the Rust compiler.
    // Buildroot exports this automatically when BR2_REPRODUCIBLE=y.
    let epoch = env::var("SOURCE_DATE_EPOCH")
        .unwrap_or_else(|_| {
            // Fallback: use a build-time panic rather than silently
            // embedding the current time, which breaks reproducibility.
            panic!(
                "SOURCE_DATE_EPOCH is not set. \
                 Set it explicitly or enable BR2_REPRODUCIBLE=y."
            )
        });

    // Parse to validate it is a valid u64 timestamp
    let _: u64 = epoch
        .trim()
        .parse()
        .expect("SOURCE_DATE_EPOCH must be a non-negative integer");

    // Expose as a compile-time environment variable accessible via env!()
    println!("cargo:rustc-env=BUILD_EPOCH={}", epoch);

    // Prevent Cargo from re-running this script on every build
    // unless SOURCE_DATE_EPOCH changes
    println!("cargo:rerun-if-env-changed=SOURCE_DATE_EPOCH");
}
```

```rust
// src/version.rs
/// Returns the build timestamp as a Unix epoch integer.
/// This value is deterministic: it comes from SOURCE_DATE_EPOCH,
/// not from std::time::SystemTime::now().
pub fn build_epoch() -> u64 {
    env!("BUILD_EPOCH")
        .parse::<u64>()
        .expect("BUILD_EPOCH must be a valid u64")
}

/// Returns the build timestamp as an ISO-8601 string.
pub fn build_date_iso8601() -> String {
    use std::time::{Duration, UNIX_EPOCH};
    let epoch = UNIX_EPOCH + Duration::from_secs(build_epoch());
    // Format without chrono to keep dependencies minimal
    let secs  = build_epoch();
    let days  = secs / 86400;
    // (simplified — use chrono or time crate in real projects)
    format!("epoch+{}d", days)
}
```

### Example 8: Deterministic HashMap in Rust

Rust's `HashMap` uses a randomised hasher (`SipHash` seeded with OS entropy)
by default. For reproducible output, use a deterministic hasher:

```rust
// Cargo.toml dependency needed:
//   [dependencies]
//   indexmap = "2"      # ordered, deterministic iteration

use std::collections::BTreeMap;  // Always sorted by key — deterministic

/// Generates a canonical key=value configuration string.
/// BTreeMap ensures the output order is always the same regardless
/// of insertion order — a requirement for reproducible config artefacts.
fn serialise_config(config: &BTreeMap<String, String>) -> String {
    config
        .iter()
        .map(|(k, v)| format!("{}={}\n", k, v))
        .collect()
}

fn main() {
    let mut config = BTreeMap::new();
    config.insert("BR2_ARCH".to_string(),         "arm".to_string());
    config.insert("BR2_REPRODUCIBLE".to_string(), "y".to_string());
    config.insert("BR2_TARGET_ARCH".to_string(),  "cortex-a9".to_string());

    let output = serialise_config(&config);
    println!("{}", output);
    // Always prints keys in alphabetical order — reproducible
}
```

### Example 9: Verifying a Downloaded Package Hash in Rust

```rust
use sha2::{Digest, Sha256};
use std::io::{self, Read};
use std::fs::File;

/// Verifies the SHA-256 digest of a file against an expected hex string.
/// Mirrors the hash-checking logic in Buildroot's download infrastructure.
///
/// Returns Ok(()) on success, Err with a description on failure.
pub fn verify_sha256(path: &str, expected_hex: &str) -> Result<(), String> {
    let mut file = File::open(path)
        .map_err(|e| format!("Cannot open {}: {}", path, e))?;

    let mut hasher = Sha256::new();
    let mut buf    = [0u8; 65536];

    loop {
        let n = file.read(&mut buf)
            .map_err(|e| format!("Read error: {}", e))?;
        if n == 0 { break; }
        hasher.update(&buf[..n]);
    }

    let digest = format!("{:x}", hasher.finalize());

    if digest != expected_hex {
        return Err(format!(
            "Hash mismatch for {}:\n  expected: {}\n  got:      {}",
            path, expected_hex, digest
        ));
    }

    println!("Hash OK: {}", path);
    Ok(())
}

fn main() -> io::Result<()> {
    let result = verify_sha256(
        "mylib-2.4.1.tar.gz",
        "e3b0c44298fc1c149afb4c8996fb92427ae41e4649b934ca495991b7852b855",
    );
    match result {
        Ok(())   => println!("Download verified."),
        Err(msg) => eprintln!("ERROR: {}", msg),
    }
    Ok(())
}
```

---

## 32.10 Complete Buildroot Workflow: End-to-End

```
  Reproducible Build Workflow
  ============================

  1. Pin Buildroot version
  ┌─────────────────────────────────────────────────────────┐
  │  git clone --branch 2024.02.3                           │
  │      https://git.buildroot.net/buildroot                │
  └──────────────────────────────────┬──────────────────────┘
                                     │
  2. Write defconfig                 │
  ┌──────────────────────────────────▼──────────────────────┐
  │  BR2_REPRODUCIBLE=y                                     │
  │  BR2_TOOLCHAIN_EXTERNAL=y                               │
  │  BR2_TOOLCHAIN_EXTERNAL_PATH="/opt/gcc-12.3-arm"        │
  └──────────────────────────────────┬──────────────────────┘
                                     │
  3. Set SOURCE_DATE_EPOCH           │
  ┌──────────────────────────────────▼──────────────────────┐
  │  export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)    │
  └──────────────────────────────────┬──────────────────────┘
                                     │
  4. Build twice (clean environments)│
  ┌──────────────────────────────────▼──────────────────────┐
  │  make -C /path/to/br O=/build1 myboard_defconfig && \   │
  │      make -C /path/to/br O=/build1                      │
  │                                                         │
  │  make -C /path/to/br O=/build2 myboard_defconfig && \   │
  │      make -C /path/to/br O=/build2                      │
  └──────────────────────────────────┬──────────────────────┘
                                     │
  5. Compare with diffoscope         │
  ┌──────────────────────────────────▼──────────────────────┐
  │  diffoscope /build1/images/rootfs.ext4 \                │
  │             /build2/images/rootfs.ext4 \                │
  │             --html report.html                          │
  └──────────────────────────────────┬──────────────────────┘
                                     │
                          ┌──────────┴──────────┐
                          │                     │
                   ┌──────▼──────┐       ┌──────▼──────┐
                   │  IDENTICAL  │       │ DIFFERENCES │
                   │   + PASS    │       │   - FAIL    │
                   └─────────────┘       └──────┬──────┘
                                                │
                                   ┌────────────▼─────────────┐
                                   │ Investigate with         │
                                   │ diffoscope HTML report   │
                                   │ Fix: timestamps, paths,  │
                                   │ sort order, tool version │
                                   └──────────────────────────┘
```

---

## 32.11 Makefile Snippets for Package Authors

If you maintain a custom Buildroot package (`br2-external`), these patterns
ensure your package participates correctly in reproducible builds:

```makefile
# package/myapp/myapp.mk

MYAPP_VERSION = 1.3.2
MYAPP_SITE    = https://releases.example.com/myapp
MYAPP_SOURCE  = myapp-$(MYAPP_VERSION).tar.gz
MYAPP_LICENSE = Apache-2.0

# Inject SOURCE_DATE_EPOCH and strip build paths
define MYAPP_CONFIGURE_CMDS
    $(MAKE) -C $(@D) configure \
        CC="$(TARGET_CC)" \
        CFLAGS="$(TARGET_CFLAGS) \
            -DBUILD_EPOCH=$(SOURCE_DATE_EPOCH) \
            -fdebug-prefix-map=$(BUILD_DIR)=. \
            -fmacro-prefix-map=$(BUILD_DIR)=."
endef

# Ensure file lists are sorted (deterministic archive membership)
define MYAPP_INSTALL_TARGET_CMDS
    find $(STAGING_DIR)/usr/lib/myapp -name '*.so' | sort | \
        xargs install -m 0755 -t $(TARGET_DIR)/usr/lib/myapp
endef

$(eval $(generic-package))
```

---

## 32.12 diffoscope Integration in CI

```
  CI Pipeline with Reproducibility Gate
  =======================================

  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  git push    │───►│  Build #1    │    │  Build #2    │
  └──────────────┘    │  (agent A)   │    │  (agent B)   │
                      └──────┬───────┘    └──────┬───────┘
                             │                   │
                             │  rootfs.ext4      │  rootfs.ext4
                             └─────────┬─────────┘
                                       │
                                ┌──────▼──────┐
                                │ diffoscope  │
                                └──────┬──────┘
                                       │
                          ┌────────────┴───────────────┐
                          │ exit 0: identical          │
                          │   → mark pipeline green    │
                          │                            │
                          │ exit 1: differences found  │
                          │   → publish HTML report    │
                          │   → block merge            │
                          └────────────────────────────┘

  Example GitLab CI YAML:
  -----------------------
  reproducibility-check:
    stage: verify
    script:
      - export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
      - make O=$CI_PROJECT_DIR/build1 myboard_defconfig && make O=build1
      - make O=$CI_PROJECT_DIR/build2 myboard_defconfig && make O=build2
      - diffoscope build1/images/rootfs.ext4 build2/images/rootfs.ext4
            --html artifacts/report.html || exit 1
    artifacts:
      paths: [artifacts/report.html]
      when:  on_failure
```

---

## Summary

Reproducible builds in Buildroot rest on four interconnected pillars:

```
  The Four Pillars of Buildroot Reproducibility
  ===============================================

  ┌──────────────────┐  ┌──────────────────┐
  │  BR2_REPRODUCIBLE│  │SOURCE_DATE_EPOCH │
  │                  │  │                  │
  │  Master Kconfig  │  │  Canonical time  │
  │  switch enabling │  │  signal replacing│
  │  all downstream  │  │  wall-clock in   │
  │  measures        │  │  all sub-tools   │
  └────────┬─────────┘  └────────┬─────────┘
           │                     │
           └──────────┬──────────┘
                      │
        ┌─────────────▼─────────────┐
        │                           │
  ┌─────▼─────────┐        ┌────────▼───────┐
  │  Deterministic│        │  diffoscope    │
  │  Toolchain &  │        │                │
  │  Package      │        │  Verification  │
  │  Version      │        │  tool to detect│
  │  Pinning      │        │  regressions   │
  └───────────────┘        └────────────────┘
```

| Pillar | Mechanism | Impact |
|---|---|---|
| `BR2_REPRODUCIBLE` | Kconfig flag | Activates path stripping, sorted file lists, hostname suppression |
| `SOURCE_DATE_EPOCH` | Environment variable | Replaces wall-clock time in GCC, tar, Python, Rust, etc. |
| Toolchain/package pinning | `.hash` files + version variables | Prevents silent upstream changes |
| `diffoscope` | Binary diff tool | Detects and diagnoses any remaining non-determinism |

Together these mechanisms make it possible to verify that a firmware image
distributed to a customer is provably identical to the image built and tested
from audited source code — a foundational guarantee for secure, trustworthy
embedded Linux systems.

---

*References:*
- [Reproducible Builds Project](https://reproducible-builds.org/)
- [Buildroot Manual — Reproducible Builds](https://buildroot.org/downloads/manual/manual.html)
- [diffoscope Documentation](https://diffoscope.org/)
- [SOURCE_DATE_EPOCH Specification](https://reproducible-builds.org/specs/source-date-epoch/)