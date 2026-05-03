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
  │  ┌──────────┐  ┌──────────┐  ┌────────────────┐     │
  │  │   gcc    │  │   make   │  │ binutils/strip │     │
  │  └──────────┘  └──────────┘  └────────────────┘     │
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
║              BUILDROOT SECURITY HARDENING — LAYERED DEFENCES                 ║
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
║  │ • PIE (-fPIE / -pie)  →  randomises executable base                 │     ║
║  │ • Kernel ASLR (CONFIG_RANDOMIZE_BASE=y)                             │     ║
║  │ • NX stack (PT_GNU_STACK non-exec)                                  │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                              │ if attacker finds a leak                      ║
║                              ▼                                               ║
║  LAYER 3 — GOT / PLT PROTECTION                                              ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • Full RELRO  (-Wl,-z,relro,-z,now)  →  GOT becomes read-only       │     ║
║  │ • BR2_RELRO_FULL=y applies to all packages                          │     ║
║  └─────────────────────────────────────────────────────────────────────┘     ║
║                              │ if attacker attempts write-what-where         ║
║                              ▼                                               ║
║  LAYER 2 — LIBRARY BOUNDS CHECKING                                           ║
║  ┌─────────────────────────────────────────────────────────────────────┐     ║
║  │ • FORTIFY_SOURCE=2  →  __memcpy_chk, __strcpy_chk, etc.             │     ║
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
  │  [gcc] [g++] [ld] [as] [make] [cmake] [pkg-config] [strip] [objdump]   │  ← dev tools
  │  [python3] [perl] [ruby] [lua]                                         │  ← scripting
  │  [wget] [curl] [ftp] [tftp] [nc] [nmap]                                │  ← network tools
  │  [bash] [sh] [dash] [zsh] [csh]                                        │  ← many shells
  │  [/usr/bin/passwd SUID] [/bin/su SUID] [/usr/bin/pkexec SUID]          │  ← SUID bins
  │  [libelf] [libdwarf] [debug symbols in every binary]                   │  ← debug info
  │                                                                        │
  │  No PIE | No SSP | No RELRO | No FORTIFY | No NX                       │
  │                                                                        │
  │  Attack surface: ████████████████████████████████████████  HUGE        │
  └────────────────────────────────────────────────────────────────────────┘

  AFTER (BR2_SSP_STRONG + BR2_RELRO_FULL + BR2_PIE + post-build strip):
  ┌────────────────────────────────────────────────────────────────────────┐
  │                                                                        │
  │  [busybox] (minimal shell + utilities, compiled with all flags)        │  ← minimal shell
  │  [your_app] (stripped, PIE, SSP, RELRO, FORTIFY)                       │  ← application
  │  [required libs only] (stripped)                                       │  ← libs
  │  [/bin/su SUID only if needed]                                         │  ← minimal SUID
  │                                                                        │
  │  PIE ✓ | SSP-strong ✓ | Full RELRO ✓ | FORTIFY=2 ✓ | NX ✓              │
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

---

# 22. Security Hardening in Buildroot

**Structure:**
- **Introduction & threat model** — why hardening matters on embedded targets, with an ASCII pipeline diagram
- **Compiler flags** deep-dives — stack canary layout, ASLR address-space diagrams, GOT/PLT before-and-after RELRO
- **Buildroot Kconfig** — full table of `BR2_SSP_*` and `BR2_RELRO_*` options with their injected flags and a ready-to-use defconfig fragment

**Code examples:**
- **C (3 examples):** stack canary demo, safe string handling with `snprintf`/`strnlen`, and a self-checking binary that probes for SSP/FORTIFY/PIE at runtime
- **C++ (2 examples):** a PIE-hardened socket service and an ELF security property checker that produces ASCII audit reports
- **Rust (3 examples):** a safe config file parser with explicit error types and length enforcement, the Buildroot `.mk` package file for Cargo, and a Unix-socket IPC server demonstrating Rust's ownership-based memory safety

**All graphics are ASCII art** — stack frame diagrams, virtual memory layouts, GOT overwrite models, FORTIFY_SOURCE internals, and audit report tables.


## Table of Contents

