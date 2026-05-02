# 07. Package Dependencies, Ordering & Virtual Packages

**Structure highlights:**

- **Dependency variables** — `*_DEPENDENCIES`, `*_PROVIDES`, `*_CONFLICTS` with full `.mk` file examples including conditional/optional dependency guards
- **Host/target split** — rules table and the `host-` prefix convention with sysroot path explanations
- **Virtual packages** — complete file structure showing the `openssl` virtual with `Config.in` choice block, empty virtual `.mk`, and both `libopenssl`/`libressl` provider `.mk` files
- **C example** — compile-time detection of OpenSSL vs LibreSSL via `LIBRESSL_VERSION_NUMBER` preprocessor guard
- **C++ example** — RAII `Sha256` wrapper using the shared `EVP_*` API that works identically under both providers
- **Rust example** — `cargo-package` integration using the `openssl` crate with `pkg-config` delegation, plus a feature-flag pattern to gate LibreSSL-specific code
- **Three ASCII diagrams** — dependency resolution Make chain, virtual package swap topology, and the host/target/staging sysroot layout
- **Summary table** — quick-reference mapping of every concept to its variable and purpose


> **Buildroot Series — Topic 07**
> Covering `*_DEPENDENCIES`, `*_PROVIDES`, `*_CONFLICTS`, host/target split propagation,
> and how virtual packages enable swappable implementations (e.g. OpenSSL vs LibreSSL).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Dependency Variables](#2-dependency-variables)
   - 2.1 `*_DEPENDENCIES`
   - 2.2 `*_PROVIDES`
   - 2.3 `*_CONFLICTS`
3. [Host / Target Split Propagation](#3-host--target-split-propagation)
4. [Virtual Packages](#4-virtual-packages)
5. [Dependency Ordering in Practice](#5-dependency-ordering-in-practice)
6. [C Code Example — Runtime Detection of SSL Library](#6-c-code-example--runtime-detection-of-ssl-library)
7. [C++ Code Example — Abstract TLS Backend](#7-c-code-example--abstract-tls-backend)
8. [Rust Code Example — Feature-Gated Crypto Backend](#8-rust-code-example--feature-gated-crypto-backend)
9. [ASCII Graphics: Dependency Resolution Flow](#9-ascii-graphics-dependency-resolution-flow)
10. [ASCII Graphics: Virtual Package Swap Diagram](#10-ascii-graphics-virtual-package-swap-diagram)
11. [ASCII Graphics: Host / Target Build Sysroot](#11-ascii-graphics-host--target-build-sysroot)
12. [Summary](#12-summary)

---

## 1. Overview

Buildroot's package infrastructure is built around **GNU Make** and a set of
well-defined makefile variables. Every package — whether it is a simple C
library, a Rust crate, or a Python module — declares:

- **what it needs** before it can be built (`*_DEPENDENCIES`),
- **what capability it exposes** to other packages (`*_PROVIDES`),
- **what it cannot coexist with** (`*_CONFLICTS`).

These three knobs, together with the host/target distinction and the concept of
*virtual packages*, give Buildroot a flexible but deterministic dependency
solver that resolves build ordering at `make` time rather than at runtime.

---

## 2. Dependency Variables

### 2.1 `*_DEPENDENCIES`

```makefile
# package/libcurl/libcurl.mk  (simplified)

LIBCURL_DEPENDENCIES = host-pkgconf zlib openssl
```

The variable lists **space-separated package names** that must be fully staged
before the current package's configure/build step begins.  Buildroot translates
each entry into a Make prerequisite, so the build graph is topologically sorted
automatically.

**Rules:**

| Situation | Declaration | Effect |
|-----------|-------------|--------|
| Target package needs target library | `MYPKG_DEPENDENCIES = libfoo` | `libfoo` is built and staged to `$(STAGING_DIR)` first |
| Target package needs a host-side tool | `MYPKG_DEPENDENCIES = host-cmake` | `host-cmake` is built to `$(HOST_DIR)` first |
| Host package needs a host library | `HOST_MYPKG_DEPENDENCIES = host-libfoo` | Both are host packages |
| Optional dependency (via `BR2_PACKAGE_LIBFOO`) | Use `ifeq` guard (see below) | Dependency added only when `libfoo` is selected |

**Optional (conditional) dependency:**

```makefile
ifeq ($(BR2_PACKAGE_LIBSSH2),y)
LIBCURL_DEPENDENCIES += libssh2
LIBCURL_CONF_OPTS   += --with-libssh2
else
LIBCURL_CONF_OPTS   += --without-libssh2
endif
```

### 2.2 `*_PROVIDES`

`*_PROVIDES` is used exclusively by **virtual packages** (see Section 4) to
announce the logical name that other packages may depend on.

```makefile
# package/libopenssl/libopenssl.mk
LIBOPENSSL_PROVIDES = openssl

# package/libressl/libressl.mk
LIBRESSL_PROVIDES = openssl
```

A consumer package can then write:

```makefile
LIBCURL_DEPENDENCIES = openssl   # resolved to whichever provider is selected
```

Buildroot resolves `openssl` to the concrete package whose `*_PROVIDES`
includes that name.  Selecting *both* providers for the same virtual is an
error caught at configure time.

### 2.3 `*_CONFLICTS`

`*_CONFLICTS` lets a package explicitly declare packages it cannot coexist
with.  Buildroot will emit a configuration-time error if conflicting packages
are both selected.

```makefile
# package/libressl/libressl.mk
LIBRESSL_CONFLICTS = libopenssl
```

This is a safety net on top of the virtual-package mechanism: even if the user
bypasses the Kconfig `select`/`depends on` guards by directly enabling both
packages, the build will abort early with a clear error message rather than
producing a silently broken target image.

---

## 3. Host / Target Split Propagation

Buildroot maintains two separate sysroots:

```
$(HOST_DIR)     — tools that run on the BUILD machine
$(STAGING_DIR)  — libraries/headers for the TARGET architecture
$(TARGET_DIR)   — final root filesystem image
```

When you list a dependency, Buildroot automatically applies the **same
host/target classification** as the depending package:

```makefile
# A *target* package
LIBPNG_DEPENDENCIES = host-pkgconf zlib

# Buildroot expands this to:
#   host-pkgconf  → built into $(HOST_DIR)   (it is already a host package)
#   zlib          → built into $(STAGING_DIR) (target context)
```

For a host package the same line reads differently:

```makefile
# A *host* package (e.g., host-libpng)
HOST_LIBPNG_DEPENDENCIES = host-pkgconf host-zlib
# Everything stays on the host sysroot.
```

**Key propagation rules:**

1. A target package listing `host-foo` is valid and common (cross-build tools).
2. A target package must **never** list a bare `host-foo` when it means the
   target version — use `foo` instead.
3. A host package listing a bare `foo` is an **error** — the build machine
   cannot link against a target library.

---

## 4. Virtual Packages

### Concept

A *virtual package* is a package that has no source code of its own.  Its sole
job is to:

- define the logical interface name (via `*_PROVIDES` on the providers),
- expose a Kconfig `choice` block so exactly one provider is selected,
- forward the `DEPENDENCIES` of consumers to the chosen concrete package.

### File Structure

```
package/
  openssl/            ← virtual package directory
    Config.in         ← choice block
    openssl.mk        ← minimal mk file (no sources)
  libopenssl/         ← concrete provider A
    Config.in
    libopenssl.mk
  libressl/           ← concrete provider B
    Config.in
    libressl.mk
```

### `openssl/Config.in` — the choice block

```kconfig
# package/openssl/Config.in

config BR2_PACKAGE_OPENSSL
    bool "openssl"

if BR2_PACKAGE_OPENSSL

choice
    prompt "SSL library"
    default BR2_PACKAGE_LIBOPENSSL

config BR2_PACKAGE_LIBOPENSSL
    bool "libopenssl"
    select BR2_PACKAGE_OPENSSL
    help
      OpenSSL 3.x — the reference implementation.

config BR2_PACKAGE_LIBRESSL
    bool "libressl"
    select BR2_PACKAGE_OPENSSL
    help
      LibreSSL — the OpenBSD fork focused on security.

endchoice
endif
```

### `openssl/openssl.mk` — the virtual mk

```makefile
################################################################################
#
# openssl  (virtual package)
#
################################################################################

# No OPENSSL_VERSION, OPENSSL_SOURCE, or OPENSSL_SITE needed.
# The infrastructure just records which provider was chosen.

$(eval $(virtual-package))
```

### Provider mk files

```makefile
# package/libopenssl/libopenssl.mk

LIBOPENSSL_VERSION  = 3.3.0
LIBOPENSSL_SITE     = https://www.openssl.org/source
LIBOPENSSL_SOURCE   = openssl-$(LIBOPENSSL_VERSION).tar.gz
LIBOPENSSL_LICENSE  = Apache-2.0

LIBOPENSSL_PROVIDES = openssl          # <-- advertise virtual name
LIBOPENSSL_CONFLICTS = libressl        # <-- cannot coexist

$(eval $(autotools-package))
```

```makefile
# package/libressl/libressl.mk

LIBRESSL_VERSION  = 3.9.1
LIBRESSL_SITE     = https://ftp.openbsd.org/pub/OpenBSD/LibreSSL
LIBRESSL_SOURCE   = libressl-$(LIBRESSL_VERSION).tar.gz
LIBRESSL_LICENSE  = OpenSSL

LIBRESSL_PROVIDES = openssl            # <-- same virtual name
LIBRESSL_CONFLICTS = libopenssl

$(eval $(cmake-package))
```

### Consumer package

```makefile
# package/libcurl/libcurl.mk

LIBCURL_DEPENDENCIES = openssl zlib host-pkgconf

# At build time Buildroot resolves "openssl" → libopenssl OR libressl
# depending on what the user selected in menuconfig.
```

---

## 5. Dependency Ordering in Practice

Buildroot converts the dependency graph into Make targets.  Each package
generates targets of the form:

```
$(BUILD_DIR)/<pkg>-<version>/.stamp_built
$(STAGING_DIR)/.../lib<pkg>.so  (installed)
```

The ordering chain for a typical package looks like:

```
<pkg>-patch   →  <pkg>-configure  →  <pkg>-build  →  <pkg>-install-staging
                         ↑
              depends on:  dep1-install-staging
                           dep2-install-staging
                           host-dep3-install-host
```

Circular dependencies are not detected gracefully — they produce infinite
recursion in Make — so the Buildroot convention is: if package A depends on
B and B depends on A, one of them must be refactored into two packages
(interface + implementation).

---

## 6. C Code Example — Runtime Detection of SSL Library

The following C snippet shows how application code can detect at compile time
which SSL library Buildroot selected, using preprocessor macros exported by
each provider.

```c
/* src/tls_info.c
 *
 * Buildroot selects either OpenSSL or LibreSSL.
 * Both expose <openssl/opensslv.h> but define different version macros.
 * This file prints which library is actually present.
 */

#include <stdio.h>
#include <openssl/opensslv.h>   /* available from either provider */
#include <openssl/ssl.h>

/* LibreSSL defines LIBRESSL_VERSION_NUMBER in addition to OPENSSL_VERSION_NUMBER */
#ifdef LIBRESSL_VERSION_NUMBER
#  define BACKEND_NAME    "LibreSSL"
#  define BACKEND_VERSION LIBRESSL_TEXT
#else
#  define BACKEND_NAME    "OpenSSL"
#  define BACKEND_VERSION OPENSSL_VERSION_TEXT
#endif

int main(void)
{
    /* SSL_library_init() is a no-op in OpenSSL 1.1+ but safe to call */
    SSL_library_init();

    printf("SSL backend : %s\n", BACKEND_NAME);
    printf("Version     : %s\n", BACKEND_VERSION);
    printf("Build flags : %s\n", SSLeay_version(SSLEAY_CFLAGS));

    return 0;
}
```

**Corresponding Buildroot package mk:**

```makefile
# package/tls-info/tls-info.mk

TLS_INFO_VERSION = 1.0.0
TLS_INFO_SITE    = $(TOPDIR)/package/tls-info/src
TLS_INFO_SITE_METHOD = local

# Depend on the VIRTUAL package name, not on a concrete provider.
TLS_INFO_DEPENDENCIES = openssl

$(eval $(autotools-package))
```

---

## 7. C++ Code Example — Abstract TLS Backend

This example demonstrates a C++ abstraction layer that compiles against
whichever SSL provider Buildroot has installed, using policy-based design so
no runtime branching is needed.

```cpp
// include/tls_backend.hpp
//
// Policy class: compiled once against the selected SSL provider.
// Buildroot guarantees that exactly one of HAVE_OPENSSL / HAVE_LIBRESSL
// is set via the package's CPPFLAGS.

#pragma once
#include <string>
#include <stdexcept>

#ifdef HAVE_OPENSSL
#  include <openssl/evp.h>
#  include <openssl/err.h>
#  include <openssl/opensslv.h>
#endif

namespace tls {

/// Returns a human-readable string identifying the linked SSL library.
inline std::string backend_info()
{
#ifdef LIBRESSL_VERSION_NUMBER
    return std::string("LibreSSL ") + LIBRESSL_TEXT;
#elif defined(OPENSSL_VERSION_TEXT)
    return std::string("OpenSSL ") + OPENSSL_VERSION_TEXT;
#else
    return "Unknown SSL backend";
#endif
}

/// Thin RAII wrapper around EVP_MD_CTX (identical API in both libraries).
class Sha256
{
public:
    Sha256()
    {
        ctx_ = EVP_MD_CTX_new();
        if (!ctx_)
            throw std::runtime_error("EVP_MD_CTX_new failed");
        if (EVP_DigestInit_ex(ctx_, EVP_sha256(), nullptr) != 1)
            throw std::runtime_error("EVP_DigestInit_ex failed");
    }

    ~Sha256() { EVP_MD_CTX_free(ctx_); }

    // Prevent copy; allow move
    Sha256(const Sha256&) = delete;
    Sha256& operator=(const Sha256&) = delete;

    void update(const void* data, std::size_t len)
    {
        if (EVP_DigestUpdate(ctx_, data, len) != 1)
            throw std::runtime_error("EVP_DigestUpdate failed");
    }

    /// Returns raw 32-byte digest
    std::string finalize()
    {
        unsigned char digest[EVP_MAX_MD_SIZE];
        unsigned int  digest_len = 0;
        if (EVP_DigestFinal_ex(ctx_, digest, &digest_len) != 1)
            throw std::runtime_error("EVP_DigestFinal_ex failed");
        return std::string(reinterpret_cast<char*>(digest), digest_len);
    }

private:
    EVP_MD_CTX* ctx_;
};

} // namespace tls
```

```cpp
// src/main.cpp
#include <iostream>
#include "tls_backend.hpp"

int main()
{
    std::cout << "Backend: " << tls::backend_info() << "\n";

    tls::Sha256 h;
    const std::string msg = "Buildroot virtual package demo";
    h.update(msg.data(), msg.size());

    std::string digest = h.finalize();
    std::cout << "SHA-256 of \"" << msg << "\":\n  ";
    for (unsigned char c : digest)
        std::printf("%02x", static_cast<unsigned char>(c));
    std::cout << "\n";
    return 0;
}
```

**Buildroot mk snippet:**

```makefile
# package/tls-demo-cpp/tls-demo-cpp.mk

TLS_DEMO_CPP_VERSION      = 1.0.0
TLS_DEMO_CPP_SITE         = $(TOPDIR)/package/tls-demo-cpp/src
TLS_DEMO_CPP_SITE_METHOD  = local
TLS_DEMO_CPP_DEPENDENCIES = openssl
TLS_DEMO_CPP_CXXFLAGS     = -DHAVE_OPENSSL   # set unconditionally;
                                              # LibreSSL defines its own guard

$(eval $(cmake-package))
```

---

## 8. Rust Code Example — Feature-Gated Crypto Backend

Rust packages in Buildroot use the `cargo-package` infrastructure.  The
virtual-package mechanism still applies: `DEPENDENCIES = openssl` ensures the
provider is staged before `cargo build` runs.  The `openssl` crate on
crates.io automatically links against whichever `libssl` is found via
`pkg-config`, which Buildroot configures to point at `$(STAGING_DIR)`.

### `Cargo.toml`

```toml
[package]
name    = "br-tls-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
# The openssl crate delegates to whatever libssl pkg-config finds.
# Buildroot's cross pkg-config is set up by the openssl virtual package.
openssl = { version = "0.10", features = ["v110", "vendored"] }

[features]
default     = []
libressl    = []   # Kconfig-driven feature flag (see mk file)
```

### `src/main.rs`

```rust
//! br-tls-demo/src/main.rs
//!
//! Demonstrates linking against the Buildroot-selected SSL provider.
//! The `openssl` crate exposes version information at compile time via
//! the OPENSSL_VERSION environment variable set by its build script.

use openssl::ssl::{SslConnector, SslMethod};
use openssl::version;

fn main() {
    // version::version() returns the runtime version string of the linked lib.
    // Under LibreSSL this starts with "LibreSSL"; under OpenSSL with "OpenSSL".
    let ver = version::version();
    let backend = if ver.starts_with("LibreSSL") {
        "LibreSSL"
    } else {
        "OpenSSL"
    };

    println!("SSL backend  : {backend}");
    println!("Version text : {ver}");
    println!("Number       : 0x{:08x}", version::number());

    // Build a connector to confirm the library is functional
    let _ctx = SslConnector::builder(SslMethod::tls())
        .expect("Failed to create SSL context — library may be misconfigured");

    println!("TLS context created successfully.");
}
```

### `package/br-tls-demo/br-tls-demo.mk`

```makefile
################################################################################
#
# br-tls-demo  — Rust demo for virtual SSL package
#
################################################################################

BR_TLS_DEMO_VERSION        = 0.1.0
BR_TLS_DEMO_SITE           = $(TOPDIR)/package/br-tls-demo/src
BR_TLS_DEMO_SITE_METHOD    = local

# Depend on the virtual package.  Buildroot resolves this to libopenssl
# or libressl depending on menuconfig selection.
BR_TLS_DEMO_DEPENDENCIES   = openssl

# Pass a Cargo feature flag when LibreSSL is selected so the crate can
# conditionally compile LibreSSL-specific code paths if needed.
ifeq ($(BR2_PACKAGE_LIBRESSL),y)
BR_TLS_DEMO_CARGO_ENV += BR_SSL_BACKEND=libressl
endif

$(eval $(cargo-package))
```

---

## 9. ASCII Graphics: Dependency Resolution Flow

```
  menuconfig selection
  ┌─────────────────────────────────────────────────────────────────┐
  │  [*] libcurl                                                    │
  │  [*]   openssl  (virtual)                                       │
  │          ├── (●) libopenssl   ← user selects this               │
  │          └── ( ) libressl                                       │
  └─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
  Buildroot dependency graph (Make targets)
  ┌───────────────────────────────────────────────────────────────┐
  │                                                               │
  │  host-pkgconf          zlib             libopenssl            │
  │       │                 │                    │                │
  │       │  .stamp_built   │  .stamp_built      │  .stamp_built  │
  │       └────────┬────────┘                    │                │
  │                │                             │                │
  │                ▼                             ▼                │
  │         libcurl-configure  ◄──────────────────                │
  │                │                                              │
  │                ▼                                              │
  │         libcurl-build                                         │
  │                │                                              │
  │                ▼                                              │
  │         libcurl-install-staging                               │
  │                │                                              │
  │                ▼                                              │
  │         myapp-configure  (depends on libcurl)                 │
  │                │                                              │
  │                ▼                                              │
  │         myapp-build                                           │
  │                │                                              │
  │                ▼                                              │
  │         myapp-install-target  ──► $(TARGET_DIR)/usr/bin/      │
  └───────────────────────────────────────────────────────────────┘
```

---

## 10. ASCII Graphics: Virtual Package Swap Diagram

```
  ╔══════════════════════════════════════════════════════════════╗
  ║           Virtual Package:  openssl                          ║
  ║   (no source — only a Kconfig choice + empty .mk)            ║
  ╚══════════════════════════════════════════════════════════════╝
             ▲                            ▲
             │  PROVIDES = openssl        │  PROVIDES = openssl
  ┌──────────┴───────────┐      ┌─────────┴────────────┐
  │     libopenssl       │      │      libressl        │
  │  (OpenSSL 3.x)       │      │  (OpenBSD fork)      │
  │                      │      │                      │
  │  libssl.so           │      │  libssl.so           │
  │  libcrypto.so        │      │  libcrypto.so        │
  │  openssl.pc          │      │  openssl.pc          │
  └──────────────────────┘      └──────────────────────┘
       ▲  CONFLICTS = libressl        ▲  CONFLICTS = libopenssl
       │                              │
       └──────────────┬───────────────┘
                      │
            Only ONE selected at a time
            enforced by Kconfig choice block
                      │
                      ▼
  ┌────────────────────────────────────────────────────────────┐
  │  Consumer packages that depend on "openssl":               │
  │                                                            │
  │    libcurl     →  DEPENDENCIES = openssl                   │
  │    wget        →  DEPENDENCIES = openssl                   │
  │    mosquitto   →  DEPENDENCIES = openssl                   │
  │    br-tls-demo →  DEPENDENCIES = openssl                   │
  │                                                            │
  │  All resolved transparently to whichever provider is       │
  │  selected — no changes needed in consumer .mk files.       │
  └────────────────────────────────────────────────────────────┘
```

---

## 11. ASCII Graphics: Host / Target Build Sysroot

```
  BUILD MACHINE (x86-64)
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  $(HOST_DIR)/                                                    │
  │  ├── bin/                                                        │
  │  │   ├── arm-linux-gnueabihf-gcc   ← cross compiler              │
  │  │   ├── cmake                     ← host-cmake                  │
  │  │   ├── pkg-config                ← host-pkgconf                │
  │  │   └── openssl   (host tool)     ← host-libopenssl             │
  │  ├── lib/                                                        │
  │  │   └── libssl.so   (for host tools only)                       │
  │  └── arm-linux-gnueabihf/sysroot/                                │
  │      └── (= STAGING_DIR, see below)                              │
  │                                                                  │
  │  $(STAGING_DIR)/   ← cross-compiled, target ABI                  │
  │  ├── usr/include/openssl/          ← headers for cross-build     │
  │  └── usr/lib/                                                    │
  │      ├── libssl.so                 ← target library              │
  │      └── libcrypto.so                                            │
  │                                                                  │
  │  $(TARGET_DIR)/    ← final rootfs (no headers, no .a files)      │
  │  └── usr/lib/                                                    │
  │      ├── libssl.so.3                                             │
  │      └── libcrypto.so.3                                          │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
              │                              │
              │  deployed to                 │  linked against
              ▼                              ▼
  TARGET DEVICE (ARM)             myapp cross-compile step
  ├── /usr/lib/libssl.so.3        uses $(STAGING_DIR) headers/libs
  └── /usr/bin/myapp
```

---

## 12. Summary

| Concept | Variable / Mechanism | Purpose |
|---|---|---|
| Build-time prerequisites | `*_DEPENDENCIES` | Ensures libs/tools are staged before a package builds |
| Conditional dependency | `ifeq ($(BR2_PACKAGE_FOO),y)` guard | Adds deps only when optional feature is enabled |
| Virtual interface name | `*_PROVIDES` | Lets providers advertise a logical name |
| Mutual exclusion | `*_CONFLICTS` | Prevents two conflicting packages from being simultaneously selected |
| Provider selection | Kconfig `choice` block | User picks exactly one provider in `menuconfig` |
| Host/target separation | `host-` prefix convention | Keeps build-machine tools out of target sysroot |
| Build ordering | Make prerequisite chains | Topological sort derived automatically from `*_DEPENDENCIES` |

### Key Takeaways

**Declaring dependencies correctly is the single most important task when
writing a new Buildroot package.** An under-specified dependency list leads
to intermittent parallel-build failures; an over-specified list slows the
build unnecessarily.

**Virtual packages decouple consumers from providers.** A package that writes
`DEPENDENCIES = openssl` will continue to work whether the user selects
OpenSSL, LibreSSL, or any future alternative that registers the same
`PROVIDES` name — without any change to the consumer `.mk` file.

**The host/target split is not optional.** Mixing host and target sysroots is
the root cause of the most subtle cross-compilation failures.  Following the
`host-` naming convention and never linking a host package against bare target
libraries keeps both sysroots clean.

**`*_CONFLICTS` is a last line of defence.** Kconfig `depends on` and `select`
should prevent invalid combinations at configuration time, but `*_CONFLICTS`
gives Buildroot a Make-level safety net that fires even when Kconfig checks
are bypassed by manual `.config` editing.

---

*Document generated for the Buildroot series — Topic 07.*
*All code examples are illustrative and simplified for clarity.*