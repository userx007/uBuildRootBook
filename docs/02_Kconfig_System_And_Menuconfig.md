Now let me create the comprehensive markdown document:The document covers the full Kconfig & menuconfig topic across 15 sections. Here's what's inside:

**Architecture & Files**
- Full data-flow ASCII diagram from `Config.in` → `.config` → Makefiles/headers
- Directory layout of the `Config.in` tree
- `.config` format rules and Make variable consumption

**Config.in Language**
- Complete grammar skeleton with all keywords
- All five symbol types (`bool`, `string`, `int`, `hex`, `tristate`) with their `.config` forms
- All dependency mechanisms: `depends on`, `select`, `imply`, `if/endif`, `choice`, `comment`, with operator precedence chart and a dependency graph ASCII diagram

**menuconfig**
- Full ncurses screen layout in ASCII
- Key bindings reference
- Search dialog mockup

**Code Examples**
- **C** — `#ifdef`/`#if` feature gates driven by preprocessor macros, plus a host-side `.config` file parser
- **C++** — policy-based transport class hierarchy (`HttpTransport` / `MqttTransport`) selected at compile time via Kconfig flags, with a `static_assert` build-time guard
- **Rust** — `build.rs` that reads Buildroot env vars and emits `cargo:rustc-cfg` directives, consumed by `#[cfg(myapp_transport_http)]` / `#[cfg(myapp_tls)]` in `main.rs`, plus `Cargo.toml` optional dependencies

**Advanced & Debugging**
- `choice` blocks, versioned symbol selection, arch fragments, `allnoconfig`/`allyesconfig` for CI, `BR2_EXTERNAL` defconfig fragments, and a pitfall reference table

# 02. Kconfig System & menuconfig

