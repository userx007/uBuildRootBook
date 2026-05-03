# Buildroot License Compliance & Legal Manifest

> `make legal-info` · `*_LICENSE` · `*_LICENSE_FILES` · SPDX Identifiers · GPL Boundary Management · Export Control

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [make legal-info — Overview](#2-make-legal-info--overview)
3. [Package Metadata Fields](#3-package-metadata-fields)
4. [SPDX Identifiers](#4-spdx-identifiers)
5. [GPL Boundary Management](#5-gpl-boundary-management)
6. [C Code Examples](#6-c-code-examples)
7. [C++ Code Examples](#7-c-code-examples-1)
8. [Rust Code Examples](#8-rust-code-examples)
9. [Export Control](#9-export-control)
10. [CI/CD Workflow Integration](#10-cicd-workflow-integration)
11. [Quick Reference Checklists](#11-quick-reference-checklists)
12. [Summary](#12-summary)

---

## 1. Introduction

Embedded Linux products built with Buildroot must satisfy two overlapping legal obligations:

- **Open-source license compliance** — delivering source code, notices, and attribution as mandated by GPL, LGPL, MIT, Apache, and other licenses.
- **Export control** — ensuring cryptographic or dual-use software is correctly classified under EAR (USA), UK Strategic Export Controls, and EU Dual-Use Regulation.

Buildroot provides a first-class infrastructure — the **legal-info subsystem** — to help teams meet both obligations without manual auditing. This document covers the subsystem end to end: its Makefile mechanics, per-package metadata fields, SPDX identifiers, GPL boundary analysis patterns, and practical C, C++, and Rust code examples that demonstrate compliance-aware programming.

---

## 2. `make legal-info` — Overview

Running the command below instructs Buildroot to collect every package's license metadata into a structured legal manifest inside the build output:

```bash
$ make legal-info

# Output is written to:
#   output/legal-info/
#   ├── licenses/         <- full text of every identified license
#   ├── sources/          <- source archives for GPL/LGPL packages
#   ├── manifest.csv      <- machine-readable per-package summary
#   └── README            <- human-readable overview
```

The target is defined in `support/legal-info/legal.mk` and iterates over `PACKAGES`, calling per-package hooks registered by `package/*.mk` files.

### 2.1 Manifest CSV Structure

`manifest.csv` contains one row per package:

| Column | Example Value | Description |
|---|---|---|
| `PACKAGE` | `openssl` | Canonical Buildroot package name |
| `VERSION` | `3.3.1` | Upstream version string |
| `LICENSE` | `Apache-2.0` | SPDX expression (see §4) |
| `LICENSE_FILES` | `LICENSE` | Relative paths inside source tree |
| `SOURCE` | `openssl-3.3.1.tar.gz` | Downloaded source archive name |
| `URI` | `https://openssl.org/…` | Canonical download URI |
| `REDISTRIBUTABLE` | `yes` | `yes` if source must be shipped |

---

## 3. Package Metadata Fields

Each package `.mk` file must declare a minimum set of variables. The legal-info subsystem reads these at build time.

### 3.1 Required Fields

```makefile
# package/myapp/myapp.mk

MYAPP_VERSION         = 1.4.2
MYAPP_SITE            = https://example.com/releases
MYAPP_SOURCE          = myapp-$(MYAPP_VERSION).tar.gz

# ── License metadata ──────────────────────────────────────────────────────
MYAPP_LICENSE         = GPL-2.0-only
MYAPP_LICENSE_FILES   = COPYING

# Optional: list additional files whose text must be shipped
MYAPP_LICENSE_FILES  += docs/THIRD-PARTY-NOTICES.txt

# Declare that CPE vulnerability tracking is active
MYAPP_CPE_ID_VENDOR   = example
MYAPP_CPE_ID_PRODUCT  = myapp

$(eval $(cmake-package))
```

### 3.2 Field Semantics

| Variable | Required? | Purpose |
|---|---|---|
| `*_LICENSE` | Yes | SPDX license expression for the package as a whole |
| `*_LICENSE_FILES` | Yes | Whitespace-separated list of license text files inside the source tree |
| `*_REDISTRIBUTE` | No | Set to `NO` only for closed-source packages with explicit permission |
| `*_ACTUAL_SOURCE_TARBALL` | No | Override when the tarball name differs from the manifest name |
| `*_IGNORE_CVES` | No | Space-separated CVE IDs to suppress from the NVD report |

---

## 4. SPDX Identifiers

Buildroot requires that `*_LICENSE` values be valid **SPDX 2.3 expressions**. This enables automated tooling (FOSSA, Black Duck, Fossology) to parse manifests without heuristic matching.

### 4.1 Common Identifiers Used in Buildroot

| SPDX ID | Full Name | Copyleft Strength | Typical Use |
|---|---|---|---|
| `GPL-2.0-only` | GNU GPL v2 only | Strong | Linux kernel modules, BusyBox |
| `GPL-2.0-or-later` | GNU GPL v2+ | Strong | Many GNU packages |
| `LGPL-2.1-only` | GNU Lesser GPL v2.1 | Weak | glibc, libxml2 |
| `LGPL-2.1-or-later` | GNU Lesser GPL v2.1+ | Weak | GLib, GTK |
| `MIT` | MIT License | None | curl, jq, many libs |
| `Apache-2.0` | Apache License 2.0 | None | OpenSSL 3.x, Rust crates |
| `BSD-2-Clause` | Simplified BSD | None | FreeBSD userland ports |
| `BSD-3-Clause` | New BSD | None | Many networking tools |
| `MPL-2.0` | Mozilla Public License 2.0 | File-level | Firefox, NSS |

### 4.2 SPDX Compound Expressions

When a package contains files under different licenses, use SPDX boolean operators:

```makefile
# Two equally applicable licenses (choose either)
LIBFOO_LICENSE = MIT OR Apache-2.0

# All licenses apply simultaneously
LIBBAR_LICENSE = LGPL-2.1-or-later AND MIT

# File-level exception
LIBBAZ_LICENSE = GPL-2.0-only WITH Classpath-exception-2.0
```

---

## 5. GPL Boundary Management

GPL boundary management is the practice of structuring your software so that proprietary components are not tainted by the GPL's copyleft requirement. The key question is: **does your proprietary code form a combined work with GPL-licensed code?**

### 5.1 Boundary Topology

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       Buildroot Root Filesystem                         │
  │                                                                         │
  │   ┌────────────────┐   ┌────────────────┐   ┌──────────────────────┐    │
  │   │  Linux Kernel  │   │    BusyBox     │   │  Proprietary App     │    │
  │   │  (GPL-2.0-only)│   │  (GPL-2.0-only)│   │  (Closed Source)     │    │
  │   └───────┬────────┘   └───────┬────────┘   └──────────┬───────────┘    │
  │           │ syscall ABI        │ execve()              │                │
  │           │ (NOT copyleft)     │ (NOT copyleft)        │                │
  │           ▼                    ▼                       │                │
  │   ┌─────────────────────────────────────┐              │                │
  │   │          C Library (glibc)          │◄──── links ──┤                │
  │   │        (LGPL-2.1-or-later)          │              │                │
  │   └─────────────────────────────────────┘              │                │
  │           ▲                                            │                │
  │           │ dynamic linking (LGPL OK with conditions)  │                │
  │           └────────────────────────────────────────────┘                │
  │                                                                         │
  │   KEY:  ══════  GPL boundary (must NOT be crossed by proprietary code)  │
  │          ──────  Safe interface (syscall, execve, dynamic LGPL link)    │
  └─────────────────────────────────────────────────────────────────────────┘

  RULE: Proprietary code may call glibc via dynamic linking (LGPL §6).
  RULE: Proprietary code must NOT statically link GPL code.
  RULE: Proprietary kernel modules that are derived works MUST be GPL.
```

### 5.2 Static vs. Dynamic Linking Decision Tree

```
  Is the library GPL?
         │
         ├─── NO (MIT/BSD/Apache/LGPL) ──────────────────────────────────────►
         │                                                  Static or dynamic
         │                                                  link both OK
         │
         └─── YES
                 │
                 ├─── Is it GPL-2.0-only?
                 │          │
                 │          ├─── YES ──► MUST release YOUR source too
                 │          │            (static or dynamic — copyleft applies)
                 │          │
                 │          └─── NO (LGPL) ──► Dynamic link is OK for
                 │                              proprietary code (LGPL §6)
                 │                              Static link: must allow
                 │                              relinking by user
                 │
                 └─── Are you writing a kernel module?
                            │
                            └─── YES ──► GPL-2.0-only required
                                         (kernel ABI is NOT a clean boundary)
```

---

## 6. C Code Examples

### 6.1 License-Compliant LGPL Dynamic Linking

This pattern demonstrates correct use of an LGPL library from a proprietary C application — satisfying the LGPL §6 requirement that the user can re-link with a modified library.

```c
/* myapp/src/main.c  —  links dynamically against libz (zlib, MIT licensed) */
/* and libcurl (MIT/curl license) — both safe for proprietary code           */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <zlib.h>      /* zlib: MIT — dynamic link OK                        */
#include <curl/curl.h> /* libcurl: curl license (MIT-like) — dynamic OK     */

/* ── SPDX license header (best practice for every source file) ─────────── */
/* SPDX-FileCopyrightText: 2024 ACME Corp.                                  */
/* SPDX-License-Identifier: LicenseRef-ACME-Proprietary                     */

static size_t write_cb(void *ptr, size_t sz, size_t nmemb, void *buf) {
    size_t total = sz * nmemb;
    strncat((char *)buf, (char *)ptr, total);
    return total;
}

int fetch_and_compress(const char *url, const char *outfile) {
    CURL   *curl;
    CURLcode res;
    char    buf[65536] = {0};
    gzFile  gz;

    curl_global_init(CURL_GLOBAL_DEFAULT);
    curl = curl_easy_init();
    if (!curl) { return 1; }

    curl_easy_setopt(curl, CURLOPT_URL,           url);
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_cb);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA,     buf);

    res = curl_easy_perform(curl);
    curl_easy_cleanup(curl);
    curl_global_cleanup();

    if (res != CURLE_OK) { return 2; }

    /* zlib gzip write — MIT library, dynamic link: no copyleft concern */
    gz = gzopen(outfile, "wb9");
    if (!gz) { return 3; }
    gzwrite(gz, buf, (unsigned)strlen(buf));
    gzclose(gz);

    printf("Wrote compressed output to %s\n", outfile);
    return 0;
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <url> <outfile.gz>\n", argv[0]);
        return EXIT_FAILURE;
    }
    return fetch_and_compress(argv[1], argv[2]);
}
```

### 6.2 GPL-Compliant Kernel Module Stub

Any Loadable Kernel Module (LKM) that interfaces with GPL-only exported kernel symbols must carry a GPL license declaration.

```c
/* drivers/mydrv/mydrv.c — kernel module: GPL-2.0-only mandatory            */
/* SPDX-License-Identifier: GPL-2.0-only                                    */
/* SPDX-FileCopyrightText: 2024 ACME Corp.                                  */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define DEVICE_NAME  "mydrv"
#define BUF_LEN      256

static int    major;
static char   msg[BUF_LEN];
static int    msg_len;

static ssize_t dev_read(struct file *f, char __user *buf,
                         size_t len, loff_t *off) {
    int bytes = min((int)len, msg_len);
    if (copy_to_user(buf, msg, bytes)) return -EFAULT;
    return bytes;
}

static struct file_operations fops = { .read = dev_read };

static int __init mydrv_init(void) {
    major = register_chrdev(0, DEVICE_NAME, &fops);
    snprintf(msg, BUF_LEN, "Hello from GPL module\n");
    msg_len = strlen(msg);
    pr_info("%s: registered with major %d\n", DEVICE_NAME, major);
    return 0;
}

static void __exit mydrv_exit(void) {
    unregister_chrdev(major, DEVICE_NAME);
}

module_init(mydrv_init);
module_exit(mydrv_exit);

/* These three macros are checked by modinfo and the kernel at load time */
MODULE_LICENSE("GPL v2");          /* MUST match SPDX in source header    */
MODULE_AUTHOR("ACME Corp.");
MODULE_DESCRIPTION("Example GPL-2.0-only character device driver");
```

---

## 7. C++ Code Examples

### 7.1 License Manifest Reader (C++17)

A production utility that parses Buildroot's `manifest.csv` and flags packages whose licenses require source redistribution.

```cpp
// tools/license_checker/main.cpp
// SPDX-License-Identifier: Apache-2.0
// SPDX-FileCopyrightText: 2024 ACME Corp.

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <vector>
#include <unordered_set>
#include <algorithm>

struct Package {
    std::string name;
    std::string version;
    std::string license;
    std::string license_files;
    bool        redistributable{false};
};

// SPDX IDs that trigger source redistribution obligations
static const std::unordered_set<std::string> COPYLEFT_IDS = {
    "GPL-1.0-only", "GPL-1.0-or-later",
    "GPL-2.0-only", "GPL-2.0-or-later",
    "GPL-3.0-only", "GPL-3.0-or-later",
    "AGPL-3.0-only", "AGPL-3.0-or-later",
};

bool is_copyleft(const std::string &spdx) {
    // Tokenise on ' ', 'AND', 'OR', 'WITH', '(', ')'
    std::istringstream ss(spdx);
    std::string token;
    while (ss >> token) {
        // Strip parentheses
        token.erase(std::remove_if(token.begin(), token.end(),
                    [](char c){ return c == '(' || c == ')'; }),
                    token.end());
        if (COPYLEFT_IDS.count(token)) return true;
    }
    return false;
}

std::vector<Package> parse_manifest(const std::string &path) {
    std::vector<Package> pkgs;
    std::ifstream f(path);
    std::string line;
    bool header = true;
    while (std::getline(f, line)) {
        if (header) { header = false; continue; }  // skip CSV header
        std::istringstream ss(line);
        Package p;
        std::string redist;
        // CSV columns: PACKAGE,VERSION,LICENSE,LICENSE_FILES,SOURCE,URI,REDISTRIBUTABLE
        std::getline(ss, p.name,          ',');
        std::getline(ss, p.version,       ',');
        std::getline(ss, p.license,       ',');
        std::getline(ss, p.license_files, ',');
        std::string src, uri;
        std::getline(ss, src,    ',');
        std::getline(ss, uri,    ',');
        std::getline(ss, redist, ',');
        p.redistributable = (redist == "yes");
        pkgs.push_back(std::move(p));
    }
    return pkgs;
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        std::cerr << "Usage: license_checker <manifest.csv>\n";
        return 1;
    }
    auto pkgs = parse_manifest(argv[1]);
    int issues = 0;
    for (auto &p : pkgs) {
        bool copyleft = is_copyleft(p.license);
        if (copyleft && !p.redistributable) {
            std::cout << "[WARN] " << p.name << " (" << p.license
                      << ") is copyleft but marked non-redistributable\n";
            ++issues;
        }
    }
    std::cout << "Checked " << pkgs.size() << " packages, "
              << issues << " issue(s) found.\n";
    return issues > 0 ? 2 : 0;
}
```

### 7.2 GPL Boundary Guard Using Shared Library Isolation

This pattern shows how a C++ application isolates GPL functionality behind a well-defined shared-library boundary, preventing copyleft from propagating into proprietary code.

```cpp
// ── gpl_plugin/gpl_codec.h  (published interface — MIT licensed) ─────────
// SPDX-License-Identifier: MIT
#pragma once
#include <cstdint>
#include <vector>

extern "C" {
    // Only this interface is exposed across the GPL boundary.
    // The implementation (gpl_codec.cpp) is GPL and lives in a separate .so
    int  gpl_encode(const uint8_t *in,  size_t in_len,
                    uint8_t       *out, size_t *out_len);
    void gpl_free  (void *ptr);
}

// ── gpl_plugin/gpl_codec.cpp  (GPL-2.0-or-later — separate shared lib) ──
// SPDX-License-Identifier: GPL-2.0-or-later
#include "gpl_codec.h"
#include <lame/lame.h>   // libmp3lame — GPL
#include <cstdlib>
#include <cstring>

int gpl_encode(const uint8_t *in,  size_t in_len,
               uint8_t       *out, size_t *out_len) {
    lame_t lame = lame_init();
    lame_set_in_samplerate(lame, 44100);
    lame_set_VBR(lame, vbr_default);
    lame_init_params(lame);

    int rc = lame_encode_buffer_interleaved(
        lame,
        (short *)in,   (int)(in_len / (2 * sizeof(short))),
        out,           (int)*out_len
    );
    lame_close(lame);
    if (rc < 0) return rc;
    *out_len = (size_t)rc;
    return 0;
}
void gpl_free(void *ptr) { free(ptr); }

// ── proprietary_app/encoder_facade.cpp  (Proprietary) ───────────────────
// SPDX-License-Identifier: LicenseRef-ACME-Proprietary
#include <dlfcn.h>
#include <stdexcept>
#include <vector>
#include <cstdint>

class EncoderFacade {
    void *lib_{nullptr};
    using EncFn  = int(*)(const uint8_t*, size_t, uint8_t*, size_t*);
    using FreeFn = void(*)(void*);
    EncFn  enc_{nullptr};
    FreeFn free_{nullptr};
public:
    EncoderFacade() {
        // Runtime dynamic load — the GPL .so is only referenced at runtime
        lib_ = dlopen("libgpl_codec.so", RTLD_LAZY);
        if (!lib_) throw std::runtime_error(dlerror());
        enc_  = (EncFn) dlsym(lib_, "gpl_encode");
        free_ = (FreeFn)dlsym(lib_, "gpl_free");
    }
    ~EncoderFacade() { if (lib_) dlclose(lib_); }

    std::vector<uint8_t> encode(const std::vector<uint8_t> &pcm) {
        std::vector<uint8_t> out(pcm.size() * 2);
        size_t out_len = out.size();
        if (enc_(pcm.data(), pcm.size(), out.data(), &out_len) < 0)
            throw std::runtime_error("encode failed");
        out.resize(out_len);
        return out;
    }
};
```

---

## 8. Rust Code Examples

### 8.1 Buildroot Package `.mk` for a Rust Binary

Rust packages use the `cargo-package` infrastructure. License metadata follows the same conventions.

```makefile
# package/my-rust-app/my-rust-app.mk
# SPDX-License-Identifier: GPL-2.0-or-later

MY_RUST_APP_VERSION       = 0.9.1
MY_RUST_APP_SITE          = $(call github,acme,my-rust-app,v$(MY_RUST_APP_VERSION))
MY_RUST_APP_SITE_METHOD   = git

# Crate-level license — check Cargo.toml AND crates.io
MY_RUST_APP_LICENSE       = Apache-2.0 OR MIT
MY_RUST_APP_LICENSE_FILES = LICENSE-APACHE LICENSE-MIT

# Cargo.lock vendored dependencies are also checked by legal-info
MY_RUST_APP_CARGO_VENDOR  = YES

$(eval $(cargo-package))
```

### 8.2 SPDX Header Embedding & SBOM Generator (Rust)

SPDX file-level comments are supported in Rust via the standard `//` syntax and are scanned by tools such as `reuse-tool` and fossology.

```rust
// SPDX-FileCopyrightText: 2024 ACME Corp. <legal@acme.example>
// SPDX-License-Identifier: Apache-2.0 OR MIT

//! `license_report` — reads Buildroot legal-info output and
//! generates a JSON SBOM fragment (CycloneDX 1.5 subset).

use std::{
    collections::HashMap,
    fs::File,
    io::{BufRead, BufReader},
    path::Path,
};

use serde::{Deserialize, Serialize};  // serde: MIT OR Apache-2.0

#[derive(Debug, Serialize, Deserialize)]
pub struct Component {
    pub name:            String,
    pub version:         String,
    pub licenses:        Vec<String>,
    pub redistributable: bool,
}

/// Parse `manifest.csv` produced by `make legal-info`.
pub fn parse_manifest<P: AsRef<Path>>(path: P) -> anyhow::Result<Vec<Component>> {
    let file = File::open(path)?;
    let reader = BufReader::new(file);
    let mut components = Vec::new();
    let mut first = true;

    for line in reader.lines() {
        let line = line?;
        if first { first = false; continue; }  // skip header
        let cols: Vec<&str> = line.splitn(7, ',').collect();
        if cols.len() < 7 { continue; }
        components.push(Component {
            name:            cols[0].to_string(),
            version:         cols[1].to_string(),
            licenses:        cols[2].split_whitespace()
                                     .map(str::to_string)
                                     .collect(),
            redistributable: cols[6].trim() == "yes",
        });
    }
    Ok(components)
}

/// Return components whose license is a strong copyleft identifier.
pub fn filter_copyleft(components: &[Component]) -> Vec<&Component> {
    const COPYLEFT: &[&str] = &[
        "GPL-2.0-only", "GPL-2.0-or-later",
        "GPL-3.0-only", "GPL-3.0-or-later",
        "AGPL-3.0-only", "AGPL-3.0-or-later",
    ];
    components
        .iter()
        .filter(|c| c.licenses.iter().any(|l| COPYLEFT.contains(&l.as_str())))
        .collect()
}

fn main() -> anyhow::Result<()> {
    let path = std::env::args().nth(1)
        .unwrap_or_else(|| "output/legal-info/manifest.csv".into());
    let components = parse_manifest(&path)?;
    let copyleft   = filter_copyleft(&components);

    println!("Total packages : {}", components.len());
    println!("Copyleft items : {}", copyleft.len());
    for c in &copyleft {
        println!("  {:30}  {}", c.name, c.licenses.join(" "));
    }

    // Emit CycloneDX-like JSON fragment
    let json = serde_json::to_string_pretty(&components)?;
    std::fs::write("sbom-fragment.json", json)?;
    println!("\nSBOM fragment written to sbom-fragment.json");
    Ok(())
}
```

### 8.3 `Cargo.toml` License Metadata

Rust packages declare license metadata in `Cargo.toml`. Buildroot's `cargo-package` infrastructure reads this during `legal-info`.

```toml
[package]
name        = "license-report"
version     = "0.9.1"
edition     = "2021"
authors     = ["ACME Corp. <legal@acme.example>"]
license     = "Apache-2.0 OR MIT"
description = "Buildroot legal-info manifest parser and SBOM generator"

# Dual-licensed crates: both files must be present in the repo
license-file = "LICENSE-APACHE"   # primary; MIT in LICENSE-MIT

[dependencies]
serde      = { version = "1", features = ["derive"] }  # MIT OR Apache-2.0
serde_json = "1"                                        # MIT OR Apache-2.0
anyhow     = "1"                                        # MIT OR Apache-2.0

# No GPL dependencies — crate remains Apache-2.0 OR MIT
```

---

## 9. Export Control

Buildroot packages may include cryptographic software subject to export regulations. Two key packages declare their status:

```makefile
# package/openssl/openssl.mk (excerpt)
OPENSSL_LICENSE       = Apache-2.0
OPENSSL_LICENSE_FILES = LICENSE.txt

# Export control annotation — triggers notice in legal manifest
OPENSSL_REDISTRIBUTE  = YES

# Inform Buildroot that this package contains encryption
OPENSSL_CPE_ID_VENDOR  = openssl
OPENSSL_CPE_ID_PRODUCT = openssl
```

### 9.1 EAR Classification Flow

```
  Package contains cryptography?
        │
        ├── NO  ──────────────────────────────────► EAR99 (no license needed)
        │
        └── YES
               │
               ├── Publicly available (OSI-approved license, posted on internet)?
               │          │
               │          └── YES ──────────────────► TSU exception (15 CFR 740.13)
               │                                       Notify BIS once at first export
               │                                       URL: submit to crypt@bis.doc.gov
               │
               └── Restricted / proprietary crypto?
                          │
                          ├── Mass-market (strength ≤ 64-bit key, retail)?
                          │          └── YES ──► License Exception ENC retail
                          │
                          └── Strong / military-grade?
                                     └── YES ──► Export License required (BIS review)

  EU Dual-Use Reg. 428/2009 Annex I, Category 5 Part 2 mirrors the above.
  UK Strategic Export Control List (SECL) applies post-Brexit.
```

### 9.2 ECCN Annotation Convention

Although Buildroot has no native ECCN field, teams can extend the manifest by post-processing it with a mapping table:

```makefile
# support/legal-info/eccn-annotate.mk  (custom addition)

ECCN_MAP := openssl:5D002 libgcrypt:5D002 libssh2:5D002 gnupg:5D002

legal-eccn: legal-info
    @python3 support/scripts/eccn_annotate.py \
        output/legal-info/manifest.csv \
        "$(ECCN_MAP)" \
        output/legal-info/manifest-eccn.csv
    @echo "ECCN-annotated manifest: output/legal-info/manifest-eccn.csv"

.PHONY: legal-eccn
```

---

## 10. CI/CD Workflow Integration

Integrating license compliance into a CI pipeline ensures violations are caught before release, not during an audit.

### 10.1 Complete Compliance Pipeline

```
  Developer pushes tag
        │
        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  CI Job: buildroot-compliance                                   │
  │                                                                 │
  │  1. make legal-info                                             │
  │       └──► output/legal-info/manifest.csv                       │
  │                                                                 │
  │  2. license_checker manifest.csv          (C++ tool, §6.1)      │
  │       └──► exit 0 OK  /  exit 2 FAIL                            │
  │                                                                 │
  │  3. license-report output/legal-info/manifest.csv               │
  │       └──► sbom-fragment.json             (Rust tool, §8.2)     │
  │                                                                 │
  │  4. reuse lint                                                  │
  │       └──► checks every source file has SPDX header             │
  │                                                                 │
  │  5. fossology-scanner sbom-fragment.json  (optional)            │
  │       └──► deep license text analysis                           │
  │                                                                 │
  │  6. eccn-annotate  ──► manifest-eccn.csv  (export control)      │
  │                                                                 │
  │  7. Package: output/legal-info.tar.gz  ──► artifact store       │
  └─────────────────────────────────────────────────────────────────┘
        │
        ▼
  Release gate: all steps pass?
       YES ──► Tag approved for release
       NO  ──► Block release, file compliance ticket
```

---

## 11. Quick Reference Checklists

### 11.1 Per-Package Checklist

| Item | Command / File | Status Check |
|---|---|---|
| `*_LICENSE` is a valid SPDX expression | `grep _LICENSE package/*.mk` | `spdx-tools validate` |
| `*_LICENSE_FILES` lists all license texts | `ls $(PKG_BUILD_DIR)/$(LIC_FILES)` | File must exist in source |
| Source archive downloadable | `make <pkg>-source` | SHA256 in `.hash` file |
| SPDX header in each source file | `reuse lint` | CI step |
| GPL modules use `MODULE_LICENSE` | `modinfo ./module.ko` | Must show `GPL` |
| Rust `Cargo.toml` has `license` field | `cargo metadata --format-version 1` | `license` key present |
| ECCN annotation added | `manifest-eccn.csv` | All crypto pkgs tagged |

### 11.2 Key `make` Targets

| Target | Description |
|---|---|
| `make legal-info` | Full legal manifest, license texts, source archives |
| `make <pkg>-legal-info` | Legal info for a single package |
| `make legal-info BR2_LEGAL_INFO_MANIFEST_CSV=1` | Force CSV output format |
| `make show-info` | JSON dump of all package metadata (includes license) |
| `make graph-depends` | Dependency graph — helps visualise GPL boundaries |

---

## 12. Summary

> **Key Takeaways**

1. `make legal-info` is the single authoritative source of truth for all package license metadata in a Buildroot image.
2. Every package `.mk` file **MUST** declare `*_LICENSE` (SPDX) and `*_LICENSE_FILES`.
3. SPDX expressions enable automated toolchain integration (FOSSA, Black Duck, reuse-tool).
4. GPL copyleft does **NOT** propagate across the syscall ABI (kernel ↔ userspace) or process boundaries (`execve`).
5. LGPL libraries may be dynamically linked from proprietary code, provided the user can re-link (LGPL §6).
6. Kernel modules that use GPL-only symbols **MUST** carry `MODULE_LICENSE("GPL")`.
7. Cryptographic packages must be assessed under EAR/TSU exception rules before export.
8. Embed SPDX file headers (`SPDX-License-Identifier` + `SPDX-FileCopyrightText`) in every source file.
9. Rust crates declare licenses in `Cargo.toml`; Buildroot's `cargo-package` infrastructure honours this.
10. Automate the full compliance pipeline in CI: `legal-info` → SPDX lint → SBOM → ECCN annotation → gate.

---

*Buildroot License Compliance & Legal Manifest — Topic 31*