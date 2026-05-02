Now let me create the detailed markdown document:Here's the complete Module 06 document. It covers:

**Structure & Anatomy** — How every package maps to a `Config.in` + `.mk` pair, with a full variable reference table and the Kconfig constructs used for menu integration.

**Three source strategies** — Versioned tarballs with hash files, Git-pinned packages (with a warning about mutable branch names), and the `BR2_OVERRIDE_SRCDIR` rsync workflow for live development.

**Three language examples**, each with working source code and both Buildroot files:
- **C** — raw `generic-package` with a custom Makefile, respecting `TARGET_CONFIGURE_OPTS` and `CROSS_COMPILE`
- **C++** — `cmake-package` with a C++17 project and `CONF_OPTS`
- **Rust** — `cargo-package` with `CARGO_ENV` overrides and the `HOST_RUSTC` dependency guard

**Build hooks** — The full `POST_EXTRACT → POST_BUILD → INSTALL_INIT_SYSV/SYSTEMD` chain with code examples.

**Debugging section** — `make myapp-show-vars`, `V=1`, `dirclean`, `readelf` checks for correct target architecture.

All diagrams are ASCII/box-drawing character based as requested.

# 06. Writing Custom Packages from Scratch

> **Buildroot Series — Module 06**
> Full anatomy of a `.mk` + `Config.in` pair, local source packages (`BR2_OVERRIDE_SRCDIR`),
> versioned tarballs, and Git-sourced packages — with C, C++, and Rust examples.

---

## Table of Contents