> **Buildroot Series** | Topic 02 of N
> A deep-dive into writing and extending `.config` and `Config.in` files, dependency
> expressions, and how the Kconfig machinery maps to package enablement in Buildroot.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture of the Kconfig System](#2-architecture-of-the-kconfig-system)
3. [The `.config` File](#3-the-config-file)
4. [The `Config.in` File Format](#4-the-configin-file-format)
5. [Symbol Types](#5-symbol-types)
6. [Dependency Expressions](#6-dependency-expressions)
7. [menuconfig – The Interactive Interface](#7-menuconfig--the-interactive-interface)
8. [How Kconfig Maps to Package Enablement](#8-how-kconfig-maps-to-package-enablement)
9. [Writing a New Package Config.in](#9-writing-a-new-package-configin)
10. [C Code Example – Reading Kconfig Symbols at Runtime](#10-c-code-example--reading-kconfig-symbols-at-runtime)
11. [C++ Code Example – Kconfig-Driven Feature Flags](#11-c-code-example--kconfig-driven-feature-flags)
12. [Rust Code Example – Conditional Compilation via build.rs](#12-rust-code-example--conditional-compilation-via-buildrs)
13. [Advanced Patterns](#13-advanced-patterns)
14. [Debugging & Pitfalls](#14-debugging--pitfalls)
15. [Summary](#15-summary)

---

## 1. Overview

Kconfig originated in the Linux kernel as a configuration language and tool-set for
describing thousands of build-time options in a structured, hierarchical way.
Buildroot adopted Kconfig wholesale to manage its own configuration space, which
covers target architecture, toolchain, C library, kernel, bootloader, and every
package in the ecosystem.

**Key components:**

| Component        | Role                                                         |
|------------------|--------------------------------------------------------------|
| `Config.in`      | DSL source files describing symbols, types, and dependencies |
| `.config`        | Flat key-value output consumed by Make                       |
| `menuconfig`     | Interactive ncurses UI for editing the configuration         |
| `conf` / `mconf` | Host tools (`scripts/kconfig/`) that process Kconfig files   |
| `autoconf.h`     | (Optional) C header generated from `.config`                 |

```
ASCII: Kconfig Data-Flow
═══════════════════════════════════════════════════════════════════════
  ┌─────────────────────────────────────────────────────────────────┐
  │                      Config.in files                            │
  │   buildroot/Config.in ──► package/*/Config.in ──► toolchain/…  │
  └─────────────────────────────┬───────────────────────────────────┘
                                 │  parsed by
                                 ▼
                    ┌────────────────────────┐
                    │   mconf / conf (host)  │◄──── menuconfig (user)
                    └────────────┬───────────┘
                                 │  writes
                                 ▼
                           ┌──────────┐
                           │ .config  │
                           └────┬─────┘
                                │  included by
                    ┌───────────┴───────────────┐
                    │                           │
                    ▼                           ▼
            ┌─────────────┐         ┌──────────────────┐
            │  Makefile   │         │  include/config/ │
            │ (BR2_…vars) │         │  autoconf.h      │
            └─────────────┘         └──────────────────┘
═══════════════════════════════════════════════════════════════════════
```

---

## 2. Architecture of the Kconfig System

### 2.1 Top-Level Entry Point

Buildroot's root `Config.in` is the single entry point. It pulls everything else in
via `source` directives:

```kconfig
# buildroot/Config.in  (simplified)
mainmenu "Buildroot $(BR2_VERSION) Configuration"

source "arch/Config.in"
source "package/Config.in"
source "toolchain/Config.in"
source "system/Config.in"
source "linux/Config.in"
source "boot/Config.in"
source "fs/Config.in"
source "target/Config.in"
```

Each `source`d file may itself `source` further files, building a tree.

### 2.2 Directory Layout

```
ASCII: Config.in File Tree
═══════════════════════════════════════════════════════════════════
  buildroot/
  ├── Config.in                 ← root entry point
  ├── arch/
  │   ├── Config.in             ← architecture symbols (BR2_ARCH, …)
  │   └── Config.in.x86        ← arch-specific fragments
  ├── package/
  │   ├── Config.in             ← sources all package fragments
  │   ├── busybox/
  │   │   └── Config.in        ← BR2_PACKAGE_BUSYBOX
  │   ├── libcurl/
  │   │   └── Config.in        ← BR2_PACKAGE_LIBCURL
  │   └── …
  ├── toolchain/
  │   └── Config.in
  └── system/
      └── Config.in
═══════════════════════════════════════════════════════════════════
```

---

## 3. The `.config` File

The `.config` file is the persistent serialised form of every Kconfig symbol value.
After `make menuconfig` or `make <board>_defconfig` it is written to the buildroot
root as `.config`.

### 3.1 File Format Rules

```ini
# .config – generated by Buildroot's Kconfig tools
# Lines starting with '#' are comments.
# Disabled boolean/tristate symbols appear as:  # BR2_FOO is not set
# Enabled symbols appear as:                    BR2_FOO=y
# String symbols:                               BR2_LOCALVERSION="v1.2"
# Integer/hex symbols:                          BR2_JLEVEL=4

BR2_x86_64=y
BR2_ARCH="x86_64"
BR2_ENDIAN="LITTLE"
BR2_GCC_TARGET_ARCH="x86-64"
# BR2_GCC_TARGET_ABI is not set
BR2_TOOLCHAIN_BUILDROOT=y
BR2_PACKAGE_BUSYBOX=y
BR2_PACKAGE_LIBCURL=y
# BR2_PACKAGE_LIBCURL_VERBOSE is not set
BR2_JLEVEL=4
BR2_OPTIMIZE_2=y
```

### 3.2 Consuming `.config` in Makefiles

Buildroot's top-level `Makefile` includes the generated `Makefile.br` which exposes
every `BR2_*` symbol as a Make variable:

```makefile
# In any package .mk file you can test:
ifeq ($(BR2_PACKAGE_LIBCURL),y)
  MY_PKG_DEPS += libcurl
endif

ifeq ($(BR2_TOOLCHAIN_HAS_THREADS),y)
  MY_PKG_CONF_OPTS += --enable-threads
endif
```

---

## 4. The `Config.in` File Format

### 4.1 Grammar Overview

```
ASCII: Config.in Grammar Skeleton
═══════════════════════════════════════════════════════════════
  config SYMBOL_NAME
      <type>
      prompt  "Human-readable prompt"
      default <value>  [if <expr>]
      depends on <expr>
      select  <symbol>  [if <expr>]
      imply   <symbol>  [if <expr>]
      range   <min> <max>  [if <expr>]
      help
          Free-form help text.
          Indented. Ends at next non-indented line.

  choice / endchoice
  menu  "Title" / endmenu
  if <expr> / endif
  source "path/to/Config.in"
  comment "Visible comment in menu"
═══════════════════════════════════════════════════════════════
```

### 4.2 A Minimal Real-World Example

```kconfig
# package/myapp/Config.in

config BR2_PACKAGE_MYAPP
    bool "myapp"
    depends on BR2_USE_MMU
    depends on BR2_TOOLCHAIN_HAS_THREADS
    select BR2_PACKAGE_LIBCURL
    select BR2_PACKAGE_ZLIB
    help
      myapp is a demonstration application that fetches data
      over HTTP and decompresses it.

      Project: https://example.com/myapp

if BR2_PACKAGE_MYAPP

config BR2_PACKAGE_MYAPP_TLS
    bool "Enable TLS support"
    depends on BR2_PACKAGE_OPENSSL
    select BR2_PACKAGE_OPENSSL
    help
      Build myapp with OpenSSL-based TLS support.

config BR2_PACKAGE_MYAPP_LOGLEVEL
    int "Default log level (0=off … 5=debug)"
    range 0 5
    default 2
    help
      Sets the compile-time default log verbosity.

config BR2_PACKAGE_MYAPP_INSTALL_DOCS
    bool "Install documentation"
    default n
    help
      Copy manpages and HTML docs to the target rootfs.

endif  # BR2_PACKAGE_MYAPP
```

---

## 5. Symbol Types

| Type       | Kconfig keyword | `.config` form          | Make variable value |
|------------|-----------------|-------------------------|---------------------|
| Boolean    | `bool`          | `=y` or `# … not set`  | `y` or empty        |
| Tristate   | `tristate`      | `=y`, `=m`, or not set  | `y`, `m`, or empty  |
| String     | `string`        | `="some string"`        | `some string`       |
| Integer    | `int`           | `=42`                   | `42`                |
| Hexadecimal| `hex`           | `=0x1000`               | `0x1000`            |

Buildroot almost exclusively uses `bool` and `string`; `int` appears for settings
such as `BR2_JLEVEL`.  Tristate (`=m` for kernel modules) is rare in Buildroot
itself but common when configuring the Linux kernel via `linux/Config.in`.

---

## 6. Dependency Expressions

### 6.1 `depends on`

Prevents a symbol from being set if its dependency evaluates to false. The symbol
disappears from menus entirely when the condition is false.

```kconfig
config BR2_PACKAGE_FOO
    bool "foo"
    depends on BR2_PACKAGE_LIBBAR          # must have libbar
    depends on !BR2_STATIC_LIBS            # must NOT be static-only
    depends on BR2_ARCH_IS_64 || BR2_ARCH_IS_32  # either 32 or 64 bit
```

Multiple `depends on` lines are ANDed together.

### 6.2 `select`

Forcibly sets another symbol to `y` (a *reverse* dependency). Use it for mandatory
library requirements to ensure the dependency is always pulled in.

```kconfig
config BR2_PACKAGE_MYAPP
    bool "myapp"
    select BR2_PACKAGE_ZLIB         # always enable zlib when myapp is on
    select BR2_PACKAGE_LIBCURL if BR2_PACKAGE_MYAPP_HTTP  # conditional select
```

> **Warning:** `select` bypasses `depends on` of the selected symbol. Avoid
> selecting a symbol that has hard constraints the selector may not satisfy —
> this can produce an unsatisfiable configuration.

### 6.3 `imply`

Weaker than `select`: suggests enabling another symbol but allows the user to
override it. Used when a feature is *beneficial* but not *mandatory*.

```kconfig
config BR2_PACKAGE_MYAPP
    bool "myapp"
    imply BR2_PACKAGE_CA_CERTIFICATES  # nice to have, not forced
```

### 6.4 `if … endif` Blocks

Wrap multiple symbols under one common condition:

```kconfig
if BR2_PACKAGE_MYAPP

config BR2_PACKAGE_MYAPP_FEATURE_A
    bool "Feature A"

config BR2_PACKAGE_MYAPP_FEATURE_B
    bool "Feature B"
    depends on BR2_PACKAGE_MYAPP_FEATURE_A  # B requires A

endif
```

### 6.5 Expression Operators

```
ASCII: Operator Precedence (highest → lowest)
════════════════════════════════════════════
  !expr               logical NOT
  expr && expr        logical AND   (same as: expr  expr on separate lines)
  expr || expr        logical OR
  expr = expr         equality
  expr != expr        inequality
════════════════════════════════════════════
```

### 6.6 Complex Dependency Example

```kconfig
config BR2_PACKAGE_COMPLEX
    bool "complex package"
    depends on (BR2_arm || BR2_aarch64)
    depends on BR2_TOOLCHAIN_HAS_THREADS
    depends on !BR2_PREFER_STATIC_LIB
    depends on BR2_PACKAGE_LIBSSL || BR2_PACKAGE_MBEDTLS
    select BR2_PACKAGE_LIBEVENT
    select BR2_PACKAGE_LIBSSL if BR2_PACKAGE_COMPLEX_TLS && !BR2_PACKAGE_MBEDTLS
    help
      Complex package requiring ARM, threads, a TLS library,
      and libevent.
```

```
ASCII: Dependency Graph for BR2_PACKAGE_COMPLEX
════════════════════════════════════════════════════════════════════
  BR2_PACKAGE_COMPLEX
        │
        ├── depends on ─── BR2_arm  ──── OR ──── BR2_aarch64
        │
        ├── depends on ─── BR2_TOOLCHAIN_HAS_THREADS
        │
        ├── depends on ─── NOT BR2_PREFER_STATIC_LIB
        │
        ├── depends on ─── BR2_PACKAGE_LIBSSL ─── OR ─── BR2_PACKAGE_MBEDTLS
        │
        └── select ──────► BR2_PACKAGE_LIBEVENT  (forced ON)
════════════════════════════════════════════════════════════════════
```

---

## 7. menuconfig – The Interactive Interface

### 7.1 Invocation

```bash
make menuconfig          # full Buildroot configuration
make linux-menuconfig    # kernel Config.in
make busybox-menuconfig  # BusyBox Config.in
make uboot-menuconfig    # U-Boot Config.in
```

### 7.2 Screen Layout (ASCII)

```
╔══════════════════════════════════════════════════════════════════════════╗
║           Buildroot 2024.02 Configuration                                ║
╠══════════════════════════════════════════════════════════════════════════╣
║  Arrow keys navigate the menu.  <Enter> selects submenus --->            ║
║  Highlighted letters are hotkeys.  Pressing <Y> includes, <N> excludes, ║
║  <M> modularizes features.  Press <Esc><Esc> to exit, <?> for Help.     ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║      Target options  --->                                                ║
║      Build options  --->                                                 ║
║      Toolchain  --->                                                     ║
║      System configuration  --->                                          ║
║  *** Kernel ***                                                          ║
║      Linux Kernel  --->                                                  ║
║  *** Target packages ***                                                 ║
║      Audio and video applications  --->                                  ║
║      Compressors and decompressors  --->                                 ║
║      Development tools  --->                                             ║
║    [*] myapp  --->                                                       ║
║      Networking applications  --->                                       ║
║                                                                          ║
╠══════════════════════════════════════════════════════════════════════════╣
║          <Select>    < Exit >    < Help >    < Save >    < Load >        ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### 7.3 Key Bindings Reference

```
ASCII: menuconfig Keyboard Map
═══════════════════════════════════════════════════════════════
  ↑ / ↓            Navigate items
  ← / →            Navigate bottom buttons
  Enter            Descend into sub-menu / toggle
  Space            Toggle bool on/off
  Y                Force enable  (=y)
  N                Force disable (not set)
  M                Set tristate to module (=m)
  /                Symbol search dialog
  ?                Help for highlighted symbol
  Esc Esc          Exit current menu / exit program
  S                Save .config
  L                Load .config from file
═══════════════════════════════════════════════════════════════
```

### 7.4 Search Dialog

Press `/` inside menuconfig to open the search dialog:

```
╔══════════════════════════════════════════════╗
║           Symbol Search                      ║
╠══════════════════════════════════════════════╣
║  Search: MYAPP_TLS_                          ║
╠══════════════════════════════════════════════╣
║  Symbol: BR2_PACKAGE_MYAPP_TLS               ║
║  Type  : bool                                ║
║  Value : n                                   ║
║  Location:                                   ║
║    -> Target packages                        ║
║       -> Networking (BR2_PACKAGE_MYAPP [=y]) ║
║          -> myapp TLS support                ║
╚══════════════════════════════════════════════╝
```

---

## 8. How Kconfig Maps to Package Enablement

### 8.1 The Mapping Chain

```
ASCII: Kconfig → Build artefact pipeline
═════════════════════════════════════════════════════════════════════════
  Config.in                 .config                package/*.mk
  ─────────────────         ──────────────────     ────────────────────
  config BR2_PACKAGE_FOO    BR2_PACKAGE_FOO=y  →   ifeq ($(BR2_PACKAGE_FOO),y)
      bool "foo"                                      $(eval $(generic-package))
                                                   endif
                                                       │
                                  ┌────────────────────┘
                                  ▼
                         package/foo/
                         ├── Config.in     ← defines symbol
                         ├── foo.mk        ← build rules
                         └── foo.hash      ← source integrity
═════════════════════════════════════════════════════════════════════════
```

### 8.2 `package/Config.in` Aggregation

Every package directory's `Config.in` is sourced by `package/Config.in`:

```kconfig
# package/Config.in  (excerpt, simplified)
menu "Networking applications"
    source "package/libcurl/Config.in"
    source "package/wget/Config.in"
    source "package/myapp/Config.in"
endmenu
```

Buildroot's `support/scripts/gen-buildroot-tree.py` helps maintain this list, but
manual addition is common for out-of-tree packages.

### 8.3 The `.mk` Package File

```makefile
# package/myapp/myapp.mk

MYAPP_VERSION       = 1.4.2
MYAPP_SITE          = https://example.com/releases
MYAPP_SOURCE        = myapp-$(MYAPP_VERSION).tar.gz
MYAPP_LICENSE       = MIT
MYAPP_LICENSE_FILES = COPYING

MYAPP_DEPENDENCIES  = zlib libcurl

# Map Kconfig bool to autoconf / cmake option
ifeq ($(BR2_PACKAGE_MYAPP_TLS),y)
MYAPP_DEPENDENCIES  += openssl
MYAPP_CONF_OPTS     += -DENABLE_TLS=ON
else
MYAPP_CONF_OPTS     += -DENABLE_TLS=OFF
endif

MYAPP_CONF_OPTS += -DLOG_LEVEL=$(BR2_PACKAGE_MYAPP_LOGLEVEL)

$(eval $(cmake-package))
```

---

## 9. Writing a New Package Config.in

Follow this checklist when adding a new package:

```
ASCII: New Package Config.in Checklist
══════════════════════════════════════════════════════════════
  Step 1 ── Create directory
           package/mypkg/
           ├── Config.in
           ├── mypkg.mk
           └── mypkg.hash

  Step 2 ── Write Config.in (see template below)

  Step 3 ── Add source line to package/Config.in
           (alphabetical within its menu section)

  Step 4 ── Test with:
           make menuconfig      ← verify symbol appears
           make mypkg           ← verify it builds
           make mypkg-dirclean  ← verify clean rebuild

  Step 5 ── Check with:
           utils/check-package package/mypkg/Config.in
           utils/check-package package/mypkg/mypkg.mk
══════════════════════════════════════════════════════════════
```

### 9.1 Config.in Template

```kconfig
################################################################################
#
# mypkg
#
################################################################################

config BR2_PACKAGE_MYPKG
    bool "mypkg"
    # Architecture constraints (if any)
    depends on BR2_USE_MMU
    # Toolchain constraints
    depends on BR2_TOOLCHAIN_HAS_THREADS
    depends on BR2_TOOLCHAIN_HAS_SYNC_4  # needs 32-bit atomics
    # Library dependencies expressed as depends on (not select)
    # when the library is truly optional or chosen by the user:
    # depends on BR2_PACKAGE_LIBFOO
    # Mandatory library deps as select:
    select BR2_PACKAGE_ZLIB
    help
      Short one-line description.

      Longer multi-line description of what the package does,
      who maintains it, and any important notes.

      https://example.com/mypkg

if BR2_PACKAGE_MYPKG

config BR2_PACKAGE_MYPKG_WITH_SSL
    bool "TLS/SSL support"
    depends on BR2_PACKAGE_OPENSSL
    select BR2_PACKAGE_OPENSSL
    help
      Enable TLS support via OpenSSL.

config BR2_PACKAGE_MYPKG_WORKERS
    int "Number of worker threads"
    range 1 64
    default 4
    help
      Controls the default worker thread count compiled in.

endif # BR2_PACKAGE_MYPKG
```

---

## 10. C Code Example – Reading Kconfig Symbols at Runtime

Kconfig symbols are primarily *build-time* constants, but they flow into C source
code through preprocessor macros generated from the Buildroot-configured kernel or
through hand-written wrappers. The example below demonstrates:

- Using `#ifdef` / `#if` guards driven by `BR2_*` or custom Kconfig defines
- A host-side utility that parses `.config` directly

### 10.1 Using Kconfig Macros Inside a Package (C)

```c
/* src/features.h  – generated or hand-maintained from Config.in values */
#ifndef MYAPP_FEATURES_H
#define MYAPP_FEATURES_H

/* CMakeLists.txt passes -DMYAPP_LOG_LEVEL=$(BR2_PACKAGE_MYAPP_LOGLEVEL) */
#ifndef MYAPP_LOG_LEVEL
#  define MYAPP_LOG_LEVEL 2
#endif

/* Passed as -DMYAPP_WITH_TLS when BR2_PACKAGE_MYAPP_TLS=y */
/* #define MYAPP_WITH_TLS */

/* Passed as -DMYAPP_WORKERS=N from BR2_PACKAGE_MYAPP_WORKERS */
#ifndef MYAPP_WORKERS
#  define MYAPP_WORKERS 4
#endif

#endif /* MYAPP_FEATURES_H */
```

```c
/* src/main.c */
#include <stdio.h>
#include <stdlib.h>
#include "features.h"

/* ------------------------------------------------------------------ */
/* Simple logging subsystem conditioned on Kconfig log level           */
/* ------------------------------------------------------------------ */

#define LOG_ERROR   1
#define LOG_WARN    2
#define LOG_INFO    3
#define LOG_DEBUG   4
#define LOG_TRACE   5

#define LOG(level, fmt, ...) \
    do { \
        if ((level) <= MYAPP_LOG_LEVEL) { \
            fprintf((level) <= LOG_WARN ? stderr : stdout, \
                    "[%s] " fmt "\n", log_level_str(level), ##__VA_ARGS__); \
        } \
    } while (0)

static const char *log_level_str(int level)
{
    switch (level) {
    case LOG_ERROR: return "ERROR";
    case LOG_WARN:  return "WARN ";
    case LOG_INFO:  return "INFO ";
    case LOG_DEBUG: return "DEBUG";
    case LOG_TRACE: return "TRACE";
    default:        return "?????";
    }
}

/* ------------------------------------------------------------------ */
/* TLS availability gate                                               */
/* ------------------------------------------------------------------ */

#ifdef MYAPP_WITH_TLS
#  include <openssl/ssl.h>

static void tls_init(void)
{
    SSL_library_init();
    LOG(LOG_INFO, "TLS initialised (OpenSSL %s)",
        OpenSSL_version(OPENSSL_VERSION));
}
#else
static void tls_init(void)
{
    LOG(LOG_WARN, "TLS support not compiled in (BR2_PACKAGE_MYAPP_TLS=n)");
}
#endif /* MYAPP_WITH_TLS */

/* ------------------------------------------------------------------ */
/* Worker pool stub conditioned on MYAPP_WORKERS                       */
/* ------------------------------------------------------------------ */

typedef struct {
    int id;
} Worker;

static Worker workers[MYAPP_WORKERS];

static void workers_init(void)
{
    LOG(LOG_INFO, "Initialising %d worker(s)", MYAPP_WORKERS);
    for (int i = 0; i < MYAPP_WORKERS; i++) {
        workers[i].id = i;
        LOG(LOG_DEBUG, "  worker[%d] ready", i);
    }
}

/* ------------------------------------------------------------------ */
/* Main                                                                */
/* ------------------------------------------------------------------ */

int main(void)
{
    LOG(LOG_INFO, "myapp starting (log-level=%d, workers=%d, tls=%s)",
        MYAPP_LOG_LEVEL,
        MYAPP_WORKERS,
#ifdef MYAPP_WITH_TLS
        "enabled"
#else
        "disabled"
#endif
    );

    tls_init();
    workers_init();

    LOG(LOG_INFO, "myapp ready");
    return EXIT_SUCCESS;
}
```

### 10.2 Host-Side `.config` Parser (C)

A utility that reads Buildroot's `.config` at host build time to extract symbol
values for code generation:

```c
/* tools/kconfig_query.c
 * Usage: kconfig_query .config BR2_PACKAGE_MYAPP_LOGLEVEL
 * Exits 0 and prints value if found; exits 1 if not set.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define MAX_LINE 512

/* Return codes */
#define RC_FOUND     0
#define RC_NOT_SET   1
#define RC_ERROR     2

int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <.config> <SYMBOL>\n", argv[0]);
        return RC_ERROR;
    }

    const char *config_path = argv[1];
    const char *symbol      = argv[2];

    FILE *fp = fopen(config_path, "r");
    if (!fp) {
        fprintf(stderr, "Cannot open '%s': %s\n", config_path, strerror(errno));
        return RC_ERROR;
    }

    /* Build the "not set" sentinel: # SYMBOL is not set */
    char not_set_buf[MAX_LINE];
    snprintf(not_set_buf, sizeof(not_set_buf), "# %s is not set", symbol);

    char line[MAX_LINE];
    int  rc = RC_NOT_SET;  /* assume not found */

    while (fgets(line, sizeof(line), fp)) {
        /* Strip trailing newline */
        size_t len = strlen(line);
        if (len > 0 && line[len - 1] == '\n')
            line[--len] = '\0';

        /* Skip blank lines and pure comments */
        if (len == 0 || (line[0] == '#' && strncmp(line, not_set_buf, len) != 0))
            continue;

        /* Check for "# SYMBOL is not set" */
        if (strcmp(line, not_set_buf) == 0) {
            rc = RC_NOT_SET;
            break;
        }

        /* Check for "SYMBOL=value" */
        size_t sym_len = strlen(symbol);
        if (strncmp(line, symbol, sym_len) == 0 && line[sym_len] == '=') {
            const char *value = line + sym_len + 1;
            /* Strip surrounding quotes for string symbols */
            if (value[0] == '"') {
                size_t vlen = strlen(value);
                if (vlen >= 2 && value[vlen - 1] == '"') {
                    printf("%.*s\n", (int)(vlen - 2), value + 1);
                } else {
                    printf("%s\n", value + 1);
                }
            } else {
                printf("%s\n", value);
            }
            rc = RC_FOUND;
            break;
        }
    }

    fclose(fp);
    return rc;
}
```

---

## 11. C++ Code Example – Kconfig-Driven Feature Flags

This example shows a C++ class hierarchy where Kconfig options select at
compile-time which backend is compiled in, using policy-based design:

```cpp
// include/config/transport.hpp
// CMakeLists.txt passes:
//   -DMYAPP_TRANSPORT_HTTP  when BR2_PACKAGE_MYAPP_HTTP=y
//   -DMYAPP_TRANSPORT_MQTT  when BR2_PACKAGE_MYAPP_MQTT=y
//   -DMYAPP_WITH_TLS        when BR2_PACKAGE_MYAPP_TLS=y

#pragma once
#include <string>
#include <stdexcept>
#include <cstdint>

// -----------------------------------------------------------------------
// Abstract transport interface
// -----------------------------------------------------------------------
class ITransport {
public:
    virtual ~ITransport() = default;
    virtual bool connect(const std::string& host, uint16_t port) = 0;
    virtual bool send(const uint8_t* data, size_t len)           = 0;
    virtual void disconnect()                                     = 0;
    [[nodiscard]] virtual const char* name() const noexcept       = 0;
};

// -----------------------------------------------------------------------
// HTTP transport (compiled in when BR2_PACKAGE_MYAPP_HTTP=y)
// -----------------------------------------------------------------------
#ifdef MYAPP_TRANSPORT_HTTP

#include <cstdio>

class HttpTransport final : public ITransport {
    std::string url_;
    bool        connected_{false};

public:
    bool connect(const std::string& host, uint16_t port) override {
#ifdef MYAPP_WITH_TLS
        url_ = "https://" + host + ":" + std::to_string(port);
#else
        url_ = "http://"  + host + ":" + std::to_string(port);
#endif
        // Stub: real impl would initialise libcurl here
        connected_ = true;
        return true;
    }

    bool send(const uint8_t* data, size_t len) override {
        if (!connected_) return false;
        // Stub: real impl would POST data via libcurl
        (void)data; (void)len;
        return true;
    }

    void disconnect() override { connected_ = false; }

    [[nodiscard]] const char* name() const noexcept override {
#ifdef MYAPP_WITH_TLS
        return "HTTPS";
#else
        return "HTTP";
#endif
    }
};

#endif  // MYAPP_TRANSPORT_HTTP

// -----------------------------------------------------------------------
// MQTT transport (compiled in when BR2_PACKAGE_MYAPP_MQTT=y)
// -----------------------------------------------------------------------
#ifdef MYAPP_TRANSPORT_MQTT

class MqttTransport final : public ITransport {
    bool connected_{false};

public:
    bool connect(const std::string& host, uint16_t port) override {
        // Stub: real impl would connect to MQTT broker
        (void)host; (void)port;
        connected_ = true;
        return true;
    }

    bool send(const uint8_t* data, size_t len) override {
        if (!connected_) return false;
        (void)data; (void)len;
        return true;
    }

    void disconnect() override { connected_ = false; }

    [[nodiscard]] const char* name() const noexcept override { return "MQTT"; }
};

#endif  // MYAPP_TRANSPORT_MQTT

// -----------------------------------------------------------------------
// Factory: returns the compile-time selected transport
// -----------------------------------------------------------------------
#include <memory>

inline std::unique_ptr<ITransport> make_transport()
{
#if defined(MYAPP_TRANSPORT_HTTP)
    return std::make_unique<HttpTransport>();
#elif defined(MYAPP_TRANSPORT_MQTT)
    return std::make_unique<MqttTransport>();
#else
    // Neither transport compiled in – fail at link time rather than
    // silently doing nothing.  The linker will report the missing symbol,
    // pointing the developer back to the Kconfig option.
    static_assert(false,
        "No transport selected. "
        "Enable BR2_PACKAGE_MYAPP_HTTP or BR2_PACKAGE_MYAPP_MQTT "
        "in menuconfig.");
#endif
}
```

```cpp
// src/main.cpp
#include <iostream>
#include <cstdlib>
#include "config/transport.hpp"

int main()
{
    auto transport = make_transport();

    std::cout << "Using transport: " << transport->name() << "\n";

    if (!transport->connect("iot.example.com", 8883)) {
        std::cerr << "Connection failed\n";
        return EXIT_FAILURE;
    }

    const uint8_t payload[] = {0xDE, 0xAD, 0xBE, 0xEF};
    transport->send(payload, sizeof(payload));
    transport->disconnect();

    std::cout << "Done.\n";
    return EXIT_SUCCESS;
}
```

---

## 12. Rust Code Example – Conditional Compilation via build.rs

Buildroot can cross-compile Rust packages using the `cargo-package` infrastructure.
Kconfig values are passed through the environment and processed in `build.rs` to
emit `cargo:rustc-cfg` directives that mirror Kconfig symbols.

### 12.1 The `Config.in` for a Rust package

```kconfig
# package/myapp-rs/Config.in

config BR2_PACKAGE_MYAPP_RS
    bool "myapp-rs (Rust)"
    depends on BR2_PACKAGE_HOST_RUSTC
    depends on BR2_TOOLCHAIN_HAS_THREADS
    select BR2_PACKAGE_ZLIB
    help
      Rust implementation of myapp.

if BR2_PACKAGE_MYAPP_RS

config BR2_PACKAGE_MYAPP_RS_TLS
    bool "TLS support"
    depends on BR2_PACKAGE_OPENSSL
    select BR2_PACKAGE_OPENSSL
    help
      Enable TLS via the openssl crate.

config BR2_PACKAGE_MYAPP_RS_WORKERS
    int "Worker thread count"
    range 1 32
    default 4

config BR2_PACKAGE_MYAPP_RS_TRANSPORT
    string "Default transport backend"
    default "http"
    help
      One of: http, mqtt, unix

endif
```

### 12.2 `myapp-rs.mk` (Buildroot cargo-package)

```makefile
# package/myapp-rs/myapp-rs.mk

MYAPP_RS_VERSION    = 0.3.1
MYAPP_RS_SITE       = $(call github,example,myapp-rs,v$(MYAPP_RS_VERSION))
MYAPP_RS_LICENSE    = MIT
MYAPP_RS_DEPENDENCIES = host-rustc zlib

# Pass Kconfig values to the Cargo build environment
MYAPP_RS_CARGO_ENV = \
    MYAPP_WORKERS=$(BR2_PACKAGE_MYAPP_RS_WORKERS) \
    MYAPP_TRANSPORT=$(BR2_PACKAGE_MYAPP_RS_TRANSPORT)

ifeq ($(BR2_PACKAGE_MYAPP_RS_TLS),y)
MYAPP_RS_CARGO_ENV += MYAPP_WITH_TLS=1
MYAPP_RS_DEPENDENCIES += openssl
endif

$(eval $(cargo-package))
```

### 12.3 `build.rs` – Translating Env Vars to `cfg` Flags

```rust
// build.rs  – runs on the HOST during cross-compilation

use std::env;

fn main() {
    // Re-run if these env vars change (important for incremental builds)
    println!("cargo:rerun-if-env-changed=MYAPP_WITH_TLS");
    println!("cargo:rerun-if-env-changed=MYAPP_WORKERS");
    println!("cargo:rerun-if-env-changed=MYAPP_TRANSPORT");

    // ── TLS flag ──────────────────────────────────────────────────────
    if env::var("MYAPP_WITH_TLS").map(|v| v == "1").unwrap_or(false) {
        println!("cargo:rustc-cfg=feature=\"tls\"");
        println!("cargo:rustc-cfg=myapp_tls");
    }

    // ── Worker count ──────────────────────────────────────────────────
    let workers: u32 = env::var("MYAPP_WORKERS")
        .ok()
        .and_then(|v| v.parse().ok())
        .unwrap_or(4);

    if workers > 16 {
        eprintln!("cargo:warning=MYAPP_WORKERS={workers} is high; consider reducing.");
    }

    // Emit as a compile-time constant accessible via env!()
    println!("cargo:rustc-env=MYAPP_WORKERS={workers}");

    // ── Transport backend ─────────────────────────────────────────────
    let transport = env::var("MYAPP_TRANSPORT").unwrap_or_else(|_| "http".into());
    match transport.as_str() {
        "http"  => println!("cargo:rustc-cfg=myapp_transport_http"),
        "mqtt"  => println!("cargo:rustc-cfg=myapp_transport_mqtt"),
        "unix"  => println!("cargo:rustc-cfg=myapp_transport_unix"),
        other   => {
            eprintln!("cargo:warning=Unknown transport '{other}', defaulting to http");
            println!("cargo:rustc-cfg=myapp_transport_http");
        }
    }
}
```

### 12.4 `src/main.rs` – Using the `cfg` Flags

```rust
// src/main.rs

use std::io;

// Worker count is a compile-time constant baked in by build.rs
const WORKERS: usize = {
    // env! macro reads MYAPP_WORKERS set by cargo:rustc-env
    // Falls back to "4" when not set (e.g. during development builds)
    match option_env!("MYAPP_WORKERS") {
        Some(s) => {
            // const parsing in stable Rust (as of 1.65)
            let bytes = s.as_bytes();
            let mut n: usize = 0;
            let mut i = 0;
            while i < bytes.len() {
                n = n * 10 + (bytes[i] - b'0') as usize;
                i += 1;
            }
            n
        }
        None => 4,
    }
};

// ── Transport trait ───────────────────────────────────────────────────

trait Transport: Send + Sync {
    fn name(&self) -> &'static str;
    fn connect(&mut self, host: &str, port: u16) -> io::Result<()>;
    fn send(&mut self, data: &[u8])               -> io::Result<()>;
    fn disconnect(&mut self);
}

// ── HTTP transport (enabled when myapp_transport_http cfg is set) ─────

#[cfg(myapp_transport_http)]
mod http_transport {
    use super::Transport;
    use std::io;

    pub struct HttpTransport { connected: bool }

    impl HttpTransport {
        pub fn new() -> Self { Self { connected: false } }
    }

    impl Transport for HttpTransport {
        fn name(&self) -> &'static str {
            #[cfg(myapp_tls)] { "HTTPS" }
            #[cfg(not(myapp_tls))] { "HTTP" }
        }

        fn connect(&mut self, host: &str, port: u16) -> io::Result<()> {
            let scheme = if cfg!(myapp_tls) { "https" } else { "http" };
            eprintln!("Connecting to {scheme}://{host}:{port}");
            self.connected = true;
            Ok(())
        }

        fn send(&mut self, data: &[u8]) -> io::Result<()> {
            if !self.connected {
                return Err(io::Error::new(io::ErrorKind::NotConnected, "not connected"));
            }
            eprintln!("Sending {} bytes over HTTP", data.len());
            Ok(())
        }

        fn disconnect(&mut self) { self.connected = false; }
    }
}

// ── MQTT transport ────────────────────────────────────────────────────

#[cfg(myapp_transport_mqtt)]
mod mqtt_transport {
    use super::Transport;
    use std::io;

    pub struct MqttTransport { connected: bool }

    impl MqttTransport {
        pub fn new() -> Self { Self { connected: false } }
    }

    impl Transport for MqttTransport {
        fn name(&self) -> &'static str { "MQTT" }

        fn connect(&mut self, host: &str, port: u16) -> io::Result<()> {
            eprintln!("Connecting to mqtt://{host}:{port}");
            self.connected = true;
            Ok(())
        }

        fn send(&mut self, data: &[u8]) -> io::Result<()> {
            if !self.connected {
                return Err(io::Error::new(io::ErrorKind::NotConnected, "not connected"));
            }
            eprintln!("Publishing {} bytes via MQTT", data.len());
            Ok(())
        }

        fn disconnect(&mut self) { self.connected = false; }
    }
}

// ── Factory ───────────────────────────────────────────────────────────

fn make_transport() -> Box<dyn Transport> {
    #[cfg(myapp_transport_http)]
    { return Box::new(http_transport::HttpTransport::new()); }

    #[cfg(myapp_transport_mqtt)]
    { return Box::new(mqtt_transport::MqttTransport::new()); }

    #[cfg(myapp_transport_unix)]
    compile_error!(
        "Unix socket transport not yet implemented. \
         Set BR2_PACKAGE_MYAPP_RS_TRANSPORT to \"http\" or \"mqtt\"."
    );

    // If no transport cfg is set (shouldn't happen with correct Config.in)
    #[allow(unreachable_code)]
    panic!("No transport compiled in – check Kconfig options");
}

// ── Main ──────────────────────────────────────────────────────────────

fn main() {
    println!(
        "myapp-rs starting: workers={WORKERS}, tls={}",
        if cfg!(myapp_tls) { "yes" } else { "no" }
    );

    let mut t = make_transport();
    println!("Transport: {}", t.name());

    t.connect("iot.example.com", 8883).expect("connect failed");
    t.send(b"\xDE\xAD\xBE\xEF").expect("send failed");
    t.disconnect();

    println!("Done.");
}
```

### 12.5 `Cargo.toml` – Optional Dependencies Matching Kconfig

```toml
[package]
name    = "myapp-rs"
version = "0.3.1"
edition = "2021"

[dependencies]
# Always present
zlib-sys = "0.1"

# Enabled only when myapp_tls cfg is set (set by build.rs)
openssl  = { version = "0.10", optional = true }

[features]
# Mirrors BR2_PACKAGE_MYAPP_RS_TLS; build.rs enables this via rustc-cfg
tls = ["openssl"]
```

---

## 13. Advanced Patterns

### 13.1 Choice Blocks

A `choice` block ensures exactly one option is selected:

```kconfig
choice
    prompt "C library"
    default BR2_TOOLCHAIN_BUILDROOT_GLIBC

config BR2_TOOLCHAIN_BUILDROOT_GLIBC
    bool "glibc"
    depends on BR2_USE_MMU

config BR2_TOOLCHAIN_BUILDROOT_MUSL
    bool "musl"
    depends on BR2_USE_MMU

config BR2_TOOLCHAIN_BUILDROOT_UCLIBC
    bool "uClibc-ng"

endchoice
```

```
ASCII: Choice Block in menuconfig
═══════════════════════════════════════════════════════════════
  C library
   ┌────────────────────────────────────────────┐
   │ (X) glibc                                  │
   │ ( ) musl                                   │
   │ ( ) uClibc-ng                              │
   └────────────────────────────────────────────┘
  Only one selection allowed at a time.
═══════════════════════════════════════════════════════════════
```

### 13.2 `comment` with Visibility

```kconfig
comment "libfoo needs a thread-capable toolchain"
    depends on !BR2_TOOLCHAIN_HAS_THREADS

config BR2_PACKAGE_LIBFOO
    bool "libfoo"
    depends on BR2_TOOLCHAIN_HAS_THREADS
```

The comment appears only when threads are *unavailable*, guiding the user on why
`libfoo` is greyed out.

### 13.3 Arch-Specific Fragments

```kconfig
# arch/Config.in.arm
config BR2_ARM_CPU_HAS_NEON
    bool
    default y
    depends on BR2_cortex_a7 || BR2_cortex_a9 || BR2_cortex_a15 \
               || BR2_cortex_a53 || BR2_cortex_a72

# package/mylib/Config.in
config BR2_PACKAGE_MYLIB
    bool "mylib"
    depends on !BR2_ARM_CPU_HAS_NEON || BR2_PACKAGE_MYLIB_NO_NEON
    # Only offer NEON-less build if NEON not present, or user opts out
```

### 13.4 Versioned Symbol Selection

Useful when a package requires a minimum library version:

```kconfig
config BR2_PACKAGE_MYAPP
    bool "myapp"
    # Requires libssl >= 3.0
    depends on BR2_PACKAGE_OPENSSL
    depends on BR2_PACKAGE_OPENSSL_3
    select BR2_PACKAGE_OPENSSL
```

### 13.5 `allnoconfig` and `allyesconfig` for CI

```bash
# Build a minimal configuration (all symbols =n by default)
make allnoconfig
# Build with everything enabled (stress-test dependency resolution)
make allyesconfig
# Check that a specific defconfig still builds
make raspberrypi4_defconfig && make
```

---

## 14. Debugging & Pitfalls

### 14.1 Common Mistakes

```
ASCII: Pitfall Map
═══════════════════════════════════════════════════════════════════════════
  PITFALL                     SYMPTOM                   FIX
  ────────────────────────────────────────────────────────────────────────
  select on a symbol          Config inconsistency /    Use depends on for
  that has depends on         weird forced enables      optional libs; only
                                                        select truly mandatory
                                                        unconditional symbols.

  Forgetting source in        Symbol invisible in       Add source line in
  package/Config.in           menuconfig                package/Config.in

  Wrong symbol prefix         BR2_ prefix missing;      Always prefix with
                              Make variable not set     BR2_PACKAGE_

  String default with         Makefile sees empty       Always quote defaults:
  no quotes                   value                     default "myvalue"

  depends on ordering         Option visible when it    Place the strictest
  wrong                       shouldn't be              depends on first

  Circular select             infinite recursion in     Audit select chains;
  chains                      Kconfig processing        use depends on instead
═══════════════════════════════════════════════════════════════════════════
```

### 14.2 Verifying Dependency Evaluation

```bash
# Dump all symbols and their computed values
grep BR2_PACKAGE_MYAPP .config

# Use the Kconfig 'conf' tool to query a symbol without menuconfig
./scripts/kconfig/conf --oldconfig Config.in   # re-processes without UI

# Use support/scripts/check-package
support/scripts/check-package package/myapp/Config.in
support/scripts/check-package package/myapp/myapp.mk
```

### 14.3 `.config` Fragments (BR2_EXTERNAL)

For out-of-tree packages, use `BR2_EXTERNAL` together with a Kconfig fragment:

```bash
# configs/myboard_defconfig  (fragment)
BR2_x86_64=y
BR2_TOOLCHAIN_BUILDROOT=y
BR2_PACKAGE_MYAPP=y
BR2_PACKAGE_MYAPP_TLS=y
BR2_PACKAGE_MYAPP_LOGLEVEL=3

# Merge fragment into a full .config
make BR2_EXTERNAL=/path/to/my-external myboard_defconfig
```

---

## 15. Summary

```
ASCII: Kconfig System – Full Overview
═══════════════════════════════════════════════════════════════════════════════
                                                                               
   ┌─────────────────── Config.in tree ───────────────────────────────────┐   
   │                                                                       │   
   │  config BR2_PACKAGE_FOO     ← bool / string / int / hex              │   
   │      bool "foo"             ← type + prompt                           │   
   │      depends on A && !B     ← visibility / availability gate         │   
   │      select LIB_C           ← force-enables a dependency             │   
   │      imply LIB_D            ← soft suggestion                        │   
   │      default y if E         ← conditional default value              │   
   │      range 1 32             ← int/hex bounds                         │   
   │      help …                 ← shown in menuconfig '?' screen         │   
   │                                                                       │   
   │  menu / endmenu  ←─ visual grouping only (no symbol)                 │   
   │  if / endif      ←─ shared visibility condition                      │   
   │  choice          ←─ mutually exclusive radio group                   │   
   │  comment         ←─ informational label (conditional)                │   
   │  source          ←─ include another Config.in file                   │   
   └──────────────────────────────────────────────────────────────────────┘   
                │                                                              
                │ mconf / conf                                                 
                ▼                                                              
   ┌──────────────────────────────────┐                                        
   │           .config                │                                        
   │  BR2_PACKAGE_FOO=y               │                                        
   │  # BR2_PACKAGE_BAR is not set    │                                        
   │  BR2_VERSION="2024.02"           │                                        
   └──────────────┬───────────────────┘                                        
                  │                                                             
       ┌──────────┴──────────┐                                                 
       ▼                     ▼                                                  
  ┌──────────┐       ┌─────────────────────────────────────────┐               
  │ Makefile │       │   C/C++/Rust source                      │               
  │ ifeq(…)  │       │   #ifdef / cfg!() / build.rs             │               
  │ .mk deps │       │   ← compile-time feature gates           │               
  └──────────┘       └─────────────────────────────────────────┘               
═══════════════════════════════════════════════════════════════════════════════
```

The Kconfig system in Buildroot is a **declarative configuration language** that:

- **Describes** the entire option space in `Config.in` files using typed symbols,
  default values, range constraints, and human-readable prompts.
- **Expresses dependencies** through `depends on` (visibility), `select` (forced
  reverse dependencies), and `imply` (soft suggestions), building a DAG that
  Kconfig solves before exposing options to the user.
- **Serialises** the resolved configuration into `.config`, a flat key-value file
  that Make and Cargo/CMake consume directly.
- **Drives package enablement** one-to-one: `BR2_PACKAGE_FOO=y` in `.config`
  causes `foo.mk` to be evaluated, its dependencies added to the build graph, and
  its source fetched, compiled, and installed.

In C and C++, Kconfig values arrive as preprocessor macros (`-DFOO=1`) passed by
CMake or Autoconf wrappers, allowing fine-grained `#ifdef`/`if constexpr` feature
gates. In Rust, a `build.rs` script reads the same environment variables injected
by the `.mk` file and translates them into `cargo:rustc-cfg` directives, enabling
idiomatic `#[cfg(…)]` conditional compilation with zero runtime overhead.

Mastering the Kconfig dependency algebra — particularly the distinction between
`depends on`, `select`, and `imply` — is essential to writing `Config.in` files
that are correct, user-friendly, and safe against unsatisfiable configurations.

---

*End of 02 – Kconfig System & menuconfig*