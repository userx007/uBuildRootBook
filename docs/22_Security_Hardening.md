# 22. Security Hardening in Buildroot

> **Topic:** Compiler hardening flags (`-fstack-protector-strong`, `-D_FORTIFY_SOURCE=2`, `-fPIE`),
> stripping SUID binaries, removing dev tools, and `BR2_SSP_*` / `BR2_RELRO_*` options.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Threat Model](#the-threat-model)
3. [Buildroot Security Configuration Options](#buildroot-security-configuration-options)
4. [Compiler Hardening Flags](#compiler-hardening-flags)
   - [Stack Protector](#stack-protector)
   - [FORTIFY_SOURCE](#fortify_source)
   - [Position Independent Executables (PIE)](#position-independent-executables-pie)
   - [RELRO](#relro)
5. [SUID Binary Hardening](#suid-binary-hardening)
6. [Removing Development Tools](#removing-development-tools)
7. [C Code Examples](#c-code-examples)
8. [C++ Code Examples](#c-code-examples-1)
9. [Rust Code Examples](#rust-code-examples)
10. [Security Layers Diagram (ASCII)](#security-layers-diagram-ascii)
11. [Attack Surface Reduction Diagram (ASCII)](#attack-surface-reduction-diagram-ascii)
12. [Buildroot menuconfig Reference](#buildroot-menuconfig-reference)
13. [Summary](#summary)

---

## Introduction

Security hardening in an embedded Linux system built with Buildroot is not a single feature but
a layered strategy. Unlike desktop Linux distributions that can apply patches and updates
continuously, embedded systems are often deployed for years in the field with fixed firmware.
This makes it critical to get security right at build time.

Buildroot provides first-class support for a range of compile-time and link-time hardening
techniques. These mechanisms raise the cost of exploitation by making common vulnerability
classes — buffer overflows, format string bugs, use-after-free — significantly harder to turn
into working exploits. The options are exposed both through `menuconfig` (interactive) and
directly in `BR2_*` Kconfig symbols that can be set in a `defconfig` file for reproducible,
auditable builds.

---

## The Threat Model

Before examining individual flags, it is worth understanding what they protect against:

```
  ATTACKER CAPABILITIES          WHAT HARDENING DEFENDS
  ─────────────────────          ──────────────────────────────────────────────
  Memory corruption bug    ──►   Stack canary detects overflow before return
  (buffer overflow)              FORTIFY_SOURCE checks bounds at libc level

  Code injection           ──►   NX/XD bit (no-exec stack)
                                 PIE + ASLR randomises load address

  ROP / ret2libc           ──►   Full RELRO makes GOT read-only after load
                                 PIE breaks gadget addresses

  Privilege escalation     ──►   No SUID binaries except minimal required set
  via SUID binary                Capabilities instead of blanket SUID

  Info leakage via         ──►   Debug symbols stripped from final image
  debug symbols                  /proc/config.gz disabled

  Post-exploit tooling     ──►   No compiler, linker, make in target rootfs
                                 No package manager, no wget/curl in minimal config
```

---

## Buildroot Security Configuration Options

Buildroot surfaces security hardening through `BR2_SSP_*` and `BR2_RELRO_*` Kconfig symbols.
These map directly to compiler and linker flags applied globally across the entire package build.

### BR2_SSP_* — Stack Smashing Protector

| Symbol                        | GCC Flag                    | Protection Level |
|-------------------------------|-----------------------------|------------------|
| `BR2_SSP_NONE`                | (no flag)                   | None             |
| `BR2_SSP_REGULAR`             | `-fstack-protector`         | Functions with 8-byte buffers |
| `BR2_SSP_STRONG`              | `-fstack-protector-strong`  | More functions, all arrays |
| `BR2_SSP_ALL`                 | `-fstack-protector-all`     | Every function (highest cost) |

**Recommended:** `BR2_SSP_STRONG` — good security/performance balance.

In your `defconfig`:

```
BR2_SSP_STRONG=y
```

### BR2_RELRO_* — RELocation Read-Only

| Symbol               | Linker Flag            | Effect |
|----------------------|------------------------|--------|
| `BR2_RELRO_NONE`     | (none)                 | GOT is writable at all times |
| `BR2_RELRO_PARTIAL`  | `-Wl,-z,relro`         | Partial GOT protection |
| `BR2_RELRO_FULL`     | `-Wl,-z,relro,-z,now`  | Full GOT locked before main() |

**Recommended:** `BR2_RELRO_FULL` for maximum protection; it forces eager binding (all symbols
resolved at load time), which slightly increases startup latency but removes the writable GOT
attack window entirely.

In your `defconfig`:

```
BR2_RELRO_FULL=y
```

---

## Compiler Hardening Flags

### Stack Protector

The stack protector inserts a secret canary value between local variables and the saved return
address. Before a function returns, the canary is checked; if it has been modified (overwritten
by a buffer overflow), the program is immediately terminated with `__stack_chk_fail()`.

```
  Stack frame layout WITH -fstack-protector-strong:

  High address
  ┌──────────────────────┐
  │  Caller's frame      │
  ├──────────────────────┤
  │  Return address      │  ◄─── attacker wants to overwrite this
  ├──────────────────────┤
  │  Saved RBP           │
  ├──────────────────────┤
  │  CANARY VALUE        │  ◄─── random secret placed here at function entry
  ├──────────────────────┤       checked before return; abort if changed
  │  Local arrays/vars   │
  ├──────────────────────┤
  │  char buf[64]        │  ◄─── overflow starts here, must pass canary to reach ret addr
  └──────────────────────┘
  Low address
```

### FORTIFY_SOURCE

`-D_FORTIFY_SOURCE=2` enables compile-time and run-time bounds checking in glibc/musl for
functions like `memcpy`, `strcpy`, `sprintf`, `gets`, `read`, etc. When the compiler can
statically determine buffer sizes, calls with provably-unsafe lengths are rejected at
compile time. For dynamic cases, run-time checks abort the program.

Level 1 (`=1`) checks only at run time. Level 2 (`=2`) additionally adds compile-time checks
and is stricter about inline vs. non-inline versions of functions.

**Note:** This flag requires optimisation (`-O1` or higher) to be effective because the compiler
needs to evaluate `__builtin_object_size()`.

### Position Independent Executables (PIE)

PIE, combined with the kernel's ASLR (Address Space Layout Randomisation), randomises the base
address of the executable at every run. This breaks hardcoded address assumptions in shellcode
and makes ROP-chain construction much harder.

```
  WITHOUT PIE (ASLR only applies to libs/heap/stack):
  ┌──────────────────────────────────────────────────┐
  │ 0x400000  [FIXED]  .text of executable           │
  │ ...                                              │
  │ 0x7f????  [RAND]   libc.so                       │
  │ 0x7f????  [RAND]   heap                          │
  │ 0x7f????  [RAND]   stack                         │
  └──────────────────────────────────────────────────┘
  Attacker can ROP into fixed executable addresses!

  WITH PIE + ASLR:
  ┌──────────────────────────────────────────────────┐
  │ 0x5?????? [RAND]   .text of executable  (PIE)    │
  │ 0x7f????  [RAND]   libc.so              (ASLR)   │
  │ 0x7f????  [RAND]   heap                 (ASLR)   │
  │ 0x7f????  [RAND]   stack                (ASLR)   │
  └──────────────────────────────────────────────────┘
  All addresses randomised on every execution!
```

In Buildroot, PIE is enabled via:

```
BR2_PIE=y
```

This adds `-fPIE` to `CFLAGS`/`CXXFLAGS` and `-pie` to `LDFLAGS` for all packages.

### RELRO

RELRO (RELocation Read-Only) hardens the Global Offset Table (GOT) — the structure the dynamic
linker uses for lazy symbol resolution. Without RELRO, the GOT is writable and a common target
for `write-what-where` exploits (e.g., overwriting a GOT entry to redirect a function call).

- **Partial RELRO:** The non-PLT GOT sections are marked read-only after loading.
- **Full RELRO:** All GOT sections are resolved eagerly at load time, then the entire GOT is
  marked read-only. This eliminates lazy binding.

---

## SUID Binary Hardening

SUID (Set-User-ID) binaries execute with elevated privileges regardless of who runs them.
A vulnerability in any SUID binary is immediately a privilege escalation vector.

### Finding SUID Binaries in the Target Rootfs

```bash
# Run this on the built target filesystem to audit SUID binaries
find output/target -perm /4000 -type f 2>/dev/null
```

### Stripping Unnecessary SUID Bits

In a post-build script (`board/myboard/post-build.sh`):

```bash
#!/bin/bash
# Strip SUID from binaries that don't need it
TARGET="$1"

# List of binaries allowed to keep SUID (minimal set)
ALLOWED_SUID="/bin/su /usr/bin/passwd"

find "${TARGET}" -perm /4000 -type f | while read -r binary; do
    allowed=0
    for permitted in ${ALLOWED_SUID}; do
        if [ "${binary}" = "${TARGET}${permitted}" ]; then
            allowed=1
            break
        fi
    done
    if [ "${allowed}" -eq 0 ]; then
        echo "[HARDENING] Removing SUID from: ${binary}"
        chmod u-s "${binary}"
    fi
done
```

Register the script in your `defconfig`:

```
BR2_ROOTFS_POST_BUILD_SCRIPT="board/myboard/post-build.sh"
```

### Using Linux Capabilities Instead of SUID

Modern kernels support fine-grained capabilities (e.g., `CAP_NET_RAW` for ping). Using
capabilities avoids blanket root elevation:

```bash
# In your post-build script — set capability instead of SUID
# Requires 'libcap' package in Buildroot
setcap cap_net_raw+ep "${TARGET}/bin/ping"
chmod u-s "${TARGET}/bin/ping"
```

---

## Removing Development Tools

A minimal target rootfs dramatically reduces the attack surface. Development tools — compilers,
linkers, make, package managers — have no place on a production embedded system.

### Buildroot Minimal Configuration Principles

```
  MINIMAL ROOTFS PRINCIPLE:

  ┌─────────────────────────────────────────────────────┐
  │              HOST (build machine)                   │
  │  ┌──────────┐  ┌──────────┐  ┌────────────────┐    │
  │  │   gcc    │  │   make   │  │ binutils/strip │    │
  │  └──────────┘  └──────────┘  └────────────────┘    │
  │        │               │             │              │
  │        └───────────────┴─────────────┘              │
  │                        │                            │
  │                  Build happens HERE                 │
  └────────────────────────┼────────────────────────────┘
                           │ cross-compile
                           ▼
  ┌─────────────────────────────────────────────────────┐
  │              TARGET (embedded device)               │
  │  ✓ Your application binary (stripped)               │
  │  ✓ Required shared libs (stripped)                  │
  │  ✓ busybox (minimal shell for maintenance)          │
  │  ✗ NO gcc, g++, ld, as, ar                          │
  │  ✗ NO make, cmake, autoconf                         │
  │  ✗ NO python, perl, ruby                            │
  │  ✗ NO package manager                               │
  │  ✗ NO debug symbols in any binary                   │
  └─────────────────────────────────────────────────────┘
```

### Strip Debug Symbols

In your `defconfig`:

```
BR2_STRIP_strip=y
BR2_STRIP_EXCLUDE_FILES=""
BR2_STRIP_EXCLUDE_DIRS=""
```

This causes Buildroot to run `$(STRIP)` (the cross-strip tool) on all installed binaries and
libraries, removing `.debug_*` sections that would otherwise leak symbol names, source file
paths, and line numbers to an attacker.

---

## C Code Examples

### Example 1: Stack Buffer Overflow — Vulnerable vs Hardened

```c
/* vulnerable.c — compiled WITHOUT hardening */
#include <stdio.h>
#include <string.h>

void process_input(const char *user_input) {
    char buffer[64];
    /*
     * strcpy does NOT check bounds.
     * Input > 64 bytes overwrites the canary, saved RBP,
     * and return address on the stack.
     */
    strcpy(buffer, user_input);  /* DANGEROUS */
    printf("Processed: %s\n", buffer);
}

int main(int argc, char *argv[]) {
    if (argc < 2) return 1;
    process_input(argv[1]);
    return 0;
}
```

```c
/* hardened.c — written defensively, compiled WITH all flags */
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define MAX_INPUT 63

void process_input(const char *user_input) {
    char buffer[MAX_INPUT + 1];  /* +1 for NUL terminator */

    /*
     * strncpy + explicit NUL termination.
     * With -D_FORTIFY_SOURCE=2, the compiler substitutes
     * __strncpy_chk() which verifies sizeof(buffer) at runtime.
     */
    strncpy(buffer, user_input, MAX_INPUT);
    buffer[MAX_INPUT] = '\0';    /* always NUL-terminate */

    printf("Processed: %s\n", buffer);
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <input>\n", argv[0]);
        return EXIT_FAILURE;
    }

    if (strlen(argv[1]) > MAX_INPUT) {
        fprintf(stderr, "Error: input too long (max %d chars)\n", MAX_INPUT);
        return EXIT_FAILURE;
    }

    process_input(argv[1]);
    return EXIT_SUCCESS;
}
```

**Build commands:**

```bash
# Vulnerable build (no hardening)
gcc -O0 -o vulnerable vulnerable.c

# Hardened build (all flags)
gcc -O2                             \
    -fstack-protector-strong        \
    -D_FORTIFY_SOURCE=2             \
    -fPIE -pie                      \
    -Wl,-z,relro,-z,now             \
    -Wall -Wextra                   \
    -o hardened hardened.c
```

### Example 2: Checking Hardening Flags on a Built Binary

```c
/* check_hardening.c
 * A utility to inspect ELF binaries for hardening properties.
 * Compile: gcc -O2 -o check_hardening check_hardening.c
 * Usage:   ./check_hardening /path/to/binary
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <elf.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

typedef struct {
    int is_pie;
    int has_stack_canary;
    int has_relro;
    int has_now;
    int has_nx;
    int is_stripped;
} HardeningInfo;

static void check_binary(const char *path, HardeningInfo *info) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) { perror("open"); return; }

    struct stat st;
    fstat(fd, &st);

    void *map = mmap(NULL, (size_t)st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    close(fd);
    if (map == MAP_FAILED) { perror("mmap"); return; }

    const Elf64_Ehdr *ehdr = (const Elf64_Ehdr *)map;

    /* Check PIE: ET_DYN with no interpreter = library; with interpreter = PIE exe */
    info->is_pie = (ehdr->e_type == ET_DYN);

    /* Walk program headers */
    const Elf64_Phdr *phdr = (const Elf64_Phdr *)((char *)map + ehdr->e_phoff);
    for (int i = 0; i < ehdr->e_phnum; i++) {
        if (phdr[i].p_type == PT_GNU_RELRO) {
            info->has_relro = 1;
        }
        if (phdr[i].p_type == PT_GNU_STACK) {
            /* NX: stack should NOT be executable (PF_X not set) */
            info->has_nx = !(phdr[i].p_flags & PF_X);
        }
        if (phdr[i].p_type == PT_DYNAMIC) {
            const Elf64_Dyn *dyn = (const Elf64_Dyn *)((char *)map + phdr[i].p_offset);
            for (; dyn->d_tag != DT_NULL; dyn++) {
                if (dyn->d_tag == DT_FLAGS && (dyn->d_un.d_val & DF_BIND_NOW))
                    info->has_now = 1;
                if (dyn->d_tag == DT_FLAGS_1 && (dyn->d_un.d_val & DF_1_NOW))
                    info->has_now = 1;
            }
        }
    }

    /* Check for symbol table (stripped if absent) */
    const Elf64_Shdr *shdr = (const Elf64_Shdr *)((char *)map + ehdr->e_shoff);
    info->is_stripped = 1;  /* assume stripped */
    for (int i = 0; i < ehdr->e_shnum; i++) {
        if (shdr[i].sh_type == SHT_SYMTAB) {
            info->is_stripped = 0;
            break;
        }
    }

    /* Stack canary: look for __stack_chk_fail in .dynstr */
    for (int i = 0; i < ehdr->e_shnum; i++) {
        if (shdr[i].sh_type == SHT_STRTAB && i != ehdr->e_shstrndx) {
            const char *strtab = (const char *)map + shdr[i].sh_offset;
            const char *end = strtab + shdr[i].sh_size;
            for (const char *p = strtab; p < end; ) {
                if (strstr(p, "__stack_chk_fail")) {
                    info->has_stack_canary = 1;
                    break;
                }
                p += strlen(p) + 1;
            }
        }
    }

    munmap(map, (size_t)st.st_size);
}

static void print_status(const char *name, int ok) {
    printf("  %-22s  %s\n", name, ok ? "[PASS]" : "[FAIL]");
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <elf-binary>\n", argv[0]);
        return EXIT_FAILURE;
    }

    HardeningInfo info = {0};
    check_binary(argv[1], &info);

    printf("\nHardening report: %s\n", argv[1]);
    printf("  %-22s  %-6s\n", "Property", "Result");
    printf("  %-22s  %-6s\n", "--------", "------");
    print_status("PIE",           info.is_pie);
    print_status("Stack canary",  info.has_stack_canary);
    print_status("RELRO",         info.has_relro);
    print_status("BIND_NOW",      info.has_now);
    print_status("NX stack",      info.has_nx);
    print_status("Stripped",      info.is_stripped);

    int score = info.is_pie + info.has_stack_canary + info.has_relro
              + info.has_now + info.has_nx + info.is_stripped;
    printf("\n  Score: %d/6\n\n", score);

    return (score == 6) ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

### Example 3: FORTIFY_SOURCE in Action

```c
/* fortify_demo.c
 * Demonstrates which calls FORTIFY_SOURCE can detect.
 * Compile: gcc -O2 -D_FORTIFY_SOURCE=2 -o fortify_demo fortify_demo.c
 */
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* This function will be caught by FORTIFY at compile time
 * if compiler can see the buffer size */
void demo_compile_time_check(void) {
    char small[4];
    /* With FORTIFY=2 and optimisation, the compiler sees that
     * sizeof(small)==4 < 8, so it generates a compile-time error
     * or replaces with the checked variant. */
    memcpy(small, "12345678", 8);  /* overflow — caught at compile time */
    printf("%s\n", small);
}

/* This is caught at run time because size is dynamic */
void demo_runtime_check(size_t n) {
    char buf[16];
    char *src = malloc(n);
    if (!src) return;
    memset(src, 'A', n);
    /*
     * At run time, __memcpy_chk() verifies n <= sizeof(buf).
     * If n > 16, it calls __chk_fail() which aborts with:
     *   *** buffer overflow detected ***
     */
    memcpy(buf, src, n);
    free(src);
    printf("Copied %zu bytes: %.16s\n", n, buf);
}

int main(void) {
    demo_runtime_check(8);   /* OK */
    demo_runtime_check(32);  /* Triggers __chk_fail at runtime */
    return 0;
}
```

---

## C++ Code Examples

### Example 4: RAII Wrapper Enforcing Safe Buffer Handling

```cpp
// safe_buffer.hpp
// A compile-time-sized buffer with bounds checking, compatible with hardening flags.
// Compile: g++ -O2 -fstack-protector-strong -D_FORTIFY_SOURCE=2 -fPIE -pie
//              -Wl,-z,relro,-z,now -std=c++17 -o safe_buf_demo safe_buffer_demo.cpp

#pragma once
#include <array>
#include <cstring>
#include <stdexcept>
#include <string_view>

template<std::size_t N>
class SafeBuffer {
    static_assert(N > 0, "Buffer size must be positive");
public:
    SafeBuffer() noexcept { data_.fill(0); }

    /**
     * Copy from string_view with strict bounds enforcement.
     * Throws std::length_error if input doesn't fit (N-1 usable bytes + NUL).
     */
    void assign(std::string_view sv) {
        if (sv.size() >= N) {
            throw std::length_error(
                "Input length " + std::to_string(sv.size()) +
                " exceeds buffer capacity " + std::to_string(N - 1));
        }
        std::memcpy(data_.data(), sv.data(), sv.size());
        data_[sv.size()] = '\0';
        used_ = sv.size();
    }

    [[nodiscard]] const char* c_str() const noexcept { return data_.data(); }
    [[nodiscard]] std::size_t size()  const noexcept { return used_; }
    [[nodiscard]] std::size_t capacity() const noexcept { return N - 1; }

    /* Bounds-checked element access */
    [[nodiscard]] char at(std::size_t i) const {
        if (i >= used_) throw std::out_of_range("SafeBuffer::at");
        return data_[i];
    }

    void clear() noexcept {
        data_.fill(0);
        used_ = 0;
    }

    /* Prevent implicit conversion to raw pointer */
    operator const char*() const = delete;

private:
    std::array<char, N> data_;
    std::size_t used_{0};
};
```

```cpp
// safe_buffer_demo.cpp
#include "safe_buffer.hpp"
#include <iostream>
#include <cassert>

int main() {
    SafeBuffer<64> buf;

    /* Normal usage */
    buf.assign("Hello, Buildroot hardening!");
    std::cout << "Content : " << buf.c_str()    << "\n";
    std::cout << "Length  : " << buf.size()      << "\n";
    std::cout << "Capacity: " << buf.capacity()  << "\n";

    /* Bounds-checked access */
    try {
        std::cout << "buf[0]  : " << buf.at(0) << "\n";
        std::cout << "buf[999]: " << buf.at(999) << "\n";  /* throws */
    } catch (const std::out_of_range& e) {
        std::cerr << "Caught expected: " << e.what() << "\n";
    }

    /* Overflow attempt */
    try {
        std::string huge(128, 'X');
        buf.assign(huge);  /* throws length_error */
    } catch (const std::length_error& e) {
        std::cerr << "Caught overflow: " << e.what() << "\n";
    }

    /* Verify buffer is still sane after the throws */
    assert(buf.size() == 27);

    return 0;
}
```

### Example 5: Buildroot Package CMakeLists.txt with Hardening

```cmake
# CMakeLists.txt for a Buildroot-built C++ package with full hardening
cmake_minimum_required(VERSION 3.16)
project(my_hardened_app CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ── Compiler hardening flags ──────────────────────────────────────────────────
# Buildroot sets these globally via BR2_SSP_STRONG / BR2_RELRO_FULL / BR2_PIE,
# but we also set them here for standalone builds and CI.

add_compile_options(
    -Wall -Wextra -Wpedantic
    -Wformat=2                   # strict printf/scanf format checking
    -Wformat-security            # catch %n format string bugs
    -fstack-protector-strong     # SSP for functions with arrays/pointers
    -D_FORTIFY_SOURCE=2          # glibc/musl bounds checks (needs -O2+)
    -O2                          # required for FORTIFY to be effective
    -fPIE                        # position-independent code
)

add_link_options(
    -pie                         # position-independent executable
    -Wl,-z,relro                 # mark non-PLT GOT read-only after load
    -Wl,-z,now                   # resolve all symbols eagerly (full RELRO)
    -Wl,-z,noexecstack           # mark stack non-executable
)

# Strip symbols in release mode (Buildroot does this via BR2_STRIP_strip)
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_link_options(-s)
endif()

add_executable(my_hardened_app
    src/main.cpp
    src/input_handler.cpp
)
```

---

## Rust Code Examples

Rust's memory safety model eliminates whole classes of vulnerabilities at compile time
(no buffer overflows, no use-after-free, no null pointer dereferences in safe code).
However, additional hardening is still important for: FFI boundaries, cryptographic
operations, and privilege management.

### Example 6: Safe Input Processing in Rust (no unsafe)

```rust
//! src/main.rs
//! Build with Buildroot Rust support: BR2_PACKAGE_HOST_RUSTC=y
//! Cross-compile: cargo build --target armv7-unknown-linux-musleabihf --release

use std::io::{self, BufRead};

const MAX_LINE_LEN: usize = 256;
const MAX_LINES: usize = 1024;

/// Process lines from stdin with strict length limits.
/// Returns processed lines or an error description.
fn process_input(reader: impl BufRead) -> Result<Vec<String>, String> {
    let mut results = Vec::new();

    for (line_no, line_result) in reader.lines().enumerate() {
        if line_no >= MAX_LINES {
            return Err(format!("Too many lines (max {})", MAX_LINES));
        }

        let line = line_result.map_err(|e| format!("IO error on line {}: {}", line_no, e))?;

        if line.len() > MAX_LINE_LEN {
            return Err(format!(
                "Line {} too long: {} chars (max {})",
                line_no, line.len(), MAX_LINE_LEN
            ));
        }

        // Sanitise: keep only printable ASCII
        let sanitised: String = line
            .chars()
            .filter(|c| c.is_ascii() && !c.is_ascii_control())
            .collect();

        results.push(sanitised);
    }

    Ok(results)
}

fn main() {
    let stdin = io::stdin();
    match process_input(stdin.lock()) {
        Ok(lines) => {
            println!("Processed {} lines:", lines.len());
            for (i, line) in lines.iter().enumerate() {
                println!("  {:4}: {}", i + 1, line);
            }
        }
        Err(e) => {
            eprintln!("Error: {}", e);
            std::process::exit(1);
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Cursor;

    #[test]
    fn test_normal_input() {
        let input = "hello\nworld\n";
        let result = process_input(Cursor::new(input)).unwrap();
        assert_eq!(result, vec!["hello", "world"]);
    }

    #[test]
    fn test_line_too_long() {
        let long_line = "A".repeat(MAX_LINE_LEN + 1) + "\n";
        let result = process_input(Cursor::new(long_line));
        assert!(result.is_err());
    }

    #[test]
    fn test_control_chars_stripped() {
        // Null bytes and control chars should be removed
        let input = "hello\x00world\x01\n";
        let result = process_input(Cursor::new(input)).unwrap();
        assert_eq!(result[0], "helloworld");
    }

    #[test]
    fn test_too_many_lines() {
        let many_lines = "line\n".repeat(MAX_LINES + 1);
        let result = process_input(Cursor::new(many_lines));
        assert!(result.is_err());
    }
}
```

### Example 7: Privilege Dropping in Rust (for daemons)

```rust
//! privilege.rs — Drop root privileges after binding privileged resources.
//! Used for daemons that start as root (e.g., to bind port 80) then drop to
//! a non-privileged user, consistent with Buildroot hardening philosophy.
//!
//! Add to Cargo.toml:
//! [dependencies]
//! libc = "0.2"

use libc::{getpwnam, setgid, setuid, gid_t, uid_t};
use std::ffi::CString;

#[derive(Debug)]
pub enum PrivilegeError {
    UserNotFound(String),
    SetGidFailed(i32),
    SetUidFailed(i32),
    AlreadyUnprivileged,
}

impl std::fmt::Display for PrivilegeError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::UserNotFound(u)   => write!(f, "User not found: {}", u),
            Self::SetGidFailed(e)   => write!(f, "setgid failed: errno {}", e),
            Self::SetUidFailed(e)   => write!(f, "setuid failed: errno {}", e),
            Self::AlreadyUnprivileged => write!(f, "Process is not running as root"),
        }
    }
}

/// Drop privileges to the named user account.
/// Must be called AFTER binding privileged sockets / opening raw devices.
///
/// # Safety
/// Calls libc functions; safe to call once at daemon startup.
pub fn drop_privileges(username: &str) -> Result<(), PrivilegeError> {
    // Safety check: only meaningful when running as root
    if unsafe { libc::getuid() } != 0 {
        return Err(PrivilegeError::AlreadyUnprivileged);
    }

    let cname = CString::new(username).expect("username contains NUL byte");

    // Look up the target user in /etc/passwd
    let pw = unsafe { getpwnam(cname.as_ptr()) };
    if pw.is_null() {
        return Err(PrivilegeError::UserNotFound(username.to_owned()));
    }

    let (target_uid, target_gid): (uid_t, gid_t) = unsafe {
        ((*pw).pw_uid, (*pw).pw_gid)
    };

    // Drop group first (cannot do it after dropping uid)
    let ret_gid = unsafe { setgid(target_gid) };
    if ret_gid != 0 {
        let errno = unsafe { *libc::__errno_location() };
        return Err(PrivilegeError::SetGidFailed(errno));
    }

    // Drop user
    let ret_uid = unsafe { setuid(target_uid) };
    if ret_uid != 0 {
        let errno = unsafe { *libc::__errno_location() };
        return Err(PrivilegeError::SetUidFailed(errno));
    }

    // Verify we cannot regain root
    assert_ne!(unsafe { libc::getuid() }, 0, "Privilege drop failed silently!");
    assert_ne!(unsafe { libc::getgid() }, 0, "Group privilege drop failed!");

    Ok(())
}

fn main() {
    // --- Privileged phase: bind socket, open raw devices, etc. ---
    println!("[*] Running as uid={}", unsafe { libc::getuid() });

    // Drop to 'nobody' after privileged setup
    match drop_privileges("nobody") {
        Ok(()) => println!("[+] Privileges dropped to 'nobody' (uid={})", unsafe { libc::getuid() }),
        Err(e) => {
            eprintln!("[-] Could not drop privileges: {}", e);
            std::process::exit(1);
        }
    }

    // --- Unprivileged phase ---
    println!("[*] Running as uid={} — privilege drop successful", unsafe { libc::getuid() });
}
```

### Example 8: Rust Cargo Config for Buildroot Cross-Compilation with Hardening

```toml
# .cargo/config.toml — Cross-compilation config for Buildroot ARM target
# Buildroot generates the cross toolchain at output/host/bin/

[target.armv7-unknown-linux-musleabihf]
linker = "arm-buildroot-linux-musleabihf-gcc"

# Pass hardening flags through the linker (rustc wraps the C linker)
rustflags = [
    # Stack protector via C linker flag
    "-C", "link-arg=-fstack-protector-strong",
    # Full RELRO
    "-C", "link-arg=-Wl,-z,relro",
    "-C", "link-arg=-Wl,-z,now",
    # Non-executable stack
    "-C", "link-arg=-Wl,-z,noexecstack",
]

[profile.release]
# Rust release mode already omits debug symbols; these further harden the binary:
opt-level     = 2
strip         = true      # strip symbols (equiv. to BR2_STRIP_strip)
panic         = "abort"   # abort on panic — no unwinding infra in embedded
lto           = true      # link-time optimisation — smaller, harder to reverse
codegen-units = 1         # single codegen unit for best LTO
```

---

## Security Layers Diagram (ASCII)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              BUILDROOT SECURITY HARDENING — LAYERED DEFENCES                ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  LAYER 6 — SYSTEM LEVEL                                                      ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • Read-only rootfs (BR2_TARGET_ROOTFS_*  + kernel ro mount)         │     ║
║  │ • No dev tools (gcc/make/ld removed from target)                    │     ║
║  │ • Minimal package selection                                         │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                              │ if attacker breaks through                    ║
║                              ▼                                               ║
║  LAYER 5 — PRIVILEGE MANAGEMENT                                              ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • SUID bits stripped (post-build.sh)                                │     ║
║  │ • Daemons run as unprivileged users                                 │     ║
║  │ • Linux capabilities instead of blanket root                        │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                              │ if attacker gains code exec                   ║
║                              ▼                                               ║
║  LAYER 4 — ADDRESS SPACE RANDOMISATION                                       ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • PIE (-fPIE / -pie)  →  randomises executable base                │     ║
║  │ • Kernel ASLR (CONFIG_RANDOMIZE_BASE=y)                             │     ║
║  │ • NX stack (PT_GNU_STACK non-exec)                                  │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                              │ if attacker finds a leak                      ║
║                              ▼                                               ║
║  LAYER 3 — GOT / PLT PROTECTION                                              ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • Full RELRO  (-Wl,-z,relro,-z,now)  →  GOT becomes read-only      │     ║
║  │ • BR2_RELRO_FULL=y applies to all packages                          │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                              │ if attacker attempts write-what-where         ║
║                              ▼                                               ║
║  LAYER 2 — LIBRARY BOUNDS CHECKING                                           ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • FORTIFY_SOURCE=2  →  __memcpy_chk, __strcpy_chk, etc.            │     ║
║  │ • Compile-time rejection of provably-unsafe calls                   │     ║
║  │ • Run-time abort on dynamic overflow                                │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                              │ if attacker overflows a buffer                ║
║                              ▼                                               ║
║  LAYER 1 — STACK CANARY (FIRST LINE OF DEFENCE)                              ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • -fstack-protector-strong  →  canary between locals and ret addr   │     ║
║  │ • BR2_SSP_STRONG=y  applies globally                                │     ║
║  │ • __stack_chk_fail() aborts before corrupted return executes        │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## Attack Surface Reduction Diagram (ASCII)

```
  ATTACK SURFACE: BEFORE vs AFTER BUILDROOT HARDENING
  ════════════════════════════════════════════════════

  BEFORE (default / unhardened):
  ┌────────────────────────────────────────────────────────────────────────┐
  │                                                                        │
  │  [gcc] [g++] [ld] [as] [make] [cmake] [pkg-config] [strip] [objdump]  │  ← dev tools
  │  [python3] [perl] [ruby] [lua]                                         │  ← scripting
  │  [wget] [curl] [ftp] [tftp] [nc] [nmap]                               │  ← network tools
  │  [bash] [sh] [dash] [zsh] [csh]                                       │  ← many shells
  │  [/usr/bin/passwd SUID] [/bin/su SUID] [/usr/bin/pkexec SUID]        │  ← SUID bins
  │  [libelf] [libdwarf] [debug symbols in every binary]                  │  ← debug info
  │                                                                        │
  │  No PIE | No SSP | No RELRO | No FORTIFY | No NX                      │
  │                                                                        │
  │  Attack surface: ████████████████████████████████████████  HUGE        │
  └────────────────────────────────────────────────────────────────────────┘

  AFTER (BR2_SSP_STRONG + BR2_RELRO_FULL + BR2_PIE + post-build strip):
  ┌────────────────────────────────────────────────────────────────────────┐
  │                                                                        │
  │  [busybox] (minimal shell + utilities, compiled with all flags)        │  ← minimal shell
  │  [your_app] (stripped, PIE, SSP, RELRO, FORTIFY)                      │  ← application
  │  [required libs only] (stripped)                                       │  ← libs
  │  [/bin/su SUID only if needed]                                         │  ← minimal SUID
  │                                                                        │
  │  PIE ✓ | SSP-strong ✓ | Full RELRO ✓ | FORTIFY=2 ✓ | NX ✓           │
  │                                                                        │
  │  Attack surface: ████  MINIMAL                                         │
  └────────────────────────────────────────────────────────────────────────┘

  REDUCTION BREAKDOWN:
  ┌────────────────────────┬──────────────────┬──────────────────┐
  │ Category               │ Before           │ After            │
  ├────────────────────────┼──────────────────┼──────────────────┤
  │ Dev / build tools      │ gcc, make, ld…   │ none             │
  │ Interpreters           │ python, perl…    │ none             │
  │ Network tools          │ wget, curl, nc…  │ none             │
  │ SUID binaries          │ many             │ 0–1 (minimal)    │
  │ Debug symbols          │ in every binary  │ stripped all     │
  │ Memory corruption      │ unprotected      │ SSP + FORTIFY    │
  │ Code-reuse attacks     │ possible         │ PIE + RELRO      │
  └────────────────────────┴──────────────────┴──────────────────┘
```

---

## Buildroot menuconfig Reference

Navigate in `make menuconfig` to set these options interactively:

```
  make menuconfig path:

  Build options  ──────────────────────────────────────────────────────────────
  └── [*] Enable compiler hardening options
      ├── Stack Smashing Protection
      │   (X) Strong (recommended)     → BR2_SSP_STRONG=y
      │   ( ) Regular
      │   ( ) All
      │   ( ) None
      ├── [*] Use Position Independent Executables (PIE)
      │                                → BR2_PIE=y
      ├── RELRO protection
      │   (X) Full (recommended)       → BR2_RELRO_FULL=y
      │   ( ) Partial
      │   ( ) None
      └── [*] Fortify Source           → implicit via BR2_OPTIMIZE_2=y
                                         (FORTIFY requires -O2)

  Target options → Binary utilities
  └── Strip target binaries            → BR2_STRIP_strip=y

  System configuration
  └── [*] Run a post-build script
      (board/myboard/post-build.sh)    → BR2_ROOTFS_POST_BUILD_SCRIPT
```

### Complete defconfig Snippet

```
# Security hardening settings for production embedded system
# Place in configs/myboard_hardened_defconfig

# Stack Smashing Protector — strong is best balance
BR2_SSP_STRONG=y

# Position Independent Executables
BR2_PIE=y

# Full RELRO — resolves all PLT at load time, makes GOT read-only
BR2_RELRO_FULL=y

# Strip all binaries and libraries in target rootfs
BR2_STRIP_strip=y
BR2_STRIP_EXCLUDE_FILES=""
BR2_STRIP_EXCLUDE_DIRS=""

# Optimisation level (required for FORTIFY_SOURCE to work)
BR2_OPTIMIZE_2=y

# Post-build script for SUID removal and custom hardening
BR2_ROOTFS_POST_BUILD_SCRIPT="board/myboard/post-build.sh"

# Do not install host tools into target
# (These are implicit — just don't select host-* packages)
# BR2_PACKAGE_GCC_FINAL is host-only, never in target

# Disable /proc/config.gz (kernel config leak)
# Set in kernel config: # CONFIG_IKCONFIG is not set
```

---

## Summary

Security hardening in Buildroot is a multi-layered, build-time-enforced discipline. No single
mechanism is sufficient; the goal is to make exploitation progressively more difficult at
every layer.

**The five core mechanisms and their Buildroot controls are:**

1. **Stack Smashing Protection (`BR2_SSP_STRONG`)** inserts random canary values to detect
   stack buffer overflows before a corrupted return address is executed. The `strong` variant
   protects all functions that contain arrays or take addresses of locals — the right balance
   between coverage and performance overhead for embedded targets.

2. **FORTIFY_SOURCE (`-D_FORTIFY_SOURCE=2`, enabled implicitly with `-O2`)** replaces unsafe
   libc calls with bounds-checked variants. Detectable overflows are caught at compile time;
   dynamic ones abort the process at run time. Requires optimisation to be effective.

3. **PIE and ASLR (`BR2_PIE`)** randomise the load address of the executable itself. Combined
   with the kernel's ASLR for the heap, stack, and libraries, this defeats hardcoded-address
   shellcode and makes ROP chain construction impractical on most 64-bit (and increasingly on
   32-bit) targets.

4. **Full RELRO (`BR2_RELRO_FULL`)** forces eager symbol resolution at load time, then marks
   the entire Global Offset Table read-only. This eliminates a major class of write-what-where
   exploit primitives that overwrite GOT entries to redirect execution.

5. **Minimal rootfs, stripped binaries, no SUID (`BR2_STRIP_strip`, post-build script)**
   reduces the attack surface to the bare minimum needed for operation. Stripping symbols
   removes information useful to reverse engineers. Eliminating compilers, interpreters, and
   network tools makes post-exploitation persistence far harder. Removing or replacing SUID
   binaries with capability-based alternatives prevents trivial privilege escalation.

Together, these measures transform a generic Buildroot-built system into a hardened embedded
platform that applies the same defence-in-depth principles used in modern general-purpose
Linux security distributions — within the constraints of resource-limited, long-lifetime
embedded deployments.

**Key Buildroot defconfig symbols:**

```
BR2_SSP_STRONG=y
BR2_PIE=y
BR2_RELRO_FULL=y
BR2_STRIP_strip=y
BR2_OPTIMIZE_2=y
BR2_ROOTFS_POST_BUILD_SCRIPT="board/myboard/post-build.sh"
```

---

*Document: 22_Security_Hardening.md — Part of the Buildroot Programming Guide series.*