1. [What Is a Buildroot Package?](#1-what-is-a-buildroot-package)
2. [Directory Layout and Anatomy](#2-directory-layout-and-anatomy)
3. [The `Config.in` File — Menu Integration](#3-the-configin-file--menu-integration)
4. [The `.mk` Makefile — Full Anatomy](#4-the-mk-makefile--full-anatomy)
5. [Package Source Types](#5-package-source-types)
   - 5.1 [Versioned Tarball Packages](#51-versioned-tarball-packages)
   - 5.2 [Git-Sourced Packages](#52-git-sourced-packages)
   - 5.3 [Local Source with `BR2_OVERRIDE_SRCDIR`](#53-local-source-with-br2_override_srcdir)
6. [C Package Example — `hello_c`](#6-c-package-example--hello_c)
7. [C++ Package Example — `hello_cpp`](#7-c-package-example--hello_cpp)
8. [Rust Package Example — `hello_rust`](#8-rust-package-example--hello_rust)
9. [Package Variables Reference](#9-package-variables-reference)
10. [Build Hooks and Post-Install Actions](#10-build-hooks-and-post-install-actions)
11. [Debugging Custom Packages](#11-debugging-custom-packages)
12. [Summary](#12-summary)

---

## 1. What Is a Buildroot Package?

A **Buildroot package** is a self-contained unit of build instructions that tells Buildroot
how to fetch, configure, compile, and install a piece of software into the target root
filesystem. Every package lives in `package/<name>/` and consists of at minimum two files:

```
package/
└── mypackage/
    ├── Config.in       ← Kconfig menu entry (user-visible)
    └── mypackage.mk    ← Build logic (fetch, configure, compile, install)
```

Buildroot provides a set of **package infrastructure macros** — `generic-package`,
`autotools-package`, `cmake-package`, `cargo-package`, etc. — that reduce boilerplate to
a handful of variable assignments.

```
  ┌─────────────────────────────────────────────────────────┐
  │                  Buildroot Package Flow                  │
  │                                                         │
  │  Config.in ──► menuconfig ──► .config                   │
  │                                   │                     │
  │                                   ▼                     │
  │  mypackage.mk ──► fetch ──► extract ──► patch           │
  │                                              │           │
  │                                              ▼           │
  │                                         configure        │
  │                                              │           │
  │                                              ▼           │
  │                                           build          │
  │                                              │           │
  │                                              ▼           │
  │                                    install to staging    │
  │                                              │           │
  │                                              ▼           │
  │                                    install to target     │
  └─────────────────────────────────────────────────────────┘
```

---

## 2. Directory Layout and Anatomy

A typical custom package directory with patches and a hash file looks like this:

```
package/
└── myapp/
    ├── Config.in           ← Kconfig entry
    ├── myapp.mk            ← Build rules
    ├── myapp.hash          ← SHA256 / SHA512 checksums for sources
    ├── 0001-fix-build.patch ← Optional patches applied after extraction
    └── S50myapp            ← Optional init script (SysV/OpenRC style)
```

The package name must be **lowercase** with only letters, digits, and hyphens.
The `.mk` filename must match the directory name exactly.

---

## 3. The `Config.in` File — Menu Integration

`Config.in` uses **Kconfig** syntax. It is sourced by the top-level
`package/Config.in` (or your external tree's equivalent) and makes the package
appear in `make menuconfig`.

```kconfig
config BR2_PACKAGE_MYAPP
    bool "myapp"
    depends on BR2_TOOLCHAIN_HAS_THREADS
    select BR2_PACKAGE_LIBCURL
    help
      A sample application demonstrating custom Buildroot packages.

      This option enables the myapp binary, which is installed
      to /usr/bin/myapp on the target filesystem.

      https://example.com/myapp
```

Key Kconfig constructs used in packages:

| Construct        | Purpose                                                      |
|------------------|--------------------------------------------------------------|
| `bool`           | Simple on/off toggle for the package                         |
| `depends on`     | Disable option when dependency is not met                    |
| `select`         | Automatically enable another config symbol                   |
| `default y/n`    | Set a default value                                          |
| `if … endif`     | Conditional block (useful for sub-options)                   |
| `choice … endchoice` | Mutually exclusive group of options                    |
| `string`         | Free-form string option (e.g., for paths or names)           |

The symbol name **must** follow the pattern `BR2_PACKAGE_<UPPERCASE_PKGNAME>`.

---

## 4. The `.mk` Makefile — Full Anatomy

This is the heart of every package. Below is a fully annotated generic template:

```makefile
################################################################################
#
# myapp
#
################################################################################

# 1. Version and source location
MYAPP_VERSION = 1.2.3
MYAPP_SITE    = https://example.com/releases
MYAPP_SOURCE  = myapp-$(MYAPP_VERSION).tar.gz

# 2. Checksums file (optional but recommended)
# Checked against myapp.hash automatically

# 3. License information (mandatory for legal-info target)
MYAPP_LICENSE           = MIT
MYAPP_LICENSE_FILES     = LICENSE

# 4. Dependencies — other Buildroot packages
MYAPP_DEPENDENCIES = libcurl zlib

# 5. Configure options passed to build system
MYAPP_CONF_OPTS = \
    -DENABLE_FEATURE_X=ON \
    -DBUILD_TESTS=OFF

# 6. Build and install hooks (optional)
define MYAPP_INSTALL_INIT_SYSV
    $(INSTALL) -m 0755 -D $(MYAPP_PKGDIR)/S50myapp \
        $(TARGET_DIR)/etc/init.d/S50myapp
endef

# 7. Infrastructure macro — must be the last line
$(eval $(cmake-package))
```

The available **infrastructure macros** are:

```
  ┌──────────────────────────────────────────────────────────┐
  │           Package Infrastructure Macros                  │
  │                                                          │
  │  generic-package     ── raw Makefile / custom build      │
  │  autotools-package   ── ./configure && make              │
  │  cmake-package       ── CMake projects                   │
  │  meson-package       ── Meson build system               │
  │  python-package      ── Python setuptools/pip            │
  │  cargo-package       ── Rust/Cargo projects              │
  │  go-package          ── Go modules                       │
  │  kernel-module       ── Linux kernel out-of-tree modules │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Package Source Types

### 5.1 Versioned Tarball Packages

The most common and reproducible form. Buildroot downloads, verifies, and caches the
tarball in `$(BR2_DL_DIR)` (default: `$(TOPDIR)/dl`).

```makefile
MYAPP_VERSION = 2.0.1
MYAPP_SITE    = https://github.com/example/myapp/releases/download/v$(MYAPP_VERSION)
MYAPP_SOURCE  = myapp-$(MYAPP_VERSION).tar.xz
```

The corresponding **hash file** `myapp.hash` lists checksums:

```
# Compute with: sha256sum myapp-2.0.1.tar.xz
sha256  a1b2c3d4e5f6...  myapp-2.0.1.tar.xz
```

Buildroot checks the hash before extracting. A mismatch aborts the build.

### 5.2 Git-Sourced Packages

Point `MYAPP_SITE` at a Git repository and set `MYAPP_SITE_METHOD = git`:

```makefile
MYAPP_VERSION  = abc1234def5678  # a full commit SHA is reproducible
MYAPP_SITE     = https://github.com/example/myapp.git
MYAPP_SITE_METHOD = git

# Optionally check out a specific branch or tag
# MYAPP_GIT_SUBMODULES = YES   # clone submodules too
```

> **Tip:** Always pin to a full commit SHA, never a branch name, for reproducible builds.
> Branch-based builds break when upstream advances.

Hash files for Git packages contain the commit hash instead of a tarball checksum:

```
# Verify with: git -C dl/myapp/git rev-parse HEAD
sha1  abc1234def567890abc1234def567890abc12345  myapp
```

### 5.3 Local Source with `BR2_OVERRIDE_SRCDIR`

During active development you want Buildroot to build from your local working tree
**without** fetching or archiving. Use the override mechanism:

```bash
# On the command line — highest priority
make MYAPP_OVERRIDE_SRCDIR=/home/dev/myapp

# Or persistently in a local.mk file (git-ignored)
echo "MYAPP_OVERRIDE_SRCDIR = /home/dev/myapp" >> local.mk
```

```
  ┌──────────────────────────────────────────────────────────┐
  │         BR2_OVERRIDE_SRCDIR Workflow                     │
  │                                                          │
  │  /home/dev/myapp/          ← your editor / git repo      │
  │         │                                                │
  │         │  rsync (Buildroot copies changed files only)   │
  │         ▼                                                │
  │  output/build/myapp-custom/  ← Buildroot build dir      │
  │         │                                                │
  │         │  configure / make / install                    │
  │         ▼                                                │
  │  output/target/usr/bin/myapp  ← target rootfs           │
  └──────────────────────────────────────────────────────────┘
```

When an override is active, `MYAPP_VERSION` is set to `custom` automatically.
To rebuild after source changes:

```bash
make myapp-rebuild      # re-compile and re-install
make myapp-dirclean     # wipe the build dir, then rebuild from scratch next time
```

---

## 6. C Package Example — `hello_c`

### Source tree under override

```
/home/dev/hello_c/
├── Makefile
└── src/
    └── main.c
```

**`src/main.c`:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define VERSION "1.0.0"
#define BANNER_WIDTH 42

static void print_banner(const char *msg)
{
    int len = (int)strlen(msg);
    int pad = (BANNER_WIDTH - len - 2) / 2;

    /* Top border */
    printf("+");
    for (int i = 0; i < BANNER_WIDTH; i++) putchar('-');
    printf("+\n");

    /* Padded message line */
    printf("|%*s%s%*s|\n",
           pad, "",
           msg,
           BANNER_WIDTH - len - pad, "");

    /* Bottom border */
    printf("+");
    for (int i = 0; i < BANNER_WIDTH; i++) putchar('-');
    printf("+\n");
}

static void draw_target_board(void)
{
    printf("\n");
    printf("  +----------------------------------+\n");
    printf("  |  Embedded Target Board           |\n");
    printf("  |  +---------+  +---------------+  |\n");
    printf("  |  |  SoC    |  |  Flash 128MB  |  |\n");
    printf("  |  |  ARM    |  +---------------+  |\n");
    printf("  |  |  v8     |                     |\n");
    printf("  |  +---------+  +---------------+  |\n");
    printf("  |               |   RAM 512MB   |  |\n");
    printf("  |               +---------------+  |\n");
    printf("  |  [ETH] [USB] [UART] [I2C]        |\n");
    printf("  +----------------------------------+\n");
    printf("\n");
}

int main(int argc, char *argv[])
{
    print_banner("hello_c v" VERSION " — Buildroot Custom Package");
    draw_target_board();

    printf("  Arguments received: %d\n", argc - 1);
    for (int i = 1; i < argc; i++)
        printf("    [%d] %s\n", i, argv[i]);

    return EXIT_SUCCESS;
}
```

**`Makefile`:**

```makefile
CC      ?= $(CROSS_COMPILE)gcc
CFLAGS  ?= -Wall -Wextra -O2
PREFIX  ?= /usr

SRCS = src/main.c
TARGET = hello_c

all: $(TARGET)

$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) -o $@ $^

install:
	install -D -m 0755 $(TARGET) $(DESTDIR)$(PREFIX)/bin/$(TARGET)

clean:
	rm -f $(TARGET)

.PHONY: all install clean
```

### Buildroot package files

**`package/hello_c/Config.in`:**

```kconfig
config BR2_PACKAGE_HELLO_C
    bool "hello_c"
    help
      A minimal C application demonstrating a custom Buildroot
      package built from local source via BR2_OVERRIDE_SRCDIR.

      Installed to /usr/bin/hello_c on the target.
```

**`package/hello_c/hello_c.mk`:**

```makefile
################################################################################
#
# hello_c
#
################################################################################

HELLO_C_VERSION = 1.0.0
HELLO_C_SITE    = $(TOPDIR)/../hello_c   # fallback when no override
HELLO_C_SITE_METHOD = local

HELLO_C_LICENSE = MIT

define HELLO_C_BUILD_CMDS
    $(MAKE) $(TARGET_CONFIGURE_OPTS) -C $(@D) all
endef

define HELLO_C_INSTALL_TARGET_CMDS
    $(MAKE) $(TARGET_CONFIGURE_OPTS) -C $(@D) install \
        DESTDIR=$(TARGET_DIR) PREFIX=/usr
endef

$(eval $(generic-package))
```

Hook into the package from `package/Config.in`:

```kconfig
source "package/hello_c/Config.in"
```

Build with override:

```bash
make HELLO_C_OVERRIDE_SRCDIR=/home/dev/hello_c hello_c
```

---

## 7. C++ Package Example — `hello_cpp`

### Source tree

```
/home/dev/hello_cpp/
├── CMakeLists.txt
└── src/
    └── main.cpp
```

**`src/main.cpp`:**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <iomanip>

constexpr const char* VERSION = "1.0.0";
constexpr int BOX_W = 50;

// ─── ASCII box drawing helper ─────────────────────────────
class AsciiBox {
public:
    explicit AsciiBox(int width) : w_(width) {}

    void top()    const { border('┌', '─', '┐'); }
    void bottom() const { border('└', '─', '┘'); }

    void line(const std::string& text) const {
        int pad  = static_cast<int>(w_ - text.size() - 2);
        int lpad = pad / 2;
        int rpad = pad - lpad;
        std::cout << "│"
                  << std::string(lpad, ' ')
                  << text
                  << std::string(rpad, ' ')
                  << "│\n";
    }

    void separator() const {
        std::cout << "├" << std::string(w_ - 2, '─') << "┤\n";
    }

private:
    int w_;

    void border(char l, char mid, char r) const {
        std::cout << l << std::string(w_ - 2, mid) << r << "\n";
    }
};

// ─── Buildroot pipeline diagram ───────────────────────────
void draw_pipeline()
{
    std::cout << "\n";
    std::cout << "  Source  ──►  Fetch  ──►  Patch  ──►  Build\n";
    std::cout << "    │                                     │\n";
    std::cout << "    │   Git / Tarball / Local              │\n";
    std::cout << "    └─────────────────────────────────────┘\n";
    std::cout << "                                     │\n";
    std::cout << "                                     ▼\n";
    std::cout << "                              staging/ + target/\n\n";
}

// ─── Main ─────────────────────────────────────────────────
int main(int argc, char* argv[])
{
    AsciiBox box(BOX_W);
    box.top();
    box.line("hello_cpp v" + std::string(VERSION));
    box.separator();
    box.line("Buildroot Custom C++ Package Demo");
    box.bottom();

    draw_pipeline();

    std::vector<std::string> args(argv + 1, argv + argc);
    if (!args.empty()) {
        std::cout << "  Command-line arguments:\n";
        for (const auto& a : args)
            std::cout << "    → " << a << "\n";
    }

    return 0;
}
```

**`CMakeLists.txt`:**

```cmake
cmake_minimum_required(VERSION 3.10)
project(hello_cpp CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(hello_cpp src/main.cpp)

install(TARGETS hello_cpp
        RUNTIME DESTINATION bin)
```

### Buildroot package files

**`package/hello_cpp/Config.in`:**

```kconfig
config BR2_PACKAGE_HELLO_CPP
    bool "hello_cpp"
    depends on BR2_INSTALL_LIBSTDCPP
    help
      A C++17 application demonstrating a CMake-based custom
      Buildroot package.

      Requires the C++ standard library on the target.
      Installed to /usr/bin/hello_cpp.
```

**`package/hello_cpp/hello_cpp.mk`:**

```makefile
################################################################################
#
# hello_cpp
#
################################################################################

HELLO_CPP_VERSION = 1.0.0
HELLO_CPP_SITE    = $(TOPDIR)/../hello_cpp
HELLO_CPP_SITE_METHOD = local

HELLO_CPP_LICENSE = MIT

HELLO_CPP_CONF_OPTS = \
    -DCMAKE_BUILD_TYPE=Release

$(eval $(cmake-package))
```

Build with override:

```bash
make HELLO_CPP_OVERRIDE_SRCDIR=/home/dev/hello_cpp hello_cpp
```

---

## 8. Rust Package Example — `hello_rust`

Buildroot supports Rust via the `cargo-package` infrastructure. The toolchain must have
`BR2_PACKAGE_HOST_RUSTC` enabled, and the target must support Rust (most ARM, AArch64,
x86 targets do).

### Source tree

```
/home/dev/hello_rust/
├── Cargo.toml
└── src/
    └── main.rs
```

**`Cargo.toml`:**

```toml
[package]
name    = "hello_rust"
version = "1.0.0"
edition = "2021"

[dependencies]
# No external crates — keeps the example self-contained
```

**`src/main.rs`:**

```rust
//! hello_rust — Buildroot Cargo package demo

const VERSION: &str = env!("CARGO_PKG_VERSION");
const BOX_WIDTH: usize = 52;

/// Draw a simple ASCII banner box
fn banner(title: &str) {
    let pad  = BOX_WIDTH.saturating_sub(title.len() + 2);
    let lpad = pad / 2;
    let rpad = pad - lpad;

    println!("+{}+", "-".repeat(BOX_WIDTH));
    println!(
        "|{}{}{}" ,
        " ".repeat(lpad),
        title,
        " ".repeat(rpad),
    );
    // Close the right edge
    println!();
    println!("+{}+", "-".repeat(BOX_WIDTH));
}

/// ASCII diagram of a cross-compilation pipeline
fn draw_cross_compile() {
    let lines = [
        "",
        "  ┌──────────────────────────────────────────┐",
        "  │         Cross-Compilation Pipeline        │",
        "  ├──────────────────────────────────────────┤",
        "  │  Host x86_64                             │",
        "  │  ┌─────────┐   rustc --target            │",
        "  │  │ Cargo   │──────────────────────────►  │",
        "  │  │ build   │   aarch64-linux-musl         │",
        "  │  └─────────┘                             │",
        "  │       │                                   │",
        "  │       ▼                                   │",
        "  │  hello_rust  (ELF ARM64 static binary)   │",
        "  │       │                                   │",
        "  │       ▼                                   │",
        "  │  output/target/usr/bin/hello_rust        │",
        "  └──────────────────────────────────────────┘",
        "",
    ];
    for line in &lines {
        println!("{}", line);
    }
}

fn main() {
    banner(&format!("hello_rust v{VERSION} — Buildroot Cargo Package"));
    draw_cross_compile();

    let args: Vec<String> = std::env::args().skip(1).collect();
    if args.is_empty() {
        println!("  No arguments. Pass some to see them listed.");
    } else {
        println!("  Arguments:");
        for (i, arg) in args.iter().enumerate() {
            println!("    [{i}] {arg}");
        }
    }
}
```

### Buildroot package files

**`package/hello_rust/Config.in`:**

```kconfig
config BR2_PACKAGE_HELLO_RUST
    bool "hello_rust"
    depends on BR2_PACKAGE_HOST_RUSTC
    depends on !BR2_STATIC_LIBS || BR2_PACKAGE_HOST_RUSTC_ARCH_SUPPORTS_STD
    help
      A minimal Rust application showing how to integrate
      a Cargo project as a Buildroot package.

      Requires the Rust toolchain (BR2_PACKAGE_HOST_RUSTC).
      Installed to /usr/bin/hello_rust on the target.
```

**`package/hello_rust/hello_rust.mk`:**

```makefile
################################################################################
#
# hello_rust
#
################################################################################

HELLO_RUST_VERSION = 1.0.0
HELLO_RUST_SITE    = $(TOPDIR)/../hello_rust
HELLO_RUST_SITE_METHOD = local

HELLO_RUST_LICENSE = MIT

HELLO_RUST_CARGO_ENV = \
    CARGO_PROFILE_RELEASE_LTO=true \
    CARGO_PROFILE_RELEASE_PANIC=abort

$(eval $(cargo-package))
```

The `cargo-package` infrastructure automatically:
- Cross-compiles with the correct `--target` triple
- Strips the binary if `BR2_STRIP_strip` is set
- Installs the binary to `$(TARGET_DIR)/usr/bin/`

Build with override:

```bash
make HELLO_RUST_OVERRIDE_SRCDIR=/home/dev/hello_rust hello_rust
```

---

## 9. Package Variables Reference

The table below lists the most important variables available in any `.mk` file.

| Variable                    | Meaning                                                  |
|-----------------------------|----------------------------------------------------------|
| `PKGNAME_VERSION`           | Version string used in tarball name and build dir        |
| `PKGNAME_SITE`              | URL or path to source                                    |
| `PKGNAME_SITE_METHOD`       | `wget`, `git`, `svn`, `local`, `scp`, etc.              |
| `PKGNAME_SOURCE`            | Tarball filename (default: `$(NAME)-$(VERSION).tar.gz`)  |
| `PKGNAME_LICENSE`           | SPDX license identifier(s)                               |
| `PKGNAME_LICENSE_FILES`     | File(s) in source containing the license text            |
| `PKGNAME_DEPENDENCIES`      | Space-separated list of target dependencies              |
| `PKGNAME_CONF_OPTS`         | Options passed to `cmake` / `./configure` / `meson`      |
| `PKGNAME_MAKE_OPTS`         | Extra flags passed to `$(MAKE)` during build             |
| `PKGNAME_INSTALL_OPTS`      | Extra flags passed to install step                       |
| `PKGNAME_INSTALL_TARGET`    | `YES` (default) — install into `$(TARGET_DIR)`           |
| `PKGNAME_INSTALL_STAGING`   | `YES` — install into `$(STAGING_DIR)` (libraries)        |
| `PKGNAME_GIT_SUBMODULES`    | `YES` — clone submodules for Git sources                 |
| `TARGET_CONFIGURE_OPTS`     | `CC`, `CXX`, `AR`, `CFLAGS`, `LDFLAGS` for cross-compile|
| `$(TARGET_DIR)`             | Root of the target filesystem being assembled            |
| `$(STAGING_DIR)`            | Sysroot for cross-compilation headers and libraries      |
| `$(HOST_DIR)`               | Tools installed on the build host                        |
| `$(PKGNAME_PKGDIR)`         | The `package/pkgname/` source directory (for scripts)    |

---

## 10. Build Hooks and Post-Install Actions

Buildroot provides hook points where you can inject custom shell commands:

```makefile
# Run after source extraction (e.g., generate configure scripts)
define MYAPP_POST_EXTRACT_CMDS
    cd $(@D) && autoreconf -fi
endef

# Run after configure (e.g., patch generated files)
define MYAPP_POST_CONFIGURE_CMDS
    sed -i 's/old/new/' $(@D)/generated.h
endef

# Run after build (e.g., strip a secondary binary)
define MYAPP_POST_BUILD_CMDS
    $(TARGET_STRIP) $(@D)/tools/helper
endef

# Install an init script
define MYAPP_INSTALL_INIT_SYSV
    $(INSTALL) -m 0755 -D $(MYAPP_PKGDIR)/S50myapp \
        $(TARGET_DIR)/etc/init.d/S50myapp
endef

# Install a systemd unit
define MYAPP_INSTALL_INIT_SYSTEMD
    $(INSTALL) -m 0644 -D $(MYAPP_PKGDIR)/myapp.service \
        $(TARGET_DIR)/usr/lib/systemd/system/myapp.service
endef
```

Hook naming follows the pattern: `PKGNAME_<STAGE>_CMDS` or `PKGNAME_INSTALL_INIT_<STYLE>`.

```
  ┌──────────────────────────────────────────────────────────┐
  │                   Hook Execution Order                   │
  │                                                          │
  │  POST_EXTRACT ──► POST_RSYNC ──► PRE_CONFIGURE          │
  │                                       │                  │
  │                               POST_CONFIGURE             │
  │                                       │                  │
  │                                POST_BUILD                │
  │                                       │                  │
  │                    POST_INSTALL_STAGING │ POST_INSTALL_TARGET
  │                                       │                  │
  │                              INSTALL_INIT_SYSV           │
  │                              INSTALL_INIT_SYSTEMD        │
  └──────────────────────────────────────────────────────────┘
```

---

## 11. Debugging Custom Packages

### Useful make targets

```bash
# Show all variables for a package
make myapp-show-vars

# Run only one build step
make myapp-extract
make myapp-configure
make myapp-build

# Re-run install without rebuilding
make myapp-reinstall

# Clean just this package (keeps dl/ cache)
make myapp-dirclean

# Full verbose build output
make myapp V=1

# Show what files a package installs
make myapp-show-info
```

### Inspecting the build directory

```bash
ls output/build/myapp-1.0.0/
# After configure step, inspect generated Makefile or build.ninja

# Check the cross-compile environment used
cat output/build/myapp-1.0.0/.br-env
```

### Checking the target rootfs

```bash
# List installed files
find output/target/usr -name "*myapp*"

# Inspect ELF headers for correct architecture
file output/target/usr/bin/myapp
readelf -h output/target/usr/bin/myapp

# Check dynamic dependencies (should only reference target libs)
$(CROSS_COMPILE)readelf -d output/target/usr/bin/myapp | grep NEEDED
```

---

## 12. Summary

Writing a custom Buildroot package boils down to two files and the right infrastructure macro:

```
  ┌──────────────────────────────────────────────────────────┐
  │              Custom Package — Quick Reference            │
  │                                                          │
  │  package/myapp/                                          │
  │  ├── Config.in   ── Kconfig bool + depends + help        │
  │  └── myapp.mk    ── PKGNAME_VERSION / SITE / LICENSE     │
  │                     + infrastructure macro call          │
  │                                                          │
  │  Source Types:                                           │
  │  ┌────────────┬────────────────────────────────────┐    │
  │  │ Tarball    │ SITE=https://...  METHOD=wget       │    │
  │  │ Git        │ SITE=https://git  METHOD=git        │    │
  │  │ Local/Dev  │ OVERRIDE_SRCDIR=/path  METHOD=local │    │
  │  └────────────┴────────────────────────────────────┘    │
  │                                                          │
  │  Language Support:                                       │
  │  • C        ──  generic-package  +  TARGET_CONFIGURE_OPTS│
  │  • C++      ──  cmake-package    +  CONF_OPTS            │
  │  • Rust     ──  cargo-package    +  CARGO_ENV            │
  │                                                          │
  │  Key Workflow:                                           │
  │  1. Write Config.in → appears in menuconfig             │
  │  2. Write .mk      → defines fetch + build + install    │
  │  3. Source Config.in from package/Config.in             │
  │  4. make menuconfig → enable package                    │
  │  5. make (or make myapp)                                 │
  └──────────────────────────────────────────────────────────┘
```

**Key takeaways:**

- Every package is a `Config.in` + `.mk` pair living under `package/<name>/`.
- Use `generic-package` for raw Makefiles, `cmake-package` for CMake projects, and `cargo-package` for Rust/Cargo.
- The `BR2_OVERRIDE_SRCDIR` mechanism is the fastest path for active development: Buildroot rsyncs your local tree on every `make myapp-rebuild` without touching the download cache.
- Always use a hash file (`myapp.hash`) for tarball packages to guarantee build reproducibility and supply-chain integrity.
- Git-sourced packages should pin a full commit SHA, never a mutable branch name.
- Hooks (`POST_BUILD_CMDS`, `INSTALL_INIT_SYSV`, etc.) cover anything the infrastructure doesn't handle automatically — init scripts, systemd units, post-install fixups.
- The `make myapp-show-vars` and `make myapp V=1` targets are your first tools when a build goes wrong.

---

*End of Module 06 — Writing Custom Packages from Scratch*