1. [Introduction](#introduction)
2. [Why Security Hardening Matters in Embedded Systems](#why-security-hardening-matters)
3. [Compiler Hardening Flags](#compiler-hardening-flags)
4. [Buildroot Security Configuration Options](#buildroot-security-configuration-options)
5. [Stripping SUID Binaries](#stripping-suid-binaries)
6. [Removing Development Tools](#removing-development-tools)
7. [Stack Protection in Practice — C Examples](#stack-protection-in-practice--c-examples)
8. [Position-Independent Executables — C++ Examples](#position-independent-executables--c-examples)
9. [Memory Safety with Rust](#memory-safety-with-rust)
10. [Fortify Source Deep Dive](#fortify-source-deep-dive)
11. [RELRO and Full Hardening Pipeline](#relro-and-full-hardening-pipeline)
12. [Verification and Audit Tools](#verification-and-audit-tools)
13. [Summary](#summary)

---

## Introduction

Security hardening in Buildroot refers to the systematic process of reducing the attack surface of an embedded Linux system by applying compiler-level protections, removing unnecessary privileges, stripping development artifacts, and configuring the build system to enforce hardened defaults across every package.

Buildroot provides a curated set of `BR2_SSP_*` and `BR2_RELRO_*` Kconfig options that inject hardening flags globally into every package's build process, so developers do not need to patch individual Makefiles. This chapter covers the full hardening pipeline from the compiler command line all the way to final image verification.

```
  ┌─────────────────────────────────────────────────────────┐
  │              BUILDROOT SECURITY HARDENING               │
  │                                                         │
  │  Source Code                                            │
  │      │                                                  │
  │      ▼                                                  │
  │  ┌──────────────────────────────────┐                   │
  │  │  Compiler (GCC / Clang)          │                   │
  │  │  -fstack-protector-strong        │                   │
  │  │  -D_FORTIFY_SOURCE=2             │                   │
  │  │  -fPIE / -pie                    │                   │
  │  │  -Wl,-z,relro,-z,now             │                   │
  │  └──────────────────────────────────┘                   │
  │      │                                                  │
  │      ▼                                                  │
  │  ┌──────────────────────────────────┐                   │
  │  │  Linker / strip pass             │                   │
  │  │  Strip SUID bits                 │                   │
  │  │  Remove dev tools                │                   │
  │  └──────────────────────────────────┘                   │
  │      │                                                  │
  │      ▼                                                  │
  │  ┌──────────────────────────────────┐                   │
  │  │  Root Filesystem Image           │                   │
  │  │  Hardened, minimal, production   │                   │
  │  └──────────────────────────────────┘                   │
  └─────────────────────────────────────────────────────────┘
```

---

## Why Security Hardening Matters in Embedded Systems

Embedded Linux systems occupy an increasingly hostile environment: IoT devices exposed to the internet, industrial controllers on flat networks, automotive ECUs with remote-update channels, and medical equipment with USB ports. Unlike desktop systems, embedded targets often:

- Run a single long-lived firmware image for years without patches.
- Lack runtime exploit mitigations (ASLR may be weak on 32-bit targets).
- Ship with default credentials or open services.
- Are physically accessible to adversaries.

The compiler and linker hardening flags described in this chapter impose virtually zero runtime cost while dramatically raising the cost of exploiting memory-corruption vulnerabilities. They are a mandatory baseline for any production Buildroot image.

```
  Attack Surface Reduction Model
  ═══════════════════════════════

  BEFORE hardening                AFTER hardening
  ─────────────────               ──────────────────
  Stack buffer overflow           Canary detects overwrite
  ┌─────────────────┐             ┌──────────────────────┐
  │ buf[16]         │             │ buf[16]              │
  │ saved RIP ◄─ !! │             │ canary  ◄─ checked   │
  │ ret addr        │             │ saved RIP            │
  └─────────────────┘             └──────────────────────┘

  GOT overwrite (no RELRO)        Read-only GOT (FULL RELRO)
  ┌───────────┐                   ┌───────────────────────┐
  │ GOT entry │ ◄── writable      │ GOT entry  read-only  │
  │ .got.plt  │     exploitable   │ .got.plt   mprotect'd │
  └───────────┘                   └───────────────────────┘

  Known libc addresses            ASLR + PIE randomises base
  text = 0x08048000 (fixed)       text = 0x5a3f1000 (random)
```

---

## Compiler Hardening Flags

### `-fstack-protector-strong`

Inserts a random canary value between local variables and the saved return address. On function exit the canary is checked; a mismatch causes an immediate `__stack_chk_fail()` abort. The `-strong` variant instruments all functions that declare arrays, take addresses of locals, or call `alloca` — a much better coverage than the older plain `-fstack-protector`.

```
  Stack frame WITH -fstack-protector-strong
  ─────────────────────────────────────────
  High addresses
  ┌──────────────────────┐
  │  caller's frame      │
  ├──────────────────────┤  ◄── frame base (rbp / fp)
  │  saved return addr   │
  │  saved frame pointer │
  ├──────────────────────┤
  │  STACK CANARY        │  ◄── random 8-byte value
  ├──────────────────────┤       checked on return
  │  local variables     │
  │    int  x            │
  │    char buf[64]      │  ◄── overflow target
  │    ...               │
  ├──────────────────────┤
  │  (red zone / align)  │
  └──────────────────────┘
  Low addresses
```

### `-D_FORTIFY_SOURCE=2`

Enables compile-time and runtime bounds checking on common libc functions (`memcpy`, `strcpy`, `sprintf`, `read`, `write`, etc.). At level 2, calls where the buffer size is statically known are verified; unknown-size calls get runtime wrappers (`__memcpy_chk`, `__sprintf_chk`, etc.).

Requires `-O1` or higher to be effective because the compiler must be able to determine buffer sizes.

### `-fPIE` / `-pie`

`-fPIE` (compile) and `-pie` (link) produce a Position-Independent Executable. Combined with the kernel's Address Space Layout Randomisation (ASLR), every run loads the executable at a different base address, making ROP-chain construction dramatically harder.

```
  Virtual Memory Layout

  Without PIE          With PIE + ASLR
  ────────────         ───────────────
  0x08048000 [text]    0x5571a000 [text]   ← random each run
  0x0804a000 [data]    0x5571c000 [data]
  0xf7e00000 [libc]    0x7f3e5000 [libc]   ← independent random
  0xffffd000 [stack]   0x7ffd3000 [stack]  ← independent random
```

### `-Wl,-z,relro` and `-Wl,-z,now`

These are linker flags (passed via `-Wl`):

- `-z relro` marks the GOT and other relocatable sections read-only after startup dynamic linking is complete (partial RELRO).
- `-z now` forces all symbol resolution at load time (no lazy binding), allowing the entire `.got.plt` to be made read-only (full RELRO).

---

## Buildroot Security Configuration Options

Buildroot exposes hardening settings through `make menuconfig` under **Build options → Security hardening options**.

```
  menuconfig tree (abbreviated)
  ══════════════════════════════

  Build options
  └── Security hardening options
      ├── [*] Stack Smashing Protection
      │       (BR2_SSP_REGULAR / STRONG / ALL)
      │
      ├── [*] FORTIFY_SOURCE support
      │       (BR2_FORTIFY_SOURCE_1 / 2)
      │
      ├── [*] Position Independent Executables (PIE)
      │       (BR2_PIE)
      │
      ├── [*] RELRO protection
      │       (BR2_RELRO_PARTIAL / FULL)
      │
      └── [*] Strip binaries
              (BR2_STRIP_strip / sstrip / none)
```

### `BR2_SSP_*` Options

| Kconfig symbol       | GCC flags added              | Coverage                        |
|----------------------|------------------------------|---------------------------------|
| `BR2_SSP_NONE`       | (none)                       | No protection                   |
| `BR2_SSP_REGULAR`    | `-fstack-protector`          | Functions with 8+ byte buffers  |
| `BR2_SSP_STRONG`     | `-fstack-protector-strong`   | Arrays, alloca, address-taken   |
| `BR2_SSP_ALL`        | `-fstack-protector-all`      | Every function (high overhead)  |

### `BR2_RELRO_*` Options

| Kconfig symbol          | Linker flags                        | Effect                       |
|-------------------------|-------------------------------------|------------------------------|
| `BR2_RELRO_NONE`        | (none)                              | Writable GOT                 |
| `BR2_RELRO_PARTIAL`     | `-Wl,-z,relro`                      | Partial read-only GOT        |
| `BR2_RELRO_FULL`        | `-Wl,-z,relro,-z,now`               | Full read-only GOT + eager   |

These symbols inject flags via `TARGET_CFLAGS` and `TARGET_LDFLAGS` in Buildroot's `package/Makefile.in` infrastructure, so every package using the standard build system inherits them automatically.

### Example `defconfig` fragment

```ini
# Security hardening — paste into your board defconfig
BR2_SSP_STRONG=y
BR2_FORTIFY_SOURCE_2=y
BR2_PIE=y
BR2_RELRO_FULL=y
BR2_STRIP_strip=y
```

---

## Stripping SUID Binaries

SUID (Set-User-ID) binaries execute with the file owner's privileges regardless of which user runs them. In a minimal embedded rootfs most SUID binaries are unnecessary and represent a privilege escalation vector.

### Finding SUID Binaries in the Staging Tree

```bash
# Run inside the Buildroot output/target directory
find output/target -perm /4000 -ls 2>/dev/null
```

```
  Typical SUID candidates in a Buildroot image
  ─────────────────────────────────────────────
  /usr/bin/passwd       ← needed if users change passwords
  /usr/bin/su           ← needed only if multi-user
  /bin/busybox          ← if busybox is SUID-installed
  /usr/bin/ping         ← needs raw socket; use cap_net_raw instead
```

### Removing SUID in a Post-build Script

Buildroot allows a `BR2_ROOTFS_POST_BUILD_SCRIPT` to run after all packages are installed but before image creation.

```bash
#!/bin/sh
# board/myboard/post-build.sh
# Strip SUID/SGID bits from all binaries except those explicitly allowed

TARGET="$1"
ALLOWED="
/bin/busybox
"

find "${TARGET}" -perm /6000 | while read -r f; do
    base="${f#${TARGET}}"
    keep=0
    for a in ${ALLOWED}; do
        [ "$base" = "$a" ] && keep=1 && break
    done
    if [ $keep -eq 0 ]; then
        echo "[hardening] removing SUID/SGID from ${base}"
        chmod ug-s "${TARGET}${base}"
    fi
done
```

Register it in your `defconfig`:

```ini
BR2_ROOTFS_POST_BUILD_SCRIPT="board/myboard/post-build.sh"
```

---

## Removing Development Tools

Every compiler, debugger, assembler, or interpreter left in the target rootfs is a free exploit-development toolkit for an attacker who has gained initial access.

### What to Exclude

```
  Development tool categories to exclude from target
  ───────────────────────────────────────────────────
  Compilers        gcc, g++, clang, tcc
  Assemblers       as, nasm, yasm
  Debuggers        gdb, gdbserver, strace, ltrace
  Interpreters     python3, perl, lua (if not needed at runtime)
  Build tools      make, cmake, pkgconf, autoconf
  Linkers          ld, gold, lld (only needed on host)
  Headers          /usr/include/** (strip package)
  Static libs      *.a (strip package or BR2_PACKAGE_*_STATIC=n)
```

### Buildroot Package Exclusion

Most development packages are simply not selected. The strip step handles libraries:

```ini
# Ensure no dev-only packages sneak in
# (these are typically off by default but worth verifying)
# BR2_PACKAGE_GDB is not set
# BR2_PACKAGE_STRACE is not set
# BR2_PACKAGE_LTRACE is not set
# BR2_PACKAGE_PYTHON3 is not set   (if not needed at runtime)
BR2_STRIP_strip=y
BR2_PREFER_STATIC_LIB=n           # avoid pulling in .a files
```

### Post-image Audit One-liner

```bash
# Check for unexpected binaries with executable stacks or no RELRO
find output/target/usr/bin output/target/bin \
    -type f -executable \
    -exec sh -c 'readelf -l "$1" 2>/dev/null | grep -q GNU_STACK \
        || echo "no GNU_STACK: $1"' _ {} \;
```

---

## Stack Protection in Practice — C Examples

### Example 1: Demonstrating Stack Canary Detection (C)

```c
/*
 * stack_demo.c
 *
 * Compile with:    gcc -fstack-protector-strong -O2 -o stack_demo stack_demo.c
 * Compile without: gcc -fno-stack-protector   -O2 -o stack_demo_unsafe stack_demo.c
 *
 * The safe version aborts via __stack_chk_fail() before the
 * attacker's overwritten return address can be used.
 */
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* Intentionally vulnerable function — DO NOT ship this */
static void vulnerable(const char *user_input)
{
    char buf[32];
    /*
     * strcpy does not check length; input > 32 chars overwrites
     * the stack canary, triggering abort() before returning.
     */
    strcpy(buf, user_input);
    printf("buf = %s\n", buf);
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <input>\n", argv[0]);
        return 1;
    }
    vulnerable(argv[1]);
    return 0;
}

/*
 * ASCII diagram of what happens at runtime:
 *
 *   stack_demo <"AAAA...AAAA"> (64 A's)
 *
 *   High  ┌─────────────────┐
 *         │  return address │ ← attacker wants to reach here
 *         ├─────────────────┤
 *         │  saved RBP      │
 *         ├─────────────────┤
 *         │  CANARY  0x??   │ ← gets overwritten with 0x41414141...
 *         ├─────────────────┤    __stack_chk_fail() fires
 *         │  buf[32]        │ ← starts here, overflows upward
 *   Low   └─────────────────┘
 *
 * Result: "*** stack smashing detected ***: terminated"
 */
```

### Example 2: Safe String Handling Avoiding SSP Triggers (C)

```c
/*
 * safe_strings.c
 *
 * Demonstrates correct string handling patterns that work
 * alongside -fstack-protector-strong without false positives.
 *
 * Compile: gcc -fstack-protector-strong -D_FORTIFY_SOURCE=2 -O2 \
 *              -fPIE -pie -o safe_strings safe_strings.c
 */
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stddef.h>

#define MAX_NAME_LEN  63
#define MAX_PATH_LEN 255

typedef struct {
    char name[MAX_NAME_LEN + 1];
    char path[MAX_PATH_LEN + 1];
    int  flags;
} config_entry_t;

/*
 * Safe config parser: uses snprintf / strnlen everywhere.
 * -D_FORTIFY_SOURCE=2 wraps these with runtime size checks
 * when sizes are statically known.
 */
static int parse_entry(const char *raw_name,
                       const char *raw_path,
                       config_entry_t *out)
{
    if (!raw_name || !raw_path || !out)
        return -1;

    /* strnlen prevents scanning past the buffer boundary */
    size_t name_len = strnlen(raw_name, MAX_NAME_LEN + 1);
    if (name_len > MAX_NAME_LEN) {
        fprintf(stderr, "name too long (%zu > %d)\n",
                name_len, MAX_NAME_LEN);
        return -1;
    }

    /* snprintf always NUL-terminates; returns chars that would   *
     * have been written, letting us detect truncation.           */
    int r = snprintf(out->name, sizeof(out->name), "%s", raw_name);
    if (r < 0 || (size_t)r >= sizeof(out->name))
        return -1;

    r = snprintf(out->path, sizeof(out->path), "%s", raw_path);
    if (r < 0 || (size_t)r >= sizeof(out->path))
        return -1;

    out->flags = 0;
    return 0;
}

int main(void)
{
    config_entry_t e;
    if (parse_entry("sensor_module", "/dev/ttyS0", &e) == 0)
        printf("name=%s  path=%s\n", e.name, e.path);
    return 0;
}
```

### Example 3: Checking Hardening Flags at Runtime (C)

```c
/*
 * check_hardening.c
 *
 * A small diagnostic that reports which protections are active
 * in the current binary.  Useful in a post-install test suite.
 *
 * Compile: gcc -o check_hardening check_hardening.c
 * Run:     readelf -s check_hardening | grep chk
 *          checksec --file=check_hardening
 */
#include <stdio.h>

/* __stack_chk_guard is defined by glibc when SSP is active */
extern uintptr_t __stack_chk_guard __attribute__((weak));

/* __sprintf_chk is one of the FORTIFY_SOURCE wrappers */
extern int __sprintf_chk(char *, int, size_t, const char *, ...)
    __attribute__((weak));

int main(void)
{
    puts("=== Binary hardening self-check ===");

    /* Stack Smashing Protection */
    if (&__stack_chk_guard)
        printf("[SSP]     Stack canary ACTIVE  (guard = %p)\n",
               (void *)(uintptr_t)__stack_chk_guard);
    else
        puts("[SSP]     Stack canary NOT active");

    /* FORTIFY_SOURCE */
    if (&__sprintf_chk)
        puts("[FORTIFY] FORTIFY_SOURCE wrappers ACTIVE");
    else
        puts("[FORTIFY] FORTIFY_SOURCE NOT active");

    /* PIE: check if the load address is high (ASLR randomised) */
    uintptr_t text_addr = (uintptr_t)main;
    if (text_addr > 0x100000UL)
        printf("[PIE]     Appears to be PIE (main @ %p)\n",
               (void *)text_addr);
    else
        printf("[PIE]     Fixed text address (main @ %p)\n",
               (void *)text_addr);

    return 0;
}
```

---

## Position-Independent Executables — C++ Examples

### Example 4: PIE-compatible Shared Service in C++

```cpp
/*
 * hardened_service.cpp
 *
 * A minimal socket-based service compiled as PIE so ASLR
 * randomises its load address on every execution.
 *
 * Compile:
 *   g++ -std=c++17 -fPIE -pie \
 *       -fstack-protector-strong \
 *       -D_FORTIFY_SOURCE=2 \
 *       -Wl,-z,relro,-z,now \
 *       -O2 -o hardened_service hardened_service.cpp
 *
 * Verify:
 *   readelf -d hardened_service | grep BIND_NOW
 *   readelf -l hardened_service | grep GNU_RELRO
 */
#include <cstdio>
#include <cstring>
#include <cstdlib>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

namespace {

constexpr int    PORT      = 9000;
constexpr size_t BUFSIZE   = 256;
constexpr int    BACKLOG   = 5;

/*
 * ┌──────────────────────────────────────────────┐
 * │  Virtual address space of hardened_service   │
 * │  on two successive runs (PIE + ASLR)         │
 * │                                              │
 * │  Run 1                  Run 2                │
 * │  0x558a32001234 [text]  0x7f3a10001234 [text]│
 * │  0x558a32003000 [data]  0x7f3a10003000 [data]│
 * │  0x558a32004000 [heap]  0x7f3a10004000 [heap]│
 * │                                              │
 * │  Attacker cannot hardcode ROP gadget addrs   │
 * └──────────────────────────────────────────────┘
 */

class Session {
public:
    explicit Session(int fd) : fd_(fd) {}
    ~Session() { if (fd_ >= 0) close(fd_); }

    Session(const Session&)            = delete;
    Session& operator=(const Session&) = delete;

    void run() {
        char buf[BUFSIZE];
        /* snprintf is FORTIFY-wrapped; overflow detected at runtime */
        ssize_t n = recv(fd_, buf, sizeof(buf) - 1, 0);
        if (n <= 0) return;
        buf[n] = '\0';

        char reply[BUFSIZE + 32];
        int r = snprintf(reply, sizeof(reply),
                         "ECHO: %s\r\n", buf);
        if (r > 0 && static_cast<size_t>(r) < sizeof(reply))
            send(fd_, reply, static_cast<size_t>(r), 0);
    }

private:
    int fd_;
};

} // anonymous namespace

int main()
{
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); return 1; }

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR,
               &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(PORT);
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);

    if (bind(server_fd, reinterpret_cast<sockaddr*>(&addr),
             sizeof(addr)) < 0) {
        perror("bind"); return 1;
    }
    listen(server_fd, BACKLOG);
    printf("Listening on 127.0.0.1:%d (PIE hardened)\n", PORT);

    for (;;) {
        int client_fd = accept(server_fd, nullptr, nullptr);
        if (client_fd < 0) continue;
        Session s(client_fd);
        s.run();
    }
}
```

### Example 5: Checking ELF Security Properties Programmatically (C++)

```cpp
/*
 * elf_checker.cpp
 *
 * Parses an ELF binary and reports its security properties.
 * Useful in a Buildroot post-build test or CI pipeline.
 *
 * Compile:
 *   g++ -std=c++17 -fPIE -pie -O2 -o elf_checker elf_checker.cpp
 *
 * Usage:
 *   ./elf_checker output/target/usr/bin/myapp
 */
#include <cstdio>
#include <cstring>
#include <fstream>
#include <string>
#include <vector>
#include <elf.h>

struct SecurityInfo {
    bool is_pie        = false;
    bool has_relro     = false;
    bool has_bind_now  = false;
    bool has_stack_canary = false;
    bool has_gnu_stack = false;
    bool stack_exec    = false;
    bool fortify       = false;
};

/*
 * ASCII report layout:
 *
 *  ┌──────────────────────────────────────────┐
 *  │  ELF Security Report: /usr/bin/myapp     │
 *  ├──────────────┬───────────────────────────┤
 *  │  PIE         │  YES  (ET_DYN)            │
 *  │  RELRO       │  FULL (GNU_RELRO + BIND)  │
 *  │  Stack guard │  YES  (__stack_chk_fail)  │
 *  │  NX stack    │  YES  (RW- not RWE)       │
 *  │  FORTIFY     │  YES  (__memcpy_chk found)│
 *  └──────────────┴───────────────────────────┘
 */
static void print_report(const std::string& path,
                         const SecurityInfo& si)
{
    printf("\n");
    printf("  +------------------------------------------+\n");
    printf("  |  ELF Security Report                     |\n");
    printf("  |  %-40s|\n", path.c_str());
    printf("  +------------------+-----------------------+\n");
    printf("  | PIE              | %-22s|\n",
           si.is_pie ? "YES (ET_DYN)" : "NO  (ET_EXEC)");
    printf("  | RELRO            | %-22s|\n",
           si.has_relro && si.has_bind_now ? "FULL" :
           si.has_relro                   ? "PARTIAL" : "NONE");
    printf("  | Stack canary     | %-22s|\n",
           si.has_stack_canary ? "YES" : "NO");
    printf("  | NX stack         | %-22s|\n",
           si.has_gnu_stack
               ? (si.stack_exec ? "NO (RWX!)" : "YES")
               : "UNKNOWN");
    printf("  | FORTIFY_SOURCE   | %-22s|\n",
           si.fortify ? "YES" : "NO");
    printf("  +------------------+-----------------------+\n\n");
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <elf-file>\n", argv[0]);
        return 1;
    }

    FILE *f = fopen(argv[1], "rb");
    if (!f) { perror("fopen"); return 1; }

    Elf64_Ehdr ehdr;
    if (fread(&ehdr, sizeof(ehdr), 1, f) != 1) {
        fclose(f); return 1;
    }

    SecurityInfo si;

    /* PIE: dynamic executables have type ET_DYN */
    si.is_pie = (ehdr.e_type == ET_DYN);

    /* Walk program headers for GNU_RELRO and GNU_STACK */
    std::vector<Elf64_Phdr> phdrs(ehdr.e_phnum);
    fseek(f, static_cast<long>(ehdr.e_phoff), SEEK_SET);
    fread(phdrs.data(), sizeof(Elf64_Phdr),
          ehdr.e_phnum, f);

    for (const auto& ph : phdrs) {
        if (ph.p_type == PT_GNU_RELRO) si.has_relro    = true;
        if (ph.p_type == PT_GNU_STACK) {
            si.has_gnu_stack = true;
            si.stack_exec    = (ph.p_flags & PF_X) != 0;
        }
    }

    /* Walk dynamic section for BIND_NOW and needed symbols */
    /* (simplified: real implementation would parse .dynsym) */
    /* For demonstration we grep the raw binary for symbol names */
    fclose(f);

    /* Re-open and scan for well-known hardening symbols */
    std::ifstream bin(argv[1], std::ios::binary);
    std::string content((std::istreambuf_iterator<char>(bin)),
                         std::istreambuf_iterator<char>());

    si.has_stack_canary =
        content.find("__stack_chk_fail") != std::string::npos;
    si.fortify =
        content.find("__memcpy_chk")   != std::string::npos ||
        content.find("__sprintf_chk")  != std::string::npos ||
        content.find("__strcpy_chk")   != std::string::npos;

    print_report(argv[1], si);

    /* Exit non-zero if any critical protection is missing */
    bool ok = si.is_pie && si.has_relro && si.has_stack_canary;
    return ok ? 0 : 1;
}
```

---

## Memory Safety with Rust

Rust's ownership model eliminates entire classes of memory-safety vulnerabilities at compile time, making it an excellent choice for new security-sensitive code in a Buildroot image. Buildroot has first-class Rust support via `BR2_PACKAGE_HOST_RUSTC`.

### Buildroot Rust Prerequisites

```ini
# In your defconfig
BR2_PACKAGE_HOST_RUSTC=y
# Cross-compilation target triple is set automatically
# from BR2_ARCH / BR2_ENDIAN
```

### Example 6: Hardened Configuration Parser in Rust

```rust
// config_parser/src/main.rs
//
// A safe configuration file parser for an embedded target.
// No unsafe blocks, no buffer overflows, no use-after-free.
//
// Buildroot package: package/config_parser/
//   config_parser.mk must set:
//     CONFIG_PARSER_VERSION  = 1.0.0
//     CONFIG_PARSER_SITE     = $(TOPDIR)/package/config_parser/src
//     CONFIG_PARSER_SITE_METHOD = local
//     $(eval $(cargo-package))

use std::collections::HashMap;
use std::fs;
use std::path::Path;

/// Parsed configuration entry.
///
/// All string data is owned; no raw pointers, no lifetime
/// management — the compiler guarantees no dangling refs.
#[derive(Debug, Clone)]
pub struct Config {
    entries: HashMap<String, String>,
}

/// Error type for configuration parsing.
#[derive(Debug)]
pub enum ConfigError {
    Io(std::io::Error),
    ParseError { line: usize, detail: String },
    ValueTooLong { key: String, max: usize, got: usize },
}

impl std::fmt::Display for ConfigError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConfigError::Io(e) =>
                write!(f, "I/O error: {e}"),
            ConfigError::ParseError { line, detail } =>
                write!(f, "parse error at line {line}: {detail}"),
            ConfigError::ValueTooLong { key, max, got } =>
                write!(f, "value for '{key}' too long: {got} > {max}"),
        }
    }
}

impl From<std::io::Error> for ConfigError {
    fn from(e: std::io::Error) -> Self {
        ConfigError::Io(e)
    }
}

const MAX_VALUE_LEN: usize = 255;
const MAX_KEY_LEN:   usize = 63;

impl Config {
    /// Parse a key=value text file.
    ///
    /// Lines beginning with '#' are comments.
    /// Whitespace around '=' is stripped.
    pub fn from_file(path: &Path) -> Result<Self, ConfigError> {
        let content = fs::read_to_string(path)?;
        let mut entries = HashMap::new();

        for (lineno, line) in content.lines().enumerate() {
            let line = line.trim();
            if line.is_empty() || line.starts_with('#') {
                continue;
            }

            let (key, value) = line
                .split_once('=')
                .ok_or_else(|| ConfigError::ParseError {
                    line: lineno + 1,
                    detail: format!("missing '=' in '{line}'"),
                })?;

            let key   = key.trim();
            let value = value.trim();

            // Enforce limits — no silent truncation
            if key.len() > MAX_KEY_LEN {
                return Err(ConfigError::ValueTooLong {
                    key: key.to_owned(),
                    max: MAX_KEY_LEN,
                    got: key.len(),
                });
            }
            if value.len() > MAX_VALUE_LEN {
                return Err(ConfigError::ValueTooLong {
                    key: key.to_owned(),
                    max: MAX_VALUE_LEN,
                    got: value.len(),
                });
            }

            entries.insert(key.to_owned(), value.to_owned());
        }

        Ok(Config { entries })
    }

    pub fn get(&self, key: &str) -> Option<&str> {
        self.entries.get(key).map(String::as_str)
    }
}

fn main() {
    let path = Path::new("/etc/myapp.conf");

    match Config::from_file(path) {
        Ok(cfg) => {
            println!("device  = {}",
                cfg.get("device").unwrap_or("(not set)"));
            println!("baud    = {}",
                cfg.get("baud_rate").unwrap_or("115200"));
        }
        Err(e) => {
            eprintln!("Error reading config: {e}");
            std::process::exit(1);
        }
    }
}
```

### Example 7: Rust Buildroot Package Makefile

```makefile
################################################################################
#
# config_parser
#
################################################################################

CONFIG_PARSER_VERSION  = 1.0.0
CONFIG_PARSER_SITE     = $(TOPDIR)/package/config_parser
CONFIG_PARSER_SITE_METHOD = local

# Cargo packages use the generic cargo-package infrastructure.
# Buildroot automatically sets RUSTFLAGS for the target triple,
# adds -C opt-level=2, and strips the binary via BR2_STRIP_strip.
$(eval $(cargo-package))
```

### Example 8: Secure IPC Listener in Rust

```rust
// secure_ipc/src/main.rs
//
// A Unix domain socket server that validates all input.
// Compiled with Rust's default deny-by-default safety model;
// no -fstack-protector needed — the borrow checker prevents
// the underlying memory errors at compile time.

use std::io::{BufRead, BufReader, Write};
use std::os::unix::net::UnixListener;
use std::path::Path;
use std::fs;

const SOCKET_PATH: &str = "/run/myapp/ipc.sock";
const MAX_MSG_LEN: usize = 1024;

/*
 * Ownership model prevents use-after-free and double-free:
 *
 *   ┌────────────────────────────────────────────┐
 *   │  let stream = listener.accept()?           │
 *   │        │                                   │
 *   │        │  stream OWNS the file descriptor  │
 *   │        │                                   │
 *   │  process_client(stream)                    │
 *   │        │  stream MOVED into function       │
 *   │        │  cannot be used after this point  │
 *   │        │                                   │
 *   │        ▼                                   │
 *   │  } ← stream dropped, fd closed             │
 *   │      automatically, no leak possible       │
 *   └────────────────────────────────────────────┘
 */

fn process_client(
    stream: std::os::unix::net::UnixStream,
) -> std::io::Result<()> {
    let mut writer = stream.try_clone()?;
    let reader = BufReader::new(stream);

    for line in reader.lines() {
        let msg = line?;

        // Enforce message length — no unbounded allocation
        if msg.len() > MAX_MSG_LEN {
            writer.write_all(b"ERR: message too long\n")?;
            continue;
        }

        // Validate: only printable ASCII
        if !msg.chars().all(|c| c.is_ascii_graphic() || c == ' ') {
            writer.write_all(b"ERR: invalid characters\n")?;
            continue;
        }

        let response = format!("OK: {msg}\n");
        writer.write_all(response.as_bytes())?;
    }
    Ok(())
}

fn main() -> std::io::Result<()> {
    let sock_path = Path::new(SOCKET_PATH);

    // Remove stale socket
    if sock_path.exists() {
        fs::remove_file(sock_path)?;
    }

    // Ensure parent directory exists
    if let Some(parent) = sock_path.parent() {
        fs::create_dir_all(parent)?;
    }

    let listener = UnixListener::bind(sock_path)?;

    // Restrict socket to owner only (mode 0600)
    fs::set_permissions(
        sock_path,
        std::os::unix::fs::PermissionsExt::from_mode(0o600),
    )?;

    eprintln!("IPC server listening on {SOCKET_PATH}");

    for stream in listener.incoming() {
        match stream {
            Ok(s) => {
                if let Err(e) = process_client(s) {
                    eprintln!("client error: {e}");
                }
            }
            Err(e) => eprintln!("accept error: {e}"),
        }
    }
    Ok(())
}
```

---

## Fortify Source Deep Dive

`-D_FORTIFY_SOURCE=2` works by replacing certain libc functions with checked variants when the destination buffer size is known at compile time.

```
  FORTIFY_SOURCE mechanism
  ════════════════════════

  Source code:
    char buf[64];
    strcpy(buf, src);

  Without FORTIFY:
    call  strcpy@plt          ← no bounds check

  With -D_FORTIFY_SOURCE=2 and known buffer size:
    mov   rdi, buf
    mov   rsi, 64             ← size injected by compiler
    mov   rdx, src
    call  __strcpy_chk@plt    ← runtime bounds check wrapper
                              ← aborts if strlen(src) >= 64

  __strcpy_chk internals (glibc):
  ┌─────────────────────────────────────────────────┐
  │  if (__builtin_expect(strlen(src) >= buflen, 0))│
  │      __chk_fail();   // → abort()               │
  │  return strcpy(dest, src);                      │
  └─────────────────────────────────────────────────┘
```

### Functions Covered by FORTIFY_SOURCE

```
  memcpy   mempcpy   memmove   memset   memchr
  strcpy   strncpy   strcat    strncat  stpcpy
  sprintf  snprintf  vsprintf  vsnprintf
  gets     fgets     read      pread    recv
  recvfrom poll      ppoll     realpath
```

### What FORTIFY_SOURCE Cannot Protect Against

```
  Limitations of FORTIFY_SOURCE=2
  ─────────────────────────────────
  ✗ Heap buffer overflows (heap allocator not instrumented)
  ✗ Integer overflows before length-restricted call
  ✗ Overflows into adjacent stack variables (not past them)
  ✗ Off-by-one where size is passed as a runtime variable
  ✗ Custom memory copy loops (not libc wrappers)

  Complement with:
  ✓ AddressSanitizer in development builds
  ✓ Stack canaries (-fstack-protector-strong)
  ✓ Heap hardening (hardened_malloc, glibc MALLOC_CHECK_)
  ✓ Valgrind / Dr. Memory in CI
```

---

## RELRO and Full Hardening Pipeline

### Understanding the GOT and PLT

```
  Dynamic linking WITHOUT RELRO
  ══════════════════════════════

  Binary calls printf():

  .text                      .plt              .got.plt
  ┌──────────────┐           ┌──────────┐      ┌───────────────┐
  │  call printf │──────────►│ plt stub │─────►│ printf addr   │
  └──────────────┘           └──────────┘      │  (writable!)  │
                                               └───────────────┘
                                                      ▲
                                               Attacker overwrites
                                               with shellcode addr


  With FULL RELRO (-z relro -z now)
  ══════════════════════════════════

  At load time: dynamic linker resolves ALL symbols eagerly,
  then mprotect()s the GOT pages to PROT_READ.

  .text                      .plt              .got  (READ-ONLY)
  ┌──────────────┐           ┌──────────┐      ┌───────────────┐
  │  call printf │──────────►│ plt stub │─────►│ printf addr   │
  └──────────────┘           └──────────┘      │  (read-only)  │
                                               └───────────────┘
                                               Overwrite → SIGSEGV
```

### Full Hardening Compilation Pipeline

```bash
#!/bin/sh
# Hardened build script for a single binary
# Mirrors what Buildroot injects via TARGET_CFLAGS / TARGET_LDFLAGS

CC="${CROSS_COMPILE}gcc"
CXX="${CROSS_COMPILE}g++"

CFLAGS_HARDENED="\
    -fstack-protector-strong \
    -D_FORTIFY_SOURCE=2 \
    -fPIE \
    -O2 \
    -Wall -Wformat -Wformat-security \
    -Werror=format-security"

LDFLAGS_HARDENED="\
    -pie \
    -Wl,-z,relro \
    -Wl,-z,now \
    -Wl,-z,noexecstack"

${CC} ${CFLAGS_HARDENED} -c -o myapp.o myapp.c
${CC} ${LDFLAGS_HARDENED} -o myapp myapp.o

# Verify
echo "=== Checking myapp ==="
readelf -d myapp | grep -E "BIND_NOW|GNU_RELRO" && echo "[OK] RELRO"
readelf -l myapp | grep GNU_STACK
file myapp | grep -q "pie" && echo "[OK] PIE"
```

### Verifying with `checksec`

```
  checksec output for a fully hardened binary:

  ┌─────────────────────────────────────────────────────────────┐
  │  RELRO           STACK CANARY  NX    PIE    RPATH  RUNPATH  │
  │  Full RELRO      Canary found  NX    PIE    No     No       │
  └─────────────────────────────────────────────────────────────┘

  checksec output for an unprotected binary:

  ┌─────────────────────────────────────────────────────────────┐
  │  RELRO           STACK CANARY  NX    PIE    RPATH  RUNPATH  │
  │  No RELRO        No canary     NX    No PIE No     No       │
  └─────────────────────────────────────────────────────────────┘
```

---

## Verification and Audit Tools

### Post-build Hardening Audit Script

```bash
#!/bin/sh
# board/myboard/check_hardening.sh
# Run as: BR2_ROOTFS_POST_BUILD_SCRIPT=board/myboard/check_hardening.sh
#
# Produces a report like:
#
# ┌────────────────────────────────────────────────────┐
# │  Hardening Audit Report                            │
# │  Target: output/target                             │
# ├────────────────────────┬───────────────────────────┤
# │  Binary                │  PIE RELRO SSP  FORTIFY   │
# ├────────────────────────┼───────────────────────────┤
# │  /usr/bin/myapp        │  YES  FULL  YES  YES      │
# │  /bin/busybox          │  YES  FULL  YES  YES      │
# │  /usr/sbin/dropbear    │  YES  PART  YES  YES      │
# │  /usr/lib/libmylib.so  │  --   FULL  YES  YES      │
# └────────────────────────┴───────────────────────────┘

TARGET="${1:-output/target}"
PASS=0
FAIL=0

printf "\n%-40s %-5s %-6s %-5s %-7s\n" \
    "Binary" "PIE" "RELRO" "SSP" "FORTIFY"
printf "%s\n" "$(printf '─%.0s' $(seq 1 65))"

find "${TARGET}" -type f -executable 2>/dev/null | while read -r f; do
    # Only ELF files
    file "$f" 2>/dev/null | grep -q ELF || continue

    pie=$(readelf -h "$f" 2>/dev/null | grep -c "DYN (")
    relro_partial=$(readelf -l "$f" 2>/dev/null | grep -c GNU_RELRO)
    bind_now=$(readelf -d "$f" 2>/dev/null | grep -c BIND_NOW)
    ssp=$(readelf -s "$f" 2>/dev/null | grep -c "__stack_chk_fail")
    fortify=$(readelf -s "$f" 2>/dev/null | grep -c "_chk@")

    pie_s=$([ "$pie"    -gt 0 ] && echo "YES" || echo "NO ")
    if   [ "$relro_partial" -gt 0 ] && [ "$bind_now" -gt 0 ]; then
        relro_s="FULL"
    elif [ "$relro_partial" -gt 0 ]; then
        relro_s="PART"
    else
        relro_s="NONE"
    fi
    ssp_s=$([ "$ssp"    -gt 0 ] && echo "YES" || echo "NO ")
    fort_s=$([ "$fortify" -gt 0 ] && echo "YES" || echo "NO ")

    rel="${f#${TARGET}}"
    printf "%-40s %-5s %-6s %-5s %-7s\n" \
        "${rel}" "${pie_s}" "${relro_s}" "${ssp_s}" "${fort_s}"
done
```

---

## Summary

Security hardening in Buildroot is a layered defence strategy that applies protections at the compiler, linker, and filesystem levels. The key mechanisms and their purposes are:

```
  ┌────────────────────────────────────────────────────────────┐
  │               SECURITY HARDENING SUMMARY                   │
  │                                                            │
  │  Layer 1 — Compiler                                        │
  │  ─────────────────                                         │
  │  -fstack-protector-strong  Detects stack-buffer overflows  │
  │  -D_FORTIFY_SOURCE=2       Bounds-checks libc functions    │
  │  -fPIE                     Enables ASLR for text/data      │
  │                                                            │
  │  Layer 2 — Linker                                          │
  │  ────────────────                                          │
  │  -pie                      Links as position-independent   │
  │  -Wl,-z,relro              Makes partial GOT read-only     │
  │  -Wl,-z,now                Full RELRO (eager binding)      │
  │  -Wl,-z,noexecstack        Non-executable stack pages      │
  │                                                            │
  │  Layer 3 — Filesystem / Image                              │
  │  ─────────────────────────                                 │
  │  Strip SUID/SGID bits      Prevent privilege escalation    │
  │  Remove dev tools          Limit attacker toolkit          │
  │  Strip debug symbols       Reduce information leakage      │
  │                                                            │
  │  Layer 4 — Buildroot Kconfig                               │
  │  ──────────────────────────                                │
  │  BR2_SSP_STRONG            Enables SSP globally            │
  │  BR2_FORTIFY_SOURCE_2      Enables FORTIFY globally        │
  │  BR2_PIE                   Enables PIE globally            │
  │  BR2_RELRO_FULL            Enables full RELRO globally     │
  │  BR2_STRIP_strip           Strips all target binaries      │
  └────────────────────────────────────────────────────────────┘
```

**C and C++** benefit from these flags transparently; existing code requires no changes, though fixing warnings (`-Wformat-security`) may be necessary.

**Rust** provides compile-time memory safety that eliminates the classes of vulnerability that SSP and FORTIFY_SOURCE are designed to catch, making it a natural fit for new security-critical components in a Buildroot image.

Together, these layers raise the cost of exploiting a production Buildroot image from "trivial one-liner" to "sophisticated multi-stage attack requiring novel techniques" — a meaningful improvement for any embedded security posture.

---

*Document: Buildroot Topic 22 — Security Hardening*
*Covers: compiler hardening flags, BR2_SSP_*, BR2_RELRO_*, SUID stripping, dev-tool removal, C / C++ / Rust examples, verification.*