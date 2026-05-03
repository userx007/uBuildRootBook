Now let me create the detailed markdown document:The document covers the full Buildroot Security Hardening topic across 13 sections with 8 code examples:

**Structure:**
- **Introduction & threat model** — why hardening matters on embedded targets, with an ASCII pipeline diagram
- **Compiler flags** deep-dives — stack canary layout, ASLR address-space diagrams, GOT/PLT before-and-after RELRO
- **Buildroot Kconfig** — full table of `BR2_SSP_*` and `BR2_RELRO_*` options with their injected flags and a ready-to-use defconfig fragment

**Code examples:**
- **C (3 examples):** stack canary demo, safe string handling with `snprintf`/`strnlen`, and a self-checking binary that probes for SSP/FORTIFY/PIE at runtime
- **C++ (2 examples):** a PIE-hardened socket service and an ELF security property checker that produces ASCII audit reports
- **Rust (3 examples):** a safe config file parser with explicit error types and length enforcement, the Buildroot `.mk` package file for Cargo, and a Unix-socket IPC server demonstrating Rust's ownership-based memory safety

**All graphics are ASCII art** — stack frame diagrams, virtual memory layouts, GOT overwrite models, FORTIFY_SOURCE internals, and audit report tables.

# 22. Security Hardening in Buildroot

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
  │              BUILDROOT SECURITY HARDENING                │
  │                                                          │
  │  Source Code                                             │
  │      │                                                   │
  │      ▼                                                   │
  │  ┌──────────────────────────────────┐                   │
  │  │  Compiler (GCC / Clang)          │                   │
  │  │  -fstack-protector-strong        │                   │
  │  │  -D_FORTIFY_SOURCE=2             │                   │
  │  │  -fPIE / -pie                    │                   │
  │  │  -Wl,-z,relro,-z,now             │                   │
  │  └──────────────────────────────────┘                   │
  │      │                                                   │
  │      ▼                                                   │
  │  ┌──────────────────────────────────┐                   │
  │  │  Linker / strip pass             │                   │
  │  │  Strip SUID bits                 │                   │
  │  │  Remove dev tools                │                   │
  │  └──────────────────────────────────┘                   │
  │      │                                                   │
  │      ▼                                                   │
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
  ┌─────────────────┐             ┌─────────────────────┐
  │ buf[16]         │             │ buf[16]              │
  │ saved RIP ◄─ !! │             │ canary  ◄─ checked   │
  │ ret addr        │             │ saved RIP            │
  └─────────────────┘             └─────────────────────┘

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
 *         │  return address  │ ← attacker wants to reach here
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