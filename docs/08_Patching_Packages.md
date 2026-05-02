Now let me create the comprehensive markdown document:Here's your complete document — **08_Patching_Packages.md** — covering all the requested aspects:

**Structure at a glance:**

- **Patch layout** — where patches live in the Buildroot tree, including `BR2_GLOBAL_PATCH_DIR` for out-of-tree BSPs, with ASCII directory diagrams
- **Naming conventions** — the `NNNN-kebab-case.patch` rule explained with good/bad examples
- **`series` file** — minimal and extended forms, strip-level options, commenting out patches
- **Quilt workflow** — `.quiltrc` setup, full step-by-step session, and a complete command reference
- **How Buildroot applies patches** — the internal `apply-patches.sh` call chain shown in ASCII
- **Conditional per-arch patching** — three patterns (subdirectories, `ifeq`, `POST_PATCH_HOOKS`) with the full set of `BR2_*` architecture symbols
- **C example** — patching a `clock_gettime()` fallback for old embedded kernels
- **C++ example** — removing illegal `-march=native` from a CMake-based JSON library, with the corresponding `.mk`
- **Rust example** — feature-gating the `ring` crate for `armv5te` targets that can't assemble its BoringSSL code, with the Cargo feature switch wired through `.mk`
- **Upstream submission workflow** — full ASCII decision tree, annotated `series` tracking comments, and the correct patch header format
- **Pitfalls & summary** — rule table, decision trees, and version-bump guidance

# 08. Patching Packages in Buildroot

> **Topic:** `series` files, `quilt`, naming conventions (`0001-*.patch`),
> upstream submission workflow, and applying patches conditionally per architecture.

---

## Table of Contents

1. [Overview](#overview)
2. [Patch Storage Layout](#patch-storage-layout)
3. [Naming Conventions](#naming-conventions)
4. [The `series` File](#the-series-file)
5. [Quilt: Creating and Managing Patches](#quilt-creating-and-managing-patches)
6. [How Buildroot Applies Patches](#how-buildroot-applies-patches)
7. [Conditional Patching per Architecture](#conditional-patching-per-architecture)
8. [C Code Example — Patching a C Library](#c-code-example--patching-a-c-library)
9. [C++ Code Example — Patching a C++ Package](#c-code-example--patching-a-c-package)
10. [Rust Code Example — Patching a Rust Package](#rust-code-example--patching-a-rust-package)
11. [Upstream Submission Workflow](#upstream-submission-workflow)
12. [Common Pitfalls](#common-pitfalls)
13. [Summary](#summary)

---

## Overview

Buildroot does not build packages from a rolling source tree — it downloads specific
**tarballs** or **VCS snapshots** and then applies a set of local patches before
compiling. This patching layer serves several purposes:

- **Bug fixes** not yet merged upstream
- **Cross-compilation fixes** required when the upstream build system assumes a
  native host
- **Feature additions** needed specifically for embedded targets
- **Security backports** applied to an older, stable upstream version

Patches in Buildroot follow the **quilt** patch-stack model: an ordered series of
`.patch` files, tracked in a `series` file, applied in sequence on top of the
upstream source.

```
Upstream tarball
      |
      v
  patch 0001  -->  patch 0002  -->  patch 0003  -->  Build
```

---

## Patch Storage Layout

Patches live **inside the package directory** of the Buildroot tree, right next to
the package's `.mk` and `Config.in` files.

```
buildroot/
  package/
    libfoo/
      Config.in
      libfoo.mk
      0001-fix-cross-compile-flags.patch
      0002-disable-host-tool-detection.patch
      0003-add-riscv-support.patch
      series                          <-- optional but recommended
```

Buildroot also accepts patches in a **global patch directory** configured via
`BR2_GLOBAL_PATCH_DIR`. This is useful for out-of-tree BSP layers:

```
my-bsp/
  patches/
    libfoo/
      0004-bsp-specific-fix.patch
```

### ASCII Layout Diagram

```
  +---------------------------------------------------+
  |              Buildroot Source Tree                |
  |                                                   |
  |  package/                                         |
  |  +-- libfoo/                                      |
  |       +-- Config.in          (Kconfig menu)       |
  |       +-- libfoo.mk          (build rules)        |
  |       +-- series             (patch order)        |
  |       +-- 0001-fix-cross.patch                    |
  |       +-- 0002-no-host-cc.patch                   |
  |       +-- 0003-arm-asm.patch                      |
  |                                                   |
  |  BR2_GLOBAL_PATCH_DIR (optional, out-of-tree):    |
  |  +-- patches/libfoo/                              |
  |       +-- 0004-board-fix.patch                    |
  +---------------------------------------------------+
                        |
          Buildroot build system reads
          patches in sorted order and
          applies them with patch(1)
          or quilt.
                        |
                        v
  +---------------------------------------------------+
  |          $(BUILD_DIR)/libfoo-1.2.3/               |
  |          (extracted + patched source)             |
  +---------------------------------------------------+
```

---

## Naming Conventions

Buildroot enforces a strict naming scheme modelled on the Linux kernel patch format:

```
NNNN-short-description-in-kebab-case.patch
```

| Part             | Rules                                                     |
|------------------|-----------------------------------------------------------|
| `NNNN`           | Four-digit zero-padded sequence number (`0001`, `0042`)   |
| `-`              | Single hyphen separator                                   |
| `short-desc`     | Lowercase, hyphen-separated, ≤ ~50 characters             |
| `.patch`         | Always `.patch`, never `.diff`                            |

### Good Examples

```
0001-fix-cross-compilation-with-musl.patch
0002-disable-werror-on-clang.patch
0003-add-mips-tlb-flush-barrier.patch
0004-pkg-config-path-fix.patch
```

### Bad Examples

```
fix.patch                  <-- no sequence number
0001_fix_cross.patch       <-- underscores instead of hyphens
patch1.diff                <-- wrong extension, no description
0001-Fix-Cross-Compile.patch <-- uppercase letters
```

The four-digit prefix ensures `ls` and `sort` produce a deterministic order,
which is critical because patches often depend on each other.

---

## The `series` File

The `series` file is a plain-text list that explicitly controls patch application
order and allows commenting out patches without deleting them.

### Minimal `series` File

```
# libfoo patch series
0001-fix-cross-compilation-with-musl.patch
0002-disable-werror-on-clang.patch
0003-add-mips-tlb-flush-barrier.patch
```

### Extended `series` File with Options

```
# libfoo patch series
# -p1 strips one path component (default, matches git format-patch output)
0001-fix-cross-compilation-with-musl.patch -p1
0002-disable-werror-on-clang.patch         -p1

# Temporarily disabled — waiting for upstream review
# 0003-experimental-lto-support.patch

# Applies at strip level 0 (rare, non-git patches)
0004-legacy-vendor-fix.patch -p0
```

**When Buildroot finds a `series` file it uses it exclusively.** If no `series`
file is present, Buildroot applies all `*.patch` files in the directory, sorted
lexicographically — which is why the `NNNN-` prefix is mandatory in that case.

---

## Quilt: Creating and Managing Patches

[Quilt](https://savannah.nongnu.org/projects/quilt) is the canonical tool for
creating, editing, and organising patch stacks. Buildroot uses quilt internally
and it is the recommended tool for maintainers.

### Installing Quilt

```bash
# Debian / Ubuntu
sudo apt-get install quilt

# Fedora / RHEL
sudo dnf install quilt

# macOS (Homebrew)
brew install quilt
```

### Recommended `.quiltrc`

```bash
# ~/.quiltrc
QUILT_DIFF_ARGS="--no-timestamps --no-index -p ab"
QUILT_REFRESH_ARGS="--no-timestamps --no-index -p ab"
QUILT_SERIES_ARGS="--color=auto"
QUILT_PATCH_OPTS="--unified"
QUILT_DIFF_OPTS="-p"
EDITOR=vim
```

### Quilt Workflow — Step by Step

```
  +-----------+     quilt push      +-----------+
  | Upstream  |  ---------------->  | After     |
  | Source    |                     | Patch N   |
  +-----------+     quilt pop       +-----------+
        ^       <------------------
        |
        |  quilt new 0004-my-fix.patch
        |  quilt add src/foo.c          (mark files before editing)
        |  vim src/foo.c                (make the change)
        |  quilt refresh               (record the diff)
        |  quilt header -e             (add commit message)
        v
  +-----------+
  | 0004-my-  |
  | fix.patch |
  +-----------+
```

#### Full Example Session

```bash
# 1. Enter the extracted source directory
cd output/build/libfoo-1.2.3/

# 2. Initialise quilt if not already done
quilt init

# 3. Import existing patches (if any)
quilt import ../../package/libfoo/*.patch

# 4. Apply all existing patches first
quilt push -a

# 5. Create a new patch on top
quilt new 0004-fix-endian-detection.patch

# 6. Tell quilt which files you will change (BEFORE editing)
quilt add src/endian.c include/endian.h

# 7. Make the fix in your editor
vim src/endian.c

# 8. Record the changes into the patch file
quilt refresh

# 9. Add a proper description header (matches upstream commit format)
quilt header -e

# 10. Review the generated patch
quilt diff -z
cat patches/0004-fix-endian-detection.patch

# 11. Copy the finished patch back to the package directory
cp patches/0004-fix-endian-detection.patch \
   ../../../../package/libfoo/0004-fix-endian-detection.patch

# 12. Update the series file
echo "0004-fix-endian-detection.patch" >> \
   ../../../../package/libfoo/series
```

### Quilt Command Reference

```
quilt series          -- list all patches in the stack
quilt applied         -- show applied patches
quilt unapplied       -- show unapplied patches
quilt push            -- apply next patch
quilt push -a         -- apply all patches
quilt pop             -- unapply top patch
quilt pop -a          -- unapply all patches
quilt new NAME.patch  -- create a new (empty) patch
quilt add FILE        -- stage file for patching
quilt edit FILE       -- add + open in $EDITOR
quilt refresh         -- update patch from working tree diff
quilt diff            -- show uncommitted changes
quilt header          -- view/edit patch description
quilt fold            -- collapse top patch into previous
quilt delete NAME     -- remove a patch from the series
quilt rename NEW      -- rename the top patch
quilt graph           -- show dependency graph (ASCII)
```

---

## How Buildroot Applies Patches

Buildroot's patching logic lives in `support/scripts/apply-patches.sh` and in
the `pkg-generic.mk` infrastructure. The call chain looks like this:

```
  libfoo.mk
    LIBFOO_VERSION = 1.2.3
    LIBFOO_SOURCE  = libfoo-$(LIBFOO_VERSION).tar.gz
    $(eval $(generic-package))
           |
           v
  pkg-generic.mk
    target: $(LIBFOO_DIR)/.stamp_patched
           |
           v
  apply-patches.sh
    1. Collect patches from package/libfoo/
    2. Collect patches from BR2_GLOBAL_PATCH_DIR/libfoo/
    3. Merge and sort (series file overrides sort order)
    4. Apply each with: patch -p1 < PATCHFILE
    5. Touch .stamp_patched on success
```

### Relevant Makefile Variables

```makefile
# In libfoo.mk — rarely needed, Buildroot auto-discovers patches
LIBFOO_PATCH       = $(sort $(wildcard package/libfoo/*.patch))

# Override patch strip level (default is 1)
LIBFOO_PATCH_EXTRA_OPTS = -p2

# Provide patches from a VCS source instead of a tarball
LIBFOO_VERSION     = abc123def456
LIBFOO_SITE        = https://github.com/example/libfoo
LIBFOO_SITE_METHOD = git
```

---

## Conditional Patching per Architecture

Buildroot does not have a built-in syntax for architecture-conditional patches in
the `series` file. The standard patterns are:

### Pattern 1 — Subdirectory per Architecture

Maintain separate patch subdirectories and select them in the `.mk` file:

```
package/libfoo/
  0001-base-fix.patch
  arm/
    0002-arm-neon-fix.patch
  riscv/
    0002-riscv-tlb.patch
  x86/
    0002-x86-sse2.patch
```

```makefile
# libfoo.mk
LIBFOO_ARCH_PATCHES_DIR = package/libfoo/$(ARCH)

define LIBFOO_APPLY_ARCH_PATCHES
    if [ -d $(LIBFOO_ARCH_PATCHES_DIR) ]; then \
        for p in $(LIBFOO_ARCH_PATCHES_DIR)/*.patch; do \
            patch -d $(@D) -p1 < $$p; \
        done; \
    fi
endef

LIBFOO_POST_PATCH_HOOKS += LIBFOO_APPLY_ARCH_PATCHES
```

### Pattern 2 — Conditional in `.mk` with `ifeq`

```makefile
# libfoo.mk
ifeq ($(BR2_arm),y)
LIBFOO_PATCH += package/libfoo/0010-arm-neon-workaround.patch
endif

ifeq ($(BR2_riscv),y)
LIBFOO_PATCH += package/libfoo/0010-riscv-tlb-barrier.patch
endif

ifeq ($(BR2_x86_64),y)
LIBFOO_PATCH += package/libfoo/0010-x86-sse2-optimise.patch
endif
```

### Pattern 3 — Using a `POST_PATCH_HOOK`

```makefile
# libfoo.mk — most flexible approach
define LIBFOO_CONDITIONAL_PATCHES
    $(if $(BR2_arm),
        patch -d $(@D) -p1 < \
            $(TOPDIR)/package/libfoo/arch/0010-arm-fix.patch)
    $(if $(BR2_mips),
        patch -d $(@D) -p1 < \
            $(TOPDIR)/package/libfoo/arch/0010-mips-fix.patch)
endef

LIBFOO_POST_PATCH_HOOKS += LIBFOO_CONDITIONAL_PATCHES
```

### Architecture Detection Variables

```makefile
# Commonly used architecture Kconfig symbols in .mk files:
BR2_arm            # 32-bit ARM (any variant)
BR2_aarch64        # 64-bit ARM (AArch64)
BR2_x86_64         # x86-64
BR2_i386           # x86 32-bit
BR2_mips           # MIPS 32-bit big-endian
BR2_mipsel         # MIPS 32-bit little-endian
BR2_mips64         # MIPS 64-bit
BR2_riscv          # RISC-V (any width)
BR2_powerpc        # PowerPC 32-bit
BR2_powerpc64      # PowerPC 64-bit
BR2_s390x          # IBM s390x
BR2_ARCH           # String: "arm", "aarch64", "x86_64", ...
BR2_ENDIAN         # "LITTLE" or "BIG"
```

---

## C Code Example — Patching a C Library

### Scenario

`libfoo` calls `clock_gettime()` without linking `-lrt` on older glibc toolchains.
The cross-compiler in Buildroot's musl toolchain does not need `-lrt`, but the
autoconf probe incorrectly skips the flag on uClibc-ng.

### Original `configure.ac` Fragment (upstream)

```c
/* src/timer.c -- upstream, before patch */
#include <time.h>
#include <stdio.h>

int get_monotonic_ns(long long *out)
{
    struct timespec ts;
    /* BUG: assumes CLOCK_MONOTONIC never returns ENOSYS */
    if (clock_gettime(CLOCK_MONOTONIC, &ts) != 0)
        return -1;
    *out = (long long)ts.tv_sec * 1000000000LL + ts.tv_nsec;
    return 0;
}
```

### Patch File: `0001-timer-handle-missing-monotonic-clock.patch`

```diff
From 4a7b1c2e3d5f6a8b9c0d1e2f3a4b5c6d7e8f90ab Mon Sep 17 00:00:00 2001
From: Jane Developer <jane@example.com>
Date: Mon, 01 Jan 2024 12:00:00 +0000
Subject: [PATCH] timer: handle missing CLOCK_MONOTONIC on older kernels

Some embedded kernels (< 2.6.28) do not support CLOCK_MONOTONIC.
Fall back to CLOCK_REALTIME in that case so libfoo still builds
and runs on those targets.

Signed-off-by: Jane Developer <jane@example.com>
---
 src/timer.c | 14 ++++++++++++--
 1 file changed, 12 insertions(+), 2 deletions(-)

diff --git a/src/timer.c b/src/timer.c
index a1b2c3d..d4e5f6a 100644
--- a/src/timer.c
+++ b/src/timer.c
@@ -1,11 +1,21 @@
 #include <time.h>
 #include <stdio.h>
+#include <errno.h>
 
 int get_monotonic_ns(long long *out)
 {
     struct timespec ts;
-    /* BUG: assumes CLOCK_MONOTONIC never returns ENOSYS */
-    if (clock_gettime(CLOCK_MONOTONIC, &ts) != 0)
-        return -1;
+    clockid_t clk = CLOCK_MONOTONIC;
+
+    /* Some old embedded kernels do not support CLOCK_MONOTONIC;
+     * fall back to CLOCK_REALTIME which is universally available. */
+    if (clock_gettime(clk, &ts) != 0) {
+        if (errno == EINVAL || errno == ENOSYS) {
+            clk = CLOCK_REALTIME;
+            if (clock_gettime(clk, &ts) != 0)
+                return -1;
+        } else {
+            return -1;
+        }
+    }
+
     *out = (long long)ts.tv_sec * 1000000000LL + ts.tv_nsec;
     return 0;
 }
```

### Corresponding `libfoo.mk` Entry

```makefile
################################################################################
#
# libfoo
#
################################################################################

LIBFOO_VERSION = 1.2.3
LIBFOO_SITE    = https://example.com/releases
LIBFOO_SOURCE  = libfoo-$(LIBFOO_VERSION).tar.gz
LIBFOO_LICENSE = MIT
LIBFOO_LICENSE_FILES = COPYING

# No explicit LIBFOO_PATCH needed — Buildroot auto-discovers *.patch files
# in package/libfoo/ and applies them in sorted order.

$(eval $(autotools-package))
```

---

## C++ Code Example — Patching a C++ Package

### Scenario

`libbar` is a C++ JSON library. Its `CMakeLists.txt` hard-codes
`-march=native`, which is illegal when cross-compiling.

### Patch File: `0001-cmake-remove-march-native-for-cross-build.patch`

```diff
From 9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b2a1f0e Mon Sep 17 00:00:00 2001
From: Build Bot <buildbot@example.com>
Date: Tue, 02 Jan 2024 09:00:00 +0000
Subject: [PATCH] cmake: remove -march=native to allow cross-compilation

-march=native instructs the compiler to optimise for the host
machine's CPU. This is incompatible with cross-compilation where
host != target. Remove the flag and let the Buildroot toolchain
wrapper supply the correct -mcpu / -march flags.

Signed-off-by: Build Bot <buildbot@example.com>
---
 CMakeLists.txt | 8 +++-----
 1 file changed, 3 insertions(+), 5 deletions(-)

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 1234abc..5678def 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -15,11 +15,9 @@ project(libbar CXX)
 
 option(LIBBAR_ENABLE_SIMD "Enable SIMD optimisations" ON)
 
-if(LIBBAR_ENABLE_SIMD)
-    # Hard-coded for native builds only — breaks cross-compilation
-    add_compile_options(-march=native -mtune=native)
-    message(STATUS "libbar: SIMD enabled with -march=native")
-endif()
+# Note: SIMD flags are intentionally omitted here.
+# The Buildroot toolchain wrapper injects the correct -mcpu/-march
+# flags via TARGET_CXXFLAGS. Set LIBBAR_ENABLE_SIMD=OFF if not needed.
 
 set(LIBBAR_SOURCES
     src/parser.cpp
```

### C++ Source Being Patched (`src/parser.cpp` context)

```cpp
// src/parser.cpp — relevant portion showing SIMD guard that still works
// after the patch (the compile flag is gone, but the runtime guard remains)

#include "libbar/parser.hpp"
#include <cstdint>

#ifdef __ARM_NEON
#  include <arm_neon.h>
#  define LIBBAR_HAVE_NEON 1
#endif

#ifdef __SSE4_2__
#  include <nmmintrin.h>
#  define LIBBAR_HAVE_SSE42 1
#endif

namespace libbar {

std::size_t Parser::scan_string(const char *buf, std::size_t len)
{
#if defined(LIBBAR_HAVE_NEON)
    // ARM NEON path — compiler activates when toolchain passes -mfpu=neon
    return scan_string_neon(buf, len);
#elif defined(LIBBAR_HAVE_SSE42)
    // x86 SSE 4.2 path
    return scan_string_sse42(buf, len);
#else
    // Portable scalar fallback — always available
    return scan_string_scalar(buf, len);
#endif
}

} // namespace libbar
```

### `libbar.mk`

```makefile
################################################################################
#
# libbar
#
################################################################################

LIBBAR_VERSION       = 2.0.1
LIBBAR_SITE          = https://github.com/example/libbar/releases/download/v$(LIBBAR_VERSION)
LIBBAR_SOURCE        = libbar-$(LIBBAR_VERSION).tar.gz
LIBBAR_LICENSE       = Apache-2.0
LIBBAR_LICENSE_FILES = LICENSE

LIBBAR_CONF_OPTS  = -DLIBBAR_BUILD_TESTS=OFF
LIBBAR_CONF_OPTS += -DLIBBAR_ENABLE_SIMD=$(if $(BR2_arm)$(BR2_aarch64)$(BR2_x86_64),ON,OFF)

$(eval $(cmake-package))
```

---

## Rust Code Example — Patching a Rust Package

### Scenario

`rutil` is a Rust CLI utility. Its `Cargo.toml` depends on the `ring` crate, which
does not compile for the `armv5te-unknown-linux-musleabi` target without a vendored
assembly patch.

### Patch File: `0001-cargo-feature-gate-ring-for-armv5te.patch`

```diff
From a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 Mon Sep 17 00:00:00 2001
From: Embedded Dev <dev@example.com>
Date: Wed, 03 Jan 2024 14:30:00 +0000
Subject: [PATCH] cargo: feature-gate ring crate for targets lacking BoringSSL ASM

The `ring` crate embeds architecture-specific assembly that does not
assemble on armv5te. Gate it behind a feature flag and fall back to
the pure-Rust `sha2` + `aes` crates for unsupported targets.

Signed-off-by: Embedded Dev <dev@example.com>
---
 Cargo.toml   | 14 +++++++++----
 src/crypto.rs| 22 ++++++++++++++++-----
 2 files changed, 27 insertions(+), 9 deletions(-)

diff --git a/Cargo.toml b/Cargo.toml
index aabbcc1..ddeeff2 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -10,7 +10,12 @@ edition = "2021"
 [dependencies]
 clap       = { version = "4", features = ["derive"] }
 serde      = { version = "1", features = ["derive"] }
-ring       = "0.17"
+ring       = { version = "0.17", optional = true }
+sha2       = { version = "0.10", optional = true }
+aes        = { version = "0.8",  optional = true }
+
+[features]
+default        = ["ring-crypto"]
+ring-crypto    = ["dep:ring"]
+pure-rust-crypto = ["dep:sha2", "dep:aes"]
 
 [profile.release]
 opt-level = "z"   # minimise binary size for embedded targets
diff --git a/src/crypto.rs b/src/crypto.rs
index 1122334..5566778 100644
--- a/src/crypto.rs
+++ b/src/crypto.rs
@@ -1,15 +1,27 @@
-use ring::digest::{Context, SHA256};
-use ring::aead;
+#[cfg(feature = "ring-crypto")]
+mod backend {
+    pub use ring::digest::{Context, SHA256};
+    pub use ring::aead;
 
-pub fn sha256_hex(data: &[u8]) -> String {
-    let mut ctx = Context::new(&SHA256);
-    ctx.update(data);
-    let digest = ctx.finish();
-    hex::encode(digest.as_ref())
+    pub fn sha256_hex(data: &[u8]) -> String {
+        let mut ctx = Context::new(&SHA256);
+        ctx.update(data);
+        let digest = ctx.finish();
+        hex::encode(digest.as_ref())
+    }
+}
+
+#[cfg(feature = "pure-rust-crypto")]
+mod backend {
+    use sha2::{Digest, Sha256};
+
+    pub fn sha256_hex(data: &[u8]) -> String {
+        let result = Sha256::digest(data);
+        hex::encode(result)
+    }
 }
+
+pub use backend::sha256_hex;
```

### `rutil.mk` with Arch-Conditional Feature Flag

```makefile
################################################################################
#
# rutil
#
################################################################################

RUTIL_VERSION  = 0.4.2
RUTIL_SITE     = https://github.com/example/rutil
RUTIL_SITE_METHOD = git
RUTIL_LICENSE  = MIT
RUTIL_LICENSE_FILES = LICENSE

# armv5te and similar old ARM cores cannot compile the `ring` crate's ASM.
# Switch to the pure-Rust crypto backend for those targets.
ifeq ($(BR2_arm)$(BR2_ARCH_IS_64),y)
  # 64-bit or modern 32-bit ARM — ring ASM works
  RUTIL_CARGO_FEATURES = ring-crypto
else ifeq ($(BR2_arm),y)
  # Old 32-bit ARM (armv5te, armv4t, etc.) — use pure Rust
  RUTIL_CARGO_FEATURES = pure-rust-crypto
else
  # All other architectures (x86_64, riscv, etc.) — ring works
  RUTIL_CARGO_FEATURES = ring-crypto
endif

RUTIL_CARGO_BUILD_OPTS = --no-default-features \
                          --features $(RUTIL_CARGO_FEATURES)

$(eval $(cargo-package))
```

### Checking the Patch Was Applied

```bash
# After a build, inspect the patched source
ls output/build/rutil-0.4.2/

# Verify a specific hunk landed
grep -n "pure-rust-crypto" output/build/rutil-0.4.2/src/crypto.rs

# Re-run just the patch step (forces re-extract + re-patch)
make rutil-dirclean rutil-patch
```

---

## Upstream Submission Workflow

Patches maintained only inside Buildroot are a maintenance burden. The goal is
always to get fixes merged upstream so future Buildroot version bumps require no
local patches.

```
  +------------------+
  | 1. Create patch  |   quilt new / git format-patch
  |    in Buildroot  |
  +------------------+
           |
           v
  +------------------+
  | 2. Test locally  |   make <pkg>-dirclean && make <pkg>
  +------------------+
           |
           v
  +------------------+
  | 3. Submit to     |   git send-email / GitHub PR
  |    upstream      |
  +------------------+
           |
           v
  +------------------+
  | 4. Track with    |   Add URL comment in series file
  |    upstream PR   |
  +------------------+
           |
    Merged upstream?
           |
    YES ---+--- NO (long wait)
    |               |
    v               v
  +--------+   +-----------+
  | Remove |   | Keep in   |
  | patch  |   | Buildroot |
  | on ver.|   | with ref  |
  | bump   |   | comment   |
  +--------+   +-----------+
```

### Annotated `series` File with Tracking URLs

```
# libfoo patch series
# Status tracked in: https://github.com/example/libfoo

# Merged upstream in v1.2.4 — remove when we bump to >= 1.2.4
# Upstream: https://github.com/example/libfoo/pull/123
0001-fix-cross-compilation-with-musl.patch

# Submitted, awaiting review
# Upstream: https://github.com/example/libfoo/pull/456
0002-disable-werror-on-clang.patch

# Buildroot-specific: not suitable for upstream (BR2 build system assumption)
0003-add-pkg-config-path.patch
```

### Patch Header Format (for Upstream Submission)

```diff
From abc123def456 Mon Sep 17 00:00:00 2001
From: Your Name <you@example.com>
Date: Fri, 05 Jan 2024 10:00:00 +0000
Subject: [PATCH] subsystem: short imperative description under 72 chars

Longer explanation of WHY the change is needed (not what — the diff
shows what). Wrap at 72 characters. Describe the problem that existed
before this patch.

On embedded targets cross-compiled with musl libc, the call to
getaddrinfo() fails at link time because libfoo does not add -lresolv
to its link flags when the resolver is built into the libc.

Fixes: #789
Link: https://github.com/example/libfoo/issues/789
Signed-off-by: Your Name <you@example.com>
---
 src/net.c | 3 ++-
 1 file changed, 2 insertions(+), 1 deletion(-)
```

---

## Common Pitfalls

### 1. Forgetting `quilt add` Before Editing

```
  Without quilt add:          With quilt add:
  +------------------+        +------------------+
  | Edit file        |        | quilt add file   |
  | quilt refresh    |        | Edit file        |
  | --> EMPTY patch! |        | quilt refresh    |
  +------------------+        | --> correct diff |
                               +------------------+
```

Always run `quilt add <file>` before opening the file in your editor.

### 2. Wrong Strip Level

```bash
# Upstream patch has paths like:
#   a/src/foo.c   b/src/foo.c     --> use -p1 (default)
#   src/foo.c     src/foo.c       --> use -p0 (rare)
#   foo/src/foo.c foo/src/foo.c   --> use -p2

# In series file:
0001-fix.patch -p1    # strip "a/" and "b/" prefixes
0002-legacy.patch -p0 # no stripping
```

### 3. Trailing Whitespace

```bash
# Check for trailing whitespace (will be rejected by many upstreams)
grep -rn ' $' package/libfoo/*.patch

# git format-patch warns automatically:
# warning: squelched N whitespace error(s)
```

### 4. Patch Applies But Build Fails

```bash
# Force a clean rebuild to rule out stale objects
make libfoo-dirclean
make libfoo

# Enable verbose build to see the actual compiler error
make libfoo V=1 2>&1 | tee /tmp/build.log
```

### 5. Patch Rejected After Version Bump

```
  When libfoo 1.2.3 -> 1.3.0:

  +---------------------+          +---------------------+
  | 0001-fix.patch      |          | 0001-fix.patch      |
  | applies cleanly     |          | FAILS: context      |
  | on 1.2.3            |   bump   | lines no longer     |
  |                     |  -----> | match 1.3.0 source  |
  +---------------------+          +---------------------+
                                           |
                                           v
                               Refresh patch against 1.3.0:
                               quilt push && quilt refresh
                               OR remove if merged upstream
```

---

## Summary

Buildroot's patching system is a disciplined, toolchain-friendly way to maintain
local modifications against upstream packages without forking source trees.

```
  Key Components at a Glance
  ============================================================

  package/mypkg/
  |
  +-- series            Ordered list of patches; optional but
  |                     recommended. Controls apply order and
  |                     allows commenting out patches safely.
  |
  +-- 0001-*.patch      First patch. Naming: four-digit prefix,
  |                     hyphenated lowercase description, .patch
  |                     extension. Generated by git format-patch
  |                     or quilt.
  |
  +-- 0002-*.patch      Subsequent patches. Must apply cleanly
  |                     on top of all previous patches.
  |
  +-- mypkg.mk          Can add MYPKG_POST_PATCH_HOOKS or
                        conditionally include arch patches via
                        ifeq($(BR2_arm),y) guards.

  Tools
  ============================================================
  quilt    -- create, edit, refresh patch stacks
  patch    -- low-level application (used by Buildroot internally)
  git      -- format-patch produces quilt-compatible output
  diffstat -- summarise patch size for upstream review emails

  Workflow Summary
  ============================================================

  Identify bug in upstream package
       |
       v
  quilt new 000N-descriptive-name.patch
  quilt add <affected files>
  <edit source>
  quilt refresh
  quilt header -e
       |
       v
  Copy patch to package/<pkg>/
  Add to series file
       |
       v
  Test: make <pkg>-dirclean && make <pkg>
       |
       v
  Submit upstream (git send-email / PR)
  Annotate series file with upstream URL
       |
       v
  When upstream merges: remove patch on next version bump
```

### Conditional Architecture Patching — Decision Tree

```
  Do you need different patches on different CPU architectures?
  |
  +-- YES: Are the patches few and simple?
  |         |
  |         +-- YES: Use ifeq($(BR2_<arch>),y) in .mk file
  |         |         LIBFOO_PATCH += package/libfoo/arch/0010-arm.patch
  |         |
  |         +-- NO:  Use per-arch subdirectories in the package dir
  |                   package/libfoo/arm/0010-*.patch
  |                   + LIBFOO_POST_PATCH_HOOKS to apply them
  |
  +-- NO:  Use flat *.patch files + series file in package dir
           Apply unconditionally; Buildroot auto-discovers them
```

### Key Rules to Remember

| Rule | Reason |
|------|--------|
| Always use `NNNN-desc.patch` naming | Ensures deterministic sort order |
| Always `quilt add` before editing | Without it, diff is empty |
| Use `-p1` (git format) patches | Consistent with Buildroot defaults |
| Add upstream URL to `series` comments | Avoids re-doing already-merged work |
| Remove patches when upstream merges | Reduces future maintenance burden |
| Test with `make <pkg>-dirclean && make <pkg>` | Catches stale build artefacts |
| Never use `-march=native` in patches | Illegal for cross-compilation |
| Feature-gate arch-specific code (Rust) | Cargo features per-arch selection |

---

*Document covers Buildroot 2024.02 LTS. Patch infrastructure has been stable
since Buildroot 2016.05; the Rust `cargo-package` infrastructure was added in
Buildroot 2018.11.*