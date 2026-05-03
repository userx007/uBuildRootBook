# 27. ptest — Runtime Package Testing in Buildroot

**Structure at a glance:**

- **Introduction** — what ptest is, why runtime testing on the target matters (endianness, ABI, kernel quirks)
- **Architecture Overview** — ASCII diagram showing the full flow from `menuconfig` on the host down to `ptest-runner` on the target
- **Enabling ptest** — the two-level config (`BR2_PACKAGE_PTEST_RUNNER` + per-package `BR2_PACKAGE_*_INSTALL_TESTS`) and how to verify the installed tree
- **ptest-runner** — how it works internally, with a simplified C source sketch of its scan/exec loop and all CLI flags
- **C harness** — a self-contained plain-C test program with hand-rolled `PASS`/`FAIL` macros and a `run-ptest` wrapper; zero extra dependencies
- **C++ / Catch2** — a full `TEST_CASE`/`SECTION` example with `--reporter tap` output, plus the `.mk` snippet to build and install it
- **Rust** — `#[test]` unit tests in `lib.rs`, a separate integration test crate, `Cargo.toml`, a shell `run-ptest` translator, and the `.mk` wildcard-safe install hook
- **Package integration** — complete `Config.in` and `libfoo.mk` templates ready to copy
- **Running on target** — ASCII filesystem layout, execution flow diagram, and a sample terminal session
- **Output format & CI parsing** — a Python script that parses `START:`/`END:` blocks and exits non-zero on any failure
- **Troubleshooting** — empty ptest dirs, wrong permissions, Rust hash-in-filename, Catch2 pkg-config issues
- **Summary** — a compact ASCII reference table of every key fact

---

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Enabling ptest in Buildroot](#enabling-ptest-in-buildroot)
4. [The ptest-runner](#the-ptest-runner)
5. [Writing ptest Harnesses — C (Plain)](#writing-ptest-harnesses--c-plain)
6. [Writing ptest Harnesses — C++ with Catch2](#writing-ptest-harnesses--c-with-catch2)
7. [Writing ptest Harnesses — Rust](#writing-ptest-harnesses--rust)
8. [Integrating ptest into a Buildroot Package](#integrating-ptest-into-a-buildroot-package)
9. [Running ptest on the Target](#running-ptest-on-the-target)
10. [Output Format and Interpretation](#output-format-and-interpretation)
11. [Troubleshooting](#troubleshooting)
12. [Summary](#summary)

---

## Introduction

`ptest` (Package Test) is a standardised runtime testing framework used by Buildroot (and originally by the Yocto Project / OpenEmbedded) to ship self-tests *inside the target root filesystem*. Unlike host-side unit tests that run during the build, ptest harnesses execute on the actual target hardware or QEMU image and report results in a machine-readable TAP (Test Anything Protocol) or simple pass/fail format.

Key goals:

- Verify that a package behaves correctly on the *target* architecture (endianness, ABI, kernel version, etc.)
- Provide a uniform runner interface regardless of the underlying test framework (CUnit, Catch2, Rust `#[test]`, Python `unittest`, etc.)
- Enable automated QA pipelines that boot an image and harvest test results over serial or SSH

---

## Architecture Overview

```
+--------------------------------------------------+
|               Build Host                         |
|                                                  |
|  menuconfig                                      |
|    └─ BR2_PACKAGE_LIBFOO_INSTALL_TESTS=y         |
|                                                  |
|  make libfoo                                     |
|    ├─ Build libfoo.so / libfoo binary            |
|    ├─ Build test binaries                        |
|    └─ Install to staging / target:               |
|         /usr/lib/ptest/libfoo/                   |
|           ├── run-ptest          (shell script)  |
|           ├── test_core          (ELF binary)    |
|           └── test_data/        (fixtures)       |
+--------------------------------------------------+
          |  flash / boot image
          v
+--------------------------------------------------+
|               Target Device / QEMU               |
|                                                  |
|  $ ptest-runner                                  |
|      └─ scans /usr/lib/ptest/*/run-ptest         |
|           └─ executes each run-ptest script      |
|                └─ collects TAP / pass/fail lines |
|                                                  |
|  Output (TAP stream):                            |
|    START: libfoo                                 |
|    PASS: test_addition                           |
|    FAIL: test_overflow                           |
|    END: libfoo                                   |
+--------------------------------------------------+
```

Every package that opts in installs a small directory under `/usr/lib/ptest/<pkgname>/`. The key file is the `run-ptest` shell script, which is the single entry point that `ptest-runner` calls.

---

## Enabling ptest in Buildroot

### Step 1 — Global ptest support

```
make menuconfig
```

Navigate to:

```
Build options
  └─ [*] Enable ptest framework support   (BR2_PACKAGE_PTEST_RUNNER)
```

This pulls in the `ptest-runner` binary and ensures the `/usr/lib/ptest` directory is created.

### Step 2 — Per-package test installation

Each package exposes its own Kconfig symbol:

```
Target packages
  └─ Libraries
       └─ [*] libfoo
            └─ [*] Install libfoo tests   (BR2_PACKAGE_LIBFOO_INSTALL_TESTS)
```

The naming convention is always:

```
BR2_PACKAGE_<PKGNAME_UPPER>_INSTALL_TESTS
```

### Step 3 — Build

```bash
make libfoo
# or rebuild everything:
make
```

Inspect the installed test tree:

```bash
find output/target/usr/lib/ptest -type f | sort

# Expected:
# output/target/usr/lib/ptest/libfoo/run-ptest
# output/target/usr/lib/ptest/libfoo/test_core
# output/target/usr/lib/ptest/libfoo/test_data/sample.txt
```

---

## The ptest-runner

`ptest-runner` is a small C program (originally from the Yocto world, adopted by Buildroot) that:

1. Scans `/usr/lib/ptest/` for subdirectories containing a `run-ptest` script
2. `chdir`s into each subdirectory (so relative paths inside tests work)
3. Executes `./run-ptest` and streams its stdout
4. Wraps output with `START: <pkg>` / `END: <pkg>` markers

Invoke it on the target:

```bash
# Run all installed ptests
ptest-runner

# Run a specific package's tests
ptest-runner -p libfoo

# Set a per-test timeout (seconds)
ptest-runner -t 60

# Save results to a file
ptest-runner 2>&1 | tee /tmp/ptest-results.txt
```

Source sketch of `ptest-runner`'s core loop (simplified for illustration):

```c
/* ptest-runner core loop — simplified */
#include <dirent.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define PTEST_DIR "/usr/lib/ptest"

int run_package(const char *pkg_path, const char *pkg_name) {
    char script[512];
    snprintf(script, sizeof(script), "%s/run-ptest", pkg_path);

    printf("START: %s\n", pkg_name);
    fflush(stdout);

    if (chdir(pkg_path) != 0) {
        perror("chdir");
        return -1;
    }

    int ret = system("./run-ptest");

    printf("END: %s\n", pkg_name);
    fflush(stdout);
    return ret;
}

int main(void) {
    DIR *dir = opendir(PTEST_DIR);
    struct dirent *entry;

    if (!dir) { perror("opendir"); return 1; }

    while ((entry = readdir(dir)) != NULL) {
        if (entry->d_type != DT_DIR) continue;
        if (entry->d_name[0] == '.') continue;

        char path[512];
        snprintf(path, sizeof(path), "%s/%s", PTEST_DIR, entry->d_name);
        run_package(path, entry->d_name);
    }

    closedir(dir);
    return 0;
}
```

---

## Writing ptest Harnesses — C (Plain)

The simplest possible ptest: a C program that prints TAP-compatible lines and a `run-ptest` wrapper.

### `tests/test_math.c`

```c
/*
 * test_math.c — minimal ptest harness in plain C
 * Outputs TAP (Test Anything Protocol) lines.
 */
#include <stdio.h>
#include <math.h>
#include <string.h>

/* ---- tiny TAP helpers -------------------------------------------- */
static int test_count = 0;
static int fail_count = 0;

#define PASS(name)  do { \
    test_count++;         \
    printf("PASS: %s\n", (name)); \
} while (0)

#define FAIL(name)  do { \
    test_count++;         \
    fail_count++;         \
    printf("FAIL: %s\n", (name)); \
} while (0)

#define CHECK(name, expr) ((expr) ? PASS(name) : FAIL(name))

/* ---- actual tests ------------------------------------------------- */
static void test_addition(void) {
    CHECK("addition_positive", (2 + 2) == 4);
    CHECK("addition_negative", (-3 + 3) == 0);
}

static void test_sqrt(void) {
    CHECK("sqrt_four",  fabs(sqrt(4.0) - 2.0) < 1e-9);
    CHECK("sqrt_nine",  fabs(sqrt(9.0) - 3.0) < 1e-9);
    CHECK("sqrt_zero",  fabs(sqrt(0.0) - 0.0) < 1e-9);
}

static void test_string_ops(void) {
    const char *hello = "hello";
    CHECK("strlen_hello",    strlen(hello) == 5);
    CHECK("strcmp_equal",    strcmp("abc", "abc") == 0);
    CHECK("strcmp_less",     strcmp("abc", "abd") < 0);
}

/* ---- main --------------------------------------------------------- */
int main(void) {
    test_addition();
    test_sqrt();
    test_string_ops();

    /* TAP plan line at the end is optional; ptest-runner accepts
       both TAP and simple PASS/FAIL lines. */
    printf("\n%d tests, %d failures\n", test_count, fail_count);
    return (fail_count > 0) ? 1 : 0;
}
```

### `tests/run-ptest` (shell entry point)

```sh
#!/bin/sh
# run-ptest — entry point called by ptest-runner
# Must be executable (chmod +x) and live beside the test binary.
set -e

TESTDIR="$(dirname "$(readlink -f "$0")")"

echo "Running libfoo plain-C tests ..."
"${TESTDIR}/test_math"
```

---

## Writing ptest Harnesses — C++ with Catch2

[Catch2](https://github.com/catchorg/Catch2) is a popular header-only (v2) / CMake-based (v3) C++ test framework. Buildroot ships `catch2` as a host or target package; enable `BR2_PACKAGE_CATCH2` (or `BR2_PACKAGE_CATCH2_HOST`).

### `tests/test_vector.cpp`

```cpp
/*
 * test_vector.cpp — ptest harness using Catch2 (v3 style)
 *
 * Build with:
 *   $(CXX) -std=c++17 test_vector.cpp -o test_vector \
 *           $(pkg-config --cflags --libs catch2-with-main)
 *
 * Catch2 produces TAP output with --reporter tap, which ptest-runner
 * understands natively.
 */
#include <catch2/catch_test_macros.hpp>
#include <vector>
#include <algorithm>
#include <stdexcept>

/* ------------------------------------------------------------------ */
TEST_CASE("Vector basics", "[vector]") {

    SECTION("default construction") {
        std::vector<int> v;
        REQUIRE(v.empty());
        REQUIRE(v.size() == 0);
    }

    SECTION("push_back and size") {
        std::vector<int> v;
        v.push_back(10);
        v.push_back(20);
        v.push_back(30);
        REQUIRE(v.size() == 3);
        REQUIRE(v[0] == 10);
        REQUIRE(v.back() == 30);
    }

    SECTION("reserve does not change size") {
        std::vector<int> v;
        v.reserve(100);
        REQUIRE(v.size() == 0);
        REQUIRE(v.capacity() >= 100);
    }
}

/* ------------------------------------------------------------------ */
TEST_CASE("Vector algorithms", "[vector][algorithm]") {

    std::vector<int> data = {5, 3, 1, 4, 2};

    SECTION("sort ascending") {
        std::sort(data.begin(), data.end());
        REQUIRE(data == std::vector<int>{1, 2, 3, 4, 5});
    }

    SECTION("find existing element") {
        auto it = std::find(data.begin(), data.end(), 3);
        REQUIRE(it != data.end());
        REQUIRE(*it == 3);
    }

    SECTION("find missing element") {
        auto it = std::find(data.begin(), data.end(), 99);
        REQUIRE(it == data.end());
    }

    SECTION("max_element") {
        auto it = std::max_element(data.begin(), data.end());
        REQUIRE(*it == 5);
    }
}

/* ------------------------------------------------------------------ */
TEST_CASE("Vector bounds checking", "[vector][safety]") {

    std::vector<int> v = {1, 2, 3};

    SECTION("at() throws on out-of-range") {
        REQUIRE_THROWS_AS(v.at(10), std::out_of_range);
    }

    SECTION("at() succeeds in range") {
        REQUIRE_NOTHROW(v.at(2));
        REQUIRE(v.at(2) == 3);
    }
}
```

### `tests/run-ptest` for Catch2

```sh
#!/bin/sh
# run-ptest — Catch2 variant
# Use --reporter tap so ptest-runner gets TAP lines (ok / not ok).
TESTDIR="$(dirname "$(readlink -f "$0")")"
exec "${TESTDIR}/test_vector" --reporter tap --order rand
```

Catch2 TAP output looks like:

```
TAP version 13
1..6
ok 1 - Vector basics/default construction
ok 2 - Vector basics/push_back and size
ok 3 - Vector basics/reserve does not change size
ok 4 - Vector algorithms/sort ascending
not ok 5 - Vector algorithms/find existing element
ok 6 - Vector bounds checking/at() throws on out-of-range
```

`ptest-runner` counts `ok` as PASS and `not ok` as FAIL.

### Catch2 package `.mk` snippet

```makefile
# Inside libfoo.mk — Catch2 test build
ifeq ($(BR2_PACKAGE_LIBFOO_INSTALL_TESTS),y)
LIBFOO_DEPENDENCIES += catch2

define LIBFOO_BUILD_TESTS
    $(TARGET_CXX) -std=c++17 \
        $(LIBFOO_CXXFLAGS) \
        $(@D)/tests/test_vector.cpp \
        -o $(@D)/tests/test_vector \
        $(shell $(PKG_CONFIG) --cflags --libs catch2-with-main)
endef
LIBFOO_POST_BUILD_HOOKS += LIBFOO_BUILD_TESTS

define LIBFOO_INSTALL_TESTS
    $(INSTALL) -d $(TARGET_DIR)/usr/lib/ptest/libfoo
    $(INSTALL) -m 0755 $(@D)/tests/run-ptest \
        $(TARGET_DIR)/usr/lib/ptest/libfoo/run-ptest
    $(INSTALL) -m 0755 $(@D)/tests/test_vector \
        $(TARGET_DIR)/usr/lib/ptest/libfoo/test_vector
endef
LIBFOO_POST_INSTALL_TARGET_HOOKS += LIBFOO_INSTALL_TESTS
endif
```

---

## Writing ptest Harnesses — Rust

Rust's built-in `#[test]` attribute produces a test runner binary when compiled with `cargo test --no-run`. Buildroot's `cargo-package` infrastructure handles cross-compilation; the test binary is then installed as the ptest harness.

### `src/lib.rs` — library with embedded unit tests

```rust
//! libbar — example Rust library with ptest-compatible unit tests.

/// Add two integers with overflow checking.
pub fn checked_add(a: i32, b: i32) -> Option<i32> {
    a.checked_add(b)
}

/// Reverse a string slice in O(n).
pub fn reverse_str(s: &str) -> String {
    s.chars().rev().collect()
}

/// Determine whether a number is prime (naïve trial division).
pub fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    let limit = (n as f64).sqrt() as u64 + 1;
    (3..=limit).step_by(2).all(|i| n % i != 0)
}

/* ================================================================== */
#[cfg(test)]
mod tests {
    use super::*;

    /* ------ checked_add ------------------------------------------- */
    #[test]
    fn test_checked_add_normal() {
        assert_eq!(checked_add(2, 3), Some(5));
    }

    #[test]
    fn test_checked_add_negative() {
        assert_eq!(checked_add(-10, 4), Some(-6));
    }

    #[test]
    fn test_checked_add_overflow() {
        // i32::MAX + 1 must return None, not panic
        assert_eq!(checked_add(i32::MAX, 1), None);
    }

    /* ------ reverse_str ------------------------------------------- */
    #[test]
    fn test_reverse_ascii() {
        assert_eq!(reverse_str("hello"), "olleh");
    }

    #[test]
    fn test_reverse_empty() {
        assert_eq!(reverse_str(""), "");
    }

    #[test]
    fn test_reverse_unicode() {
        // Reverses by char (Unicode scalar), not by byte
        assert_eq!(reverse_str("café"), "éfac");
    }

    /* ------ is_prime ---------------------------------------------- */
    #[test]
    fn test_prime_small_primes() {
        let primes = [2u64, 3, 5, 7, 11, 13, 17, 19, 23];
        for &p in &primes {
            assert!(is_prime(p), "{} should be prime", p);
        }
    }

    #[test]
    fn test_prime_composites() {
        let composites = [0u64, 1, 4, 6, 8, 9, 10, 15, 100];
        for &c in &composites {
            assert!(!is_prime(c), "{} should not be prime", c);
        }
    }

    #[test]
    fn test_prime_large() {
        assert!(is_prime(7_919));       // 1000th prime
        assert!(!is_prime(7_920));
    }
}
```

### `tests/integration_test.rs` — integration-level ptest

```rust
//! Integration tests for libbar — runs as a separate test binary.
//! `cargo test --test integration_test` builds `target/.../integration_test`.

use libbar::{checked_add, is_prime, reverse_str};

#[test]
fn integration_add_then_check_prime() {
    // 2 + 5 = 7, which is prime
    let result = checked_add(2, 5).expect("should not overflow");
    assert!(is_prime(result as u64));
}

#[test]
fn integration_reverse_is_involution() {
    let original = "Buildroot ptest";
    assert_eq!(reverse_str(&reverse_str(original)), original);
}
```

### `Cargo.toml`

```toml
[package]
name    = "libbar"
version = "0.1.0"
edition = "2021"

[lib]
name = "libbar"
crate-type = ["rlib", "cdylib"]   # cdylib for C FFI if needed

[[test]]
name = "integration_test"
path = "tests/integration_test.rs"

[profile.test]
# Keep debug info for better panic messages on the target
debug = true
opt-level = 1
```

### `run-ptest` for Rust

Rust test binaries accept `--format terse` or `--format json`. The simplest ptest-compatible output uses the default format parsed by a small shell wrapper that translates `test X ... ok` / `FAILED` lines:

```sh
#!/bin/sh
# run-ptest — Rust variant
# The Rust test runner outputs lines like:
#   test tests::test_prime_small_primes ... ok
#   test tests::test_checked_add_overflow ... FAILED
# We translate them to PASS:/FAIL: for ptest-runner.

TESTDIR="$(dirname "$(readlink -f "$0")")"

run_binary() {
    local bin="$1"
    [ -x "${TESTDIR}/${bin}" ] || return 0

    "${TESTDIR}/${bin}" --test-threads=1 2>&1 | while IFS= read -r line; do
        case "${line}" in
            test\ *\ ...\ ok)
                name=$(echo "${line}" | sed 's/^test //;s/ \.\.\. ok$//')
                echo "PASS: ${bin}::${name}"
                ;;
            test\ *\ ...\ FAILED)
                name=$(echo "${line}" | sed 's/^test //;s/ \.\.\. FAILED$//')
                echo "FAIL: ${bin}::${name}"
                ;;
            *)
                # Forward other output (panic messages, ignored tests) as-is
                echo "${line}"
                ;;
        esac
    done
}

run_binary "libbar_unit_tests"
run_binary "integration_test"
```

### Buildroot `.mk` for the Rust package

```makefile
################################################################################
# libbar — Rust library with ptest harness
################################################################################
LIBBAR_VERSION = 0.1.0
LIBBAR_SITE    = $(BR2_EXTERNAL_MYBOARD_PATH)/package/libbar
LIBBAR_SITE_METHOD = local

LIBBAR_LICENSE = MIT

# Tell the cargo-package infra to also build test binaries
ifeq ($(BR2_PACKAGE_LIBBAR_INSTALL_TESTS),y)
LIBBAR_CARGO_BUILD_OPTS += --tests
endif

$(eval $(cargo-package))

# ---- Install test binaries and run-ptest script ----------------------------
ifeq ($(BR2_PACKAGE_LIBBAR_INSTALL_TESTS),y)
define LIBBAR_INSTALL_TESTS
    $(INSTALL) -d $(TARGET_DIR)/usr/lib/ptest/libbar

    # Install unit-test binary (name derived from package + hash by cargo)
    $(INSTALL) -m 0755 \
        $(wildcard $(@D)/target/$(RUSTC_TARGET_NAME)/debug/deps/libbar_unit_tests-*[^.d]) \
        $(TARGET_DIR)/usr/lib/ptest/libbar/libbar_unit_tests

    # Install integration-test binary
    $(INSTALL) -m 0755 \
        $(wildcard $(@D)/target/$(RUSTC_TARGET_NAME)/debug/deps/integration_test-*[^.d]) \
        $(TARGET_DIR)/usr/lib/ptest/libbar/integration_test

    $(INSTALL) -m 0755 $(LIBBAR_PKGDIR)/run-ptest \
        $(TARGET_DIR)/usr/lib/ptest/libbar/run-ptest
endef
LIBBAR_POST_INSTALL_TARGET_HOOKS += LIBBAR_INSTALL_TESTS
endif
```

---

## Integrating ptest into a Buildroot Package

### Complete `Config.in` example

```kconfig
config BR2_PACKAGE_LIBFOO
    bool "libfoo"
    help
      The libfoo library provides foo functionality.
      https://example.com/libfoo

if BR2_PACKAGE_LIBFOO

config BR2_PACKAGE_LIBFOO_INSTALL_TESTS
    bool "Install libfoo tests"
    depends on BR2_PACKAGE_PTEST_RUNNER
    select BR2_PACKAGE_CATCH2
    help
      Install the libfoo ptest harness under /usr/lib/ptest/libfoo.
      Tests use Catch2 and can be executed with ptest-runner.

endif
```

### Complete `libfoo.mk`

```makefile
################################################################################
# libfoo
################################################################################
LIBFOO_VERSION        = 1.2.3
LIBFOO_SITE           = https://example.com/libfoo-$(LIBFOO_VERSION).tar.gz
LIBFOO_INSTALL_STAGING = YES
LIBFOO_LICENSE        = LGPL-2.1+
LIBFOO_LICENSE_FILES   = COPYING.LESSER

LIBFOO_CONF_OPTS = \
    -DBUILD_SHARED_LIBS=ON

ifeq ($(BR2_PACKAGE_LIBFOO_INSTALL_TESTS),y)
LIBFOO_CONF_OPTS    += -DBUILD_TESTS=ON
LIBFOO_DEPENDENCIES += catch2
endif

$(eval $(cmake-package))

# ---- ptest installation ------------------------------------------------
ifeq ($(BR2_PACKAGE_LIBFOO_INSTALL_TESTS),y)
define LIBFOO_INSTALL_PTEST
    $(INSTALL) -d $(TARGET_DIR)/usr/lib/ptest/libfoo
    $(INSTALL) -m 0755 $(LIBFOO_PKGDIR)/run-ptest \
        $(TARGET_DIR)/usr/lib/ptest/libfoo/run-ptest
    # CMake places test binaries under build/tests/
    $(INSTALL) -m 0755 $(@D)/tests/test_math \
        $(TARGET_DIR)/usr/lib/ptest/libfoo/test_math
    $(INSTALL) -m 0755 $(@D)/tests/test_vector \
        $(TARGET_DIR)/usr/lib/ptest/libfoo/test_vector
    # Copy any fixture files
    cp -r $(LIBFOO_PKGDIR)/test_data \
        $(TARGET_DIR)/usr/lib/ptest/libfoo/
endef
LIBFOO_POST_INSTALL_TARGET_HOOKS += LIBFOO_INSTALL_PTEST
endif
```

---

## Running ptest on the Target

### Target filesystem layout (ASCII art)

```
/usr/lib/ptest/
├── libfoo/
│   ├── run-ptest          <-- entry point (chmod 755)
│   ├── test_math          <-- plain-C binary
│   ├── test_vector        <-- Catch2 C++ binary
│   └── test_data/
│       └── sample.txt
│
├── libbar/
│   ├── run-ptest          <-- entry point (chmod 755)
│   ├── libbar_unit_tests  <-- Rust unit-test binary
│   └── integration_test   <-- Rust integration-test binary
│
└── python3-requests/
    ├── run-ptest
    └── tests/
        └── test_requests.py
```

### Execution flow (ASCII art)

```
ptest-runner
    │
    ├─── [scan] /usr/lib/ptest/libfoo/run-ptest   found
    │        │
    │        └── chdir /usr/lib/ptest/libfoo/
    │            exec ./run-ptest
    │              │
    │              ├─ ./test_math          stdout → PASS/FAIL lines
    │              └─ ./test_vector --reporter tap
    │
    ├─── [scan] /usr/lib/ptest/libbar/run-ptest   found
    │        │
    │        └── chdir /usr/lib/ptest/libbar/
    │            exec ./run-ptest
    │              │
    │              ├─ ./libbar_unit_tests --test-threads=1
    │              └─ ./integration_test  --test-threads=1
    │
    └─── [scan] /usr/lib/ptest/python3-requests/  found
             └── exec ./run-ptest
                   └─ python3 -m pytest tests/ -v
```

### Example terminal session

```
# ptest-runner -p libfoo -t 30
START: libfoo
Running libfoo plain-C tests ...
PASS: addition_positive
PASS: addition_negative
PASS: sqrt_four
PASS: sqrt_nine
PASS: sqrt_zero
PASS: strlen_hello
PASS: strcmp_equal
PASS: strcmp_less
8 tests, 0 failures
TAP version 13
1..6
ok 1 - Vector basics/default construction
ok 2 - Vector basics/push_back and size
ok 3 - Vector basics/reserve does not change size
ok 4 - Vector algorithms/sort ascending
ok 5 - Vector algorithms/find existing element
ok 6 - Vector bounds checking/at() throws on out-of-range
END: libfoo

# ptest-runner -p libbar -t 60
START: libbar
PASS: libbar_unit_tests::tests::test_checked_add_normal
PASS: libbar_unit_tests::tests::test_checked_add_negative
PASS: libbar_unit_tests::tests::test_checked_add_overflow
PASS: libbar_unit_tests::tests::test_reverse_ascii
PASS: libbar_unit_tests::tests::test_reverse_empty
PASS: libbar_unit_tests::tests::test_reverse_unicode
PASS: libbar_unit_tests::tests::test_prime_small_primes
PASS: libbar_unit_tests::tests::test_prime_composites
PASS: libbar_unit_tests::tests::test_prime_large
PASS: integration_test::integration_add_then_check_prime
PASS: integration_test::integration_reverse_is_involution
END: libbar
```

---

## Output Format and Interpretation

`ptest-runner` wraps each package with `START:` / `END:` banners. The lines in between can be:

| Line pattern           | Meaning                                      |
|------------------------|----------------------------------------------|
| `PASS: <test-name>`    | Test passed (plain ptest format)             |
| `FAIL: <test-name>`    | Test failed (plain ptest format)             |
| `ok N - <description>` | TAP pass (Catch2 / pytest-tap)               |
| `not ok N - <desc>`    | TAP fail                                     |
| `# ...`                | TAP comment / diagnostic                     |
| `SKIP: <test-name>`    | Test skipped (optional, not counted as fail) |
| Any other line         | Forwarded verbatim (build info, panics, etc.)|

### Parsing results in CI (Python snippet)

```python
#!/usr/bin/env python3
"""Parse ptest-runner output and exit non-zero if any test failed."""

import re, sys

PASS_RE = re.compile(r'^(?:PASS:|ok \d+ -)')
FAIL_RE = re.compile(r'^(?:FAIL:|not ok \d+ -)')
START_RE = re.compile(r'^START:\s+(\S+)')
END_RE   = re.compile(r'^END:\s+(\S+)')

def parse_ptest_output(path: str) -> dict:
    results = {}
    current = None

    with open(path) as fh:
        for line in fh:
            line = line.rstrip()
            if m := START_RE.match(line):
                current = m.group(1)
                results[current] = {"pass": 0, "fail": 0}
            elif m := END_RE.match(line):
                current = None
            elif current:
                if PASS_RE.match(line):
                    results[current]["pass"] += 1
                elif FAIL_RE.match(line):
                    results[current]["fail"] += 1
    return results

if __name__ == "__main__":
    results = parse_ptest_output(sys.argv[1])
    total_fail = 0
    for pkg, counts in results.items():
        status = "OK" if counts["fail"] == 0 else "FAILED"
        print(f"  [{status}] {pkg}: "
              f"{counts['pass']} passed, {counts['fail']} failed")
        total_fail += counts["fail"]
    sys.exit(1 if total_fail else 0)
```

---

## Troubleshooting

### ptest directory is empty after build

```
find output/target/usr/lib/ptest -maxdepth 1 -type d
# Shows only the top directory — no packages installed
```

Checklist:

```
1. Is BR2_PACKAGE_PTEST_RUNNER=y set?
   → make menuconfig → Build options → Enable ptest framework support

2. Is BR2_PACKAGE_<PKG>_INSTALL_TESTS=y set?
   → make menuconfig → Target packages → <pkg> → Install tests

3. Did you rebuild after changing config?
   → make <pkgname>-rebuild
```

### `run-ptest` not executable

```bash
# On host, check:
ls -la output/target/usr/lib/ptest/libfoo/run-ptest
# -rw-r--r-- ← BAD: must be -rwxr-xr-x

# Fix in .mk:
$(INSTALL) -m 0755 ...  # not 0644
```

### Rust test binary not found (wildcard expansion fails)

Cargo embeds a hash in test binary names, e.g. `libbar_unit_tests-8f3a1c`. Use `find` instead of a bare wildcard:

```makefile
define LIBBAR_INSTALL_TESTS
    $(INSTALL) -d $(TARGET_DIR)/usr/lib/ptest/libbar
    UNIT_BIN=$$(find $(@D)/target/$(RUSTC_TARGET_NAME)/debug/deps \
        -name "libbar_unit_tests-*" ! -name "*.d" -type f | head -1); \
    $(INSTALL) -m 0755 "$${UNIT_BIN}" \
        $(TARGET_DIR)/usr/lib/ptest/libbar/libbar_unit_tests
endef
```

### Catch2 not found via pkg-config on the target

Catch2 is a *build-time* / *host* dependency for linking. Ensure:

```makefile
LIBFOO_DEPENDENCIES += host-catch2
# and pass its include path explicitly:
LIBFOO_CONF_OPTS += -DCatch2_DIR=$(HOST_DIR)/lib/cmake/Catch2
```

---

## Summary

```
+---------------------------------------------------------------+
|              ptest in Buildroot — At a Glance                 |
+---------------------------------------------------------------+
|                                                               |
|  CONFIG                                                       |
|  ──────                                                       |
|  BR2_PACKAGE_PTEST_RUNNER=y         global enable             |
|  BR2_PACKAGE_<PKG>_INSTALL_TESTS=y  per-package opt-in        |
|                                                               |
|  INSTALLED LAYOUT (target)                                    |
|  ─────────────────────────                                    |
|  /usr/lib/ptest/<pkg>/                                        |
|      run-ptest    ← shell entry point (MUST be chmod 755)     |
|      <test-bins>  ← C, C++, or Rust test executables          |
|      <fixtures>/  ← data files, if any                        |
|                                                               |
|  RUNNER                                                       |
|  ──────                                                       |
|  ptest-runner          run all ptests                         |
|  ptest-runner -p <pkg> run one package                        |
|  ptest-runner -t <sec> per-test timeout                       |
|                                                               |
|  OUTPUT FORMAT                                                |
|  ─────────────                                                |
|  START: <pkg>                                                 |
|  PASS: <test>  /  ok N - <test>    (pass)                     |
|  FAIL: <test>  /  not ok N - <test>(fail)                     |
|  END: <pkg>                                                   |
|                                                               |
|  LANGUAGE INTEGRATION                                         |
|  ────────────────────                                         |
|  C        → hand-rolled PASS/FAIL macros, TAP output          |
|  C++      → Catch2 --reporter tap → native TAP                |
|  Rust     → cargo test --no-run; shell translates             |
|             "ok" / "FAILED" lines to PASS:/FAIL:              |
|                                                               |
|  MK HOOKS                                                     |
|  ────────                                                     |
|  LIBFOO_POST_BUILD_HOOKS   += LIBFOO_BUILD_TESTS              |
|  LIBFOO_POST_INSTALL_TARGET_HOOKS += LIBFOO_INSTALL_PTEST     |
|                                                               |
|  CI TIP                                                       |
|  ──────                                                       |
|  Boot image in QEMU → ptest-runner > results.txt →            |
|  parse with Python script → exit 1 on any FAIL                |
+---------------------------------------------------------------+
```

### Key takeaways

**ptest-runner** provides a single, uniform entry point for all package-level runtime tests on a Buildroot target. Each package installs a `run-ptest` shell script alongside its test binaries under `/usr/lib/ptest/<pkgname>/`. The runner calls every `run-ptest` it finds, collects `PASS:`/`FAIL:` (or TAP `ok`/`not ok`) lines, and wraps them with `START:` / `END:` banners.

For **C** packages the simplest approach is a hand-rolled TAP printer — no additional dependencies required. For **C++** packages, **Catch2** with `--reporter tap` produces standards-conformant TAP with zero boilerplate. For **Rust** packages, `cargo test --no-run` produces a test binary whose default output is easily translated to `PASS:`/`FAIL:` by a shell wrapper.

The Buildroot integration lives entirely in the package `.mk` file through `POST_BUILD_HOOKS` (to compile tests) and `POST_INSTALL_TARGET_HOOKS` (to copy binaries and the `run-ptest` script). The `Config.in` entry gates the feature behind `depends on BR2_PACKAGE_PTEST_RUNNER` so it is only visible and selectable when the framework is globally enabled.

Incorporating ptest into a Buildroot-based project enables automated regression testing at the *hardware* or *QEMU* level, catching architecture-specific bugs (endianness, alignment, ABI mismatch, missing kernel features) that host-side unit tests can never expose.