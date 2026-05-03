Now I'll create the comprehensive markdown document:Here's what the document covers across its ~500 lines:

**Structure of the document:**

- **Introduction** — why overlays exist and what problem they solve
- **What Is a Filesystem Overlay?** — concept with ASCII merge diagram
- **`BR2_ROOTFS_OVERLAY`** — the variable, `menuconfig` location, and how Buildroot's internal `rsync -a` applies it
- **Directory Structure & Merge Order** — single and multi-overlay layouts, left-to-right precedence with ASCII step diagrams, recommended multi-board tree layout, and permission handling
- **OverlayFS at Runtime** — kernel config, ASCII upper/lower/work/merged architecture diagram, mount command, fstab entry, init script, and multi-lower-layer stacking (kernel ≥ 4.0)
- **Per-Board Configuration Differences** — four strategies (separate dirs, post-build script, template substitution, `BR2_EXTERNAL`)

**Code examples (8 total):**

| # | Lang | What it does |
|---|------|--------------|
| 1 | C | Reads `/etc/hwconfig.conf` placed by overlay |
| 2 | C | Mounts OverlayFS via `mount(2)` syscall |
| 3 | C | Scans upper dir and reports whiteout files |
| 4 | C++ | Typed board-variant config manager (INI parser) |
| 5 | C++ | `statfs`-based overlay detector + `/proc/mounts` dump |
| 6 | Rust | TOML board config reader with `serde` |
| 7 | Rust | OverlayFS mount helper using the `nix` crate |
| 8 | Rust | Diff reporter comparing overlay dir vs target rootfs |

**Summary** — ASCII reference table covering build-time vs runtime layering, per-board strategies, key rules, and a language integration overview.

# 16. Filesystem Overlays in Buildroot

> **Topic:** `BR2_ROOTFS_OVERLAY` directories, merge order, overlayfs at runtime,
> and managing per-board config differences.

---

## Table of Contents

1. [Introduction](#introduction)
2. [What Is a Filesystem Overlay?](#what-is-a-filesystem-overlay)
3. [BR2_ROOTFS_OVERLAY — The Buildroot Variable](#br2_rootfs_overlay--the-buildroot-variable)
4. [Directory Structure & Merge Order](#directory-structure--merge-order)
5. [OverlayFS at Runtime](#overlayfs-at-runtime)
6. [Per-Board Configuration Differences](#per-board-configuration-differences)
7. [C Code Examples](#c-code-examples)
8. [C++ Code Examples](#c-code-examples-1)
9. [Rust Code Examples](#rust-code-examples)
10. [Summary](#summary)

---

## Introduction

In embedded Linux development with **Buildroot**, the root filesystem is typically
assembled from package outputs and a base skeleton. However, real-world products
nearly always require:

- Custom configuration files (`/etc/network/interfaces`, `/etc/hostname`, etc.)
- Board-specific init scripts
- Per-variant application configurations
- Factory calibration data or certificates

Rather than patching every file through a package or post-build script,
Buildroot provides **filesystem overlays**: directories whose contents are
merged verbatim over the assembled root filesystem at build time.

---

## What Is a Filesystem Overlay?

A filesystem overlay is a directory tree that mirrors the target root filesystem
layout. During the Buildroot build process, after all packages are installed,
the contents of each overlay directory are **copied on top** of the staging
root filesystem.

```
Overlay concept (build time):

  [Package outputs]          [Overlay dir]
  /target/                   /board/myboard/rootfs-overlay/
  ├── bin/                   ├── etc/
  ├── etc/                   │   ├── hostname
  │   └── fstab              │   └── network/
  ├── lib/                   │       └── interfaces
  └── usr/                   └── usr/
                                 └── local/
                                     └── bin/
                                         └── my-app.sh

             ↓  rsync / cp merge

  /target/ (final)
  ├── bin/
  ├── etc/
  │   ├── fstab          ← from packages
  │   ├── hostname       ← from overlay (NEW)
  │   └── network/
  │       └── interfaces ← from overlay (NEW)
  ├── lib/
  └── usr/
      └── local/
          └── bin/
              └── my-app.sh ← from overlay (NEW)
```

Files in the overlay **replace** existing files of the same path; the overlay
does **not** merge file contents — it overwrites at the file level.

---

## BR2_ROOTFS_OVERLAY — The Buildroot Variable

### Setting the variable

In your Buildroot `.config` (or `menuconfig` under
`System configuration → Root filesystem overlay directories`):

```
BR2_ROOTFS_OVERLAY="board/myboard/rootfs-overlay"
```

Multiple paths are **space-separated**:

```
BR2_ROOTFS_OVERLAY="board/common/rootfs-overlay board/myboard/rootfs-overlay"
```

### In `menuconfig`

```
System configuration
  └── (board/myboard/rootfs-overlay) Root filesystem overlay directories
```

### Equivalent in a fragment config

```makefile
# configs/myboard_defconfig  (fragment)
BR2_ROOTFS_OVERLAY="board/myboard/rootfs-overlay"
```

### How Buildroot applies it

Internally Buildroot executes (simplified):

```bash
for overlay in ${BR2_ROOTFS_OVERLAY}; do
    rsync -a --exclude='.empty' --exclude='.*' \
        "${overlay}/" "${TARGET_DIR}/"
done
```

The `rsync -a` preserves permissions, symlinks, and timestamps — critical for
`setuid` binaries and device nodes that must arrive with specific modes.

---

## Directory Structure & Merge Order

### Single overlay layout

```
board/
└── myboard/
    └── rootfs-overlay/
        ├── etc/
        │   ├── hostname          # replaces /etc/hostname
        │   ├── inittab           # replaces /etc/inittab
        │   └── network/
        │       └── interfaces    # adds /etc/network/interfaces
        ├── lib/
        │   └── firmware/
        │       └── wifi.bin      # adds firmware blob
        └── usr/
            └── local/
                └── bin/
                    └── monitor   # adds custom binary
```

### Multiple overlays — merge order matters

```
BR2_ROOTFS_OVERLAY="board/common/overlay board/myboard/overlay board/myboard-v2/overlay"
```

Merge is **left-to-right**: later overlays win.

```
Merge sequence:

  Step 1: Apply board/common/overlay
  ┌─────────────────────┐
  │  target/            │
  │  └── etc/           │
  │      └── hostname   │  ← "common-board"
  └─────────────────────┘

  Step 2: Apply board/myboard/overlay  (WINS over common)
  ┌─────────────────────┐
  │  target/            │
  │  └── etc/           │
  │      └── hostname   │  ← "myboard"  (overwrites "common-board")
  └─────────────────────┘

  Step 3: Apply board/myboard-v2/overlay  (WINS over myboard)
  ┌─────────────────────┐
  │  target/            │
  │  └── etc/           │
  │      └── hostname   │  ← "myboard-v2"  (final value)
  └─────────────────────┘
```

### Recommended multi-board directory layout

```
board/
├── common/
│   └── rootfs-overlay/          # files shared across ALL boards
│       ├── etc/
│       │   ├── profile.d/
│       │   │   └── aliases.sh
│       │   └── sysctl.conf
│       └── usr/
│           └── local/
│               └── bin/
│                   └── health-check
├── product-A/
│   └── rootfs-overlay/          # product-A specifics
│       └── etc/
│           ├── hostname         # "product-a"
│           └── hwconfig.conf
├── product-A-rev2/
│   └── rootfs-overlay/          # hardware rev differences only
│       └── lib/
│           └── firmware/
│               └── nic-v2.bin
└── product-B/
    └── rootfs-overlay/
        └── etc/
            ├── hostname         # "product-b"
            └── hwconfig.conf
```

Corresponding defconfigs:

```makefile
# configs/product_a_defconfig
BR2_ROOTFS_OVERLAY="board/common/rootfs-overlay board/product-A/rootfs-overlay"

# configs/product_a_rev2_defconfig
BR2_ROOTFS_OVERLAY="board/common/rootfs-overlay board/product-A/rootfs-overlay \
                    board/product-A-rev2/rootfs-overlay"

# configs/product_b_defconfig
BR2_ROOTFS_OVERLAY="board/common/rootfs-overlay board/product-B/rootfs-overlay"
```

### File permission handling in overlays

```bash
# Device nodes must be listed in a device table, NOT placed in the overlay.
# For regular files, set permissions before build:

chmod 755 board/myboard/rootfs-overlay/usr/local/bin/monitor
chmod 600 board/myboard/rootfs-overlay/etc/secret.key
chmod 640 board/myboard/rootfs-overlay/etc/app/config.ini

# Alternatively use BR2_ROOTFS_DEVICE_TABLE for special nodes.
```

---

## OverlayFS at Runtime

Build-time overlays handle static differences. For **runtime** overlays —
where a read-only base system is combined with a writable upper layer —
Linux **OverlayFS** (`overlay` kernel module) is used.

### Kernel configuration prerequisite

```
CONFIG_OVERLAY_FS=y          # built-in (or =m for module)
CONFIG_OVERLAY_FS_REDIRECT_DIR=y
CONFIG_OVERLAY_FS_INDEX=y
```

### OverlayFS structure

```
Runtime OverlayFS layout:

  ┌──────────────────────────────────────────┐
  │            MERGED (mount point)          │
  │   /mnt/root  ← what processes see       │
  └────────────────┬─────────────────────────┘
                   │
       ┌───────────┴────────────┐
       │                        │
  ┌────┴──────┐          ┌──────┴──────┐
  │  UPPER    │          │   LOWER     │
  │ /mnt/rw   │          │  /mnt/ro    │
  │ (writable)│          │ (read-only) │
  │ tmpfs or  │          │ squashfs or │
  │ ext4 on   │          │ ext4 (ro)   │
  │ flash     │          │ from eMMC   │
  └───────────┘          └─────────────┘
       │
  ┌────┴──────┐
  │  WORK     │
  │ /mnt/work │
  │ (internal │
  │  scratch) │
  └───────────┘

Read:   upper wins if file exists there, otherwise falls through to lower
Write:  copy-on-write into upper; lower unchanged
Delete: whiteout file created in upper; lower unchanged
```

### Mount command

```bash
# Manual mount (init script or fstab)
mount -t overlay overlay \
    -o lowerdir=/mnt/ro,upperdir=/mnt/rw,workdir=/mnt/work \
    /mnt/root
```

### `/etc/fstab` entry (place via overlay)

```
# /etc/fstab
overlay  /        overlay  lowerdir=/rom,upperdir=/overlay/upper,workdir=/overlay/work  0 0
tmpfs    /overlay  tmpfs   size=64m,mode=755                                             0 0
```

### Init script for overlayfs (S00overlay)

Place this via the build-time overlay at
`board/myboard/rootfs-overlay/etc/init.d/S00overlay`:

```sh
#!/bin/sh
# S00overlay — mount overlayfs early in boot

LOWER=/rom
UPPER=/mnt/overlay/upper
WORK=/mnt/overlay/work
MERGED=/

case "$1" in
  start)
    echo "Setting up OverlayFS..."
    mkdir -p "$UPPER" "$WORK"
    mount -t overlay overlay \
        -o "lowerdir=${LOWER},upperdir=${UPPER},workdir=${WORK}" \
        "$MERGED"
    ;;
  stop)
    umount "$MERGED"
    ;;
  *)
    echo "Usage: $0 {start|stop}"
    exit 1
esac
```

### OverlayFS with multiple lower layers (kernel ≥ 4.0)

```bash
# Stack multiple read-only layers: rightmost is lowest priority
mount -t overlay overlay \
    -o lowerdir=/layer-high:/layer-mid:/layer-low,upperdir=/upper,workdir=/work \
    /merged

# Example: OTA update as a new lower layer, factory image as base
mount -t overlay overlay \
    -o lowerdir=/mnt/ota-update:/mnt/factory,upperdir=/mnt/rw/overlay,workdir=/mnt/rw/work \
    /
```

```
Multi-lower OverlayFS (runtime, e.g. OTA + factory):

  ┌──────────────────────────────────────────────────┐
  │                   MERGED /                       │
  └──────────────────────────────────────────────────┘
         ↑ priority (highest to lowest)
  ┌──────────┐  ┌────────────┐  ┌────────────────┐
  │  UPPER   │  │  LOWER[0]  │  │   LOWER[1]     │
  │  /mnt/rw │  │ /mnt/ota   │  │ /mnt/factory   │
  │ writable │  │ OTA update │  │ base image     │
  └──────────┘  └────────────┘  └────────────────┘
```

---

## Per-Board Configuration Differences

### Strategy 1: Separate overlay directories (cleanest)

Each board has its own overlay with only board-specific files:

```
board/
├── common/rootfs-overlay/etc/sysctl.conf          # all boards
├── product-A/rootfs-overlay/etc/hostname           # "product-a"
└── product-B/rootfs-overlay/etc/hostname           # "product-b"
```

### Strategy 2: Post-build script selecting overlay

```makefile
# board/myboard/post-build.sh
#!/bin/bash
BOARD_VARIANT="${1:-default}"
OVERLAY="board/myboard/rootfs-overlay-${BOARD_VARIANT}"
if [ -d "${OVERLAY}" ]; then
    rsync -a "${OVERLAY}/" "${TARGET_DIR}/"
fi
```

```makefile
# In defconfig:
BR2_ROOTFS_POST_BUILD_SCRIPT="board/myboard/post-build.sh"
BR2_ROOTFS_POST_BUILD_SCRIPT_ARGS="rev2"
```

### Strategy 3: Template substitution via post-build script

Place template files in the overlay, process them in post-build:

```
board/myboard/rootfs-overlay/etc/hostname.tmpl:
  ${BOARD_HOSTNAME}

board/myboard/post-build.sh:
  sed "s/\${BOARD_HOSTNAME}/${BOARD_HOSTNAME}/" \
      "${TARGET_DIR}/etc/hostname.tmpl" \
      > "${TARGET_DIR}/etc/hostname"
  rm "${TARGET_DIR}/etc/hostname.tmpl"
```

### Strategy 4: BR2_EXTERNAL with multiple board overlays

```
my-external/
├── Config.in
├── external.mk
├── configs/
│   ├── board_a_defconfig
│   └── board_b_defconfig
└── board/
    ├── common/
    │   └── rootfs-overlay/
    ├── board-a/
    │   └── rootfs-overlay/
    └── board-b/
        └── rootfs-overlay/
```

```makefile
# my-external/configs/board_a_defconfig
BR2_EXTERNAL_MY_EXTERNAL_PATH="$(BR2_EXTERNAL_MY_EXTERNAL_PATH)"
BR2_ROOTFS_OVERLAY="${BR2_EXTERNAL_MY_EXTERNAL_PATH}/board/common/rootfs-overlay \
                    ${BR2_EXTERNAL_MY_EXTERNAL_PATH}/board/board-a/rootfs-overlay"
```

---

## C Code Examples

### Example 1: Reading board identity at runtime (from overlay-provided file)

A C application reads `/etc/hwconfig.conf` placed via the overlay and
configures itself per-board.

```c
/* board_config.c
 * Reads /etc/hwconfig.conf installed via BR2_ROOTFS_OVERLAY
 * Compile: gcc -o board_config board_config.c
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define HWCONFIG_PATH  "/etc/hwconfig.conf"
#define MAX_LINE       256
#define MAX_KEY        64
#define MAX_VAL        192

typedef struct {
    char board_name[MAX_VAL];
    char hw_revision[MAX_VAL];
    int  num_leds;
    int  uart_baud;
} BoardConfig;

static int parse_hwconfig(const char *path, BoardConfig *cfg)
{
    FILE *fp;
    char line[MAX_LINE];
    char key[MAX_KEY], val[MAX_VAL];

    memset(cfg, 0, sizeof(*cfg));
    cfg->uart_baud = 115200;   /* sensible default */

    fp = fopen(path, "r");
    if (!fp) {
        fprintf(stderr, "Cannot open %s: %s\n", path, strerror(errno));
        return -1;
    }

    while (fgets(line, sizeof(line), fp)) {
        /* skip comments and blank lines */
        if (line[0] == '#' || line[0] == '\n')
            continue;

        if (sscanf(line, "%63[^=]=%191[^\n]", key, val) != 2)
            continue;

        /* trim trailing whitespace from key */
        char *p = key + strlen(key) - 1;
        while (p > key && (*p == ' ' || *p == '\t')) *p-- = '\0';

        if (strcmp(key, "BOARD_NAME") == 0)
            strncpy(cfg->board_name, val, MAX_VAL - 1);
        else if (strcmp(key, "HW_REVISION") == 0)
            strncpy(cfg->hw_revision, val, MAX_VAL - 1);
        else if (strcmp(key, "NUM_LEDS") == 0)
            cfg->num_leds = atoi(val);
        else if (strcmp(key, "UART_BAUD") == 0)
            cfg->uart_baud = atoi(val);
    }

    fclose(fp);
    return 0;
}

int main(void)
{
    BoardConfig cfg;

    if (parse_hwconfig(HWCONFIG_PATH, &cfg) != 0)
        return EXIT_FAILURE;

    printf("Board    : %s\n", cfg.board_name);
    printf("HW Rev   : %s\n", cfg.hw_revision);
    printf("LEDs     : %d\n", cfg.num_leds);
    printf("UART baud: %d\n", cfg.uart_baud);

    return EXIT_SUCCESS;
}
```

The `/etc/hwconfig.conf` placed in `board/product-A/rootfs-overlay/etc/hwconfig.conf`:

```ini
# Hardware configuration — Product A Rev 1
BOARD_NAME=product-a
HW_REVISION=1.0
NUM_LEDS=4
UART_BAUD=115200
```

And in `board/product-A-rev2/rootfs-overlay/etc/hwconfig.conf`:

```ini
# Hardware configuration — Product A Rev 2
BOARD_NAME=product-a
HW_REVISION=2.0
NUM_LEDS=8
UART_BAUD=921600
```

---

### Example 2: Detecting and mounting OverlayFS in C (init helper)

```c
/* overlay_mount.c
 * Early userspace helper to set up OverlayFS.
 * Link with: gcc -o overlay_mount overlay_mount.c
 *
 * ASCII diagram of what we build:
 *
 *   tmpfs /overlay
 *   ├── upper/    (writable layer)
 *   └── work/     (overlayfs scratch)
 *
 *   overlay mount → /  (merged view)
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mount.h>
#include <sys/stat.h>
#include <errno.h>

#define TMPFS_MOUNT   "/overlay"
#define UPPER_DIR     "/overlay/upper"
#define WORK_DIR      "/overlay/work"
#define LOWER_DIR     "/rom"
#define MERGED_DIR    "/mnt/root"

static int make_dir(const char *path)
{
    if (mkdir(path, 0755) == -1 && errno != EEXIST) {
        fprintf(stderr, "mkdir %s: %s\n", path, strerror(errno));
        return -1;
    }
    return 0;
}

int main(void)
{
    char opts[512];

    /* 1. Mount tmpfs to hold upper + work dirs */
    if (mount("tmpfs", TMPFS_MOUNT, "tmpfs", 0, "size=32m,mode=755") == -1) {
        fprintf(stderr, "mount tmpfs: %s\n", strerror(errno));
        return EXIT_FAILURE;
    }
    printf("[overlay_mount] tmpfs mounted at %s\n", TMPFS_MOUNT);

    /* 2. Create upper and work directories */
    if (make_dir(UPPER_DIR) || make_dir(WORK_DIR) || make_dir(MERGED_DIR))
        return EXIT_FAILURE;

    /* 3. Compose overlayfs options */
    snprintf(opts, sizeof(opts),
             "lowerdir=%s,upperdir=%s,workdir=%s",
             LOWER_DIR, UPPER_DIR, WORK_DIR);

    /* 4. Mount overlayfs */
    if (mount("overlay", MERGED_DIR, "overlay", 0, opts) == -1) {
        fprintf(stderr, "mount overlay: %s\n", strerror(errno));
        return EXIT_FAILURE;
    }

    printf("[overlay_mount] OverlayFS mounted:\n");
    printf("  lower  = %s  (read-only base)\n", LOWER_DIR);
    printf("  upper  = %s  (writable layer)\n", UPPER_DIR);
    printf("  work   = %s  (scratch)\n",         WORK_DIR);
    printf("  merged = %s  (visible to system)\n", MERGED_DIR);

    return EXIT_SUCCESS;
}
```

---

### Example 3: Checking overlay whiteout files (diagnostic tool)

```c
/* check_whiteouts.c
 * Lists whiteout files in an OverlayFS upper directory.
 * Whiteouts mark files deleted from the lower layer.
 *
 * ASCII whiteout concept:
 *
 *   upper/
 *   ├── modified_file      ← copy-on-write of lower file
 *   ├── new_file           ← created in upper only
 *   └── .wh.deleted_file   ← whiteout: hides lower/deleted_file
 *
 * Compile: gcc -o check_whiteouts check_whiteouts.c
 */

#include <stdio.h>
#include <string.h>
#include <dirent.h>
#include <sys/stat.h>
#include <linux/fs.h>   /* for OVL_WHITEOUT_DEV */

#define WHITEOUT_PREFIX  ".wh."

static void scan_dir(const char *path, int depth)
{
    DIR *dir;
    struct dirent *ent;
    struct stat st;
    char full[1024];
    int i;

    dir = opendir(path);
    if (!dir) return;

    while ((ent = readdir(dir)) != NULL) {
        if (strcmp(ent->d_name, ".") == 0 ||
            strcmp(ent->d_name, "..") == 0)
            continue;

        snprintf(full, sizeof(full), "%s/%s", path, ent->d_name);

        for (i = 0; i < depth * 2; i++) putchar(' ');

        if (strncmp(ent->d_name, WHITEOUT_PREFIX,
                    strlen(WHITEOUT_PREFIX)) == 0) {
            /* It's a whiteout */
            printf("[WHITEOUT] %s  -> hides lower: /%s\n",
                   ent->d_name,
                   ent->d_name + strlen(WHITEOUT_PREFIX));
        } else {
            lstat(full, &st);
            if (S_ISDIR(st.st_mode)) {
                printf("[DIR ] %s/\n", ent->d_name);
                scan_dir(full, depth + 1);
            } else {
                printf("[FILE] %s  (%lld bytes)\n",
                       ent->d_name, (long long)st.st_size);
            }
        }
    }
    closedir(dir);
}

int main(int argc, char *argv[])
{
    const char *upper = (argc > 1) ? argv[1] : "/overlay/upper";
    printf("Scanning OverlayFS upper dir: %s\n\n", upper);
    scan_dir(upper, 0);
    return 0;
}
```

---

## C++ Code Examples

### Example 4: Board variant manager using overlay config files

```cpp
// board_variant.cpp
// Loads per-board config from /etc/board-variant.json (placed via overlay)
// and exposes typed configuration to the application.
//
// Compile: g++ -std=c++17 -o board_variant board_variant.cpp
//
// File placed via overlay: board/product-A/rootfs-overlay/etc/board-variant.conf
// (using a simple INI-style format to avoid JSON library deps)

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <map>
#include <stdexcept>
#include <algorithm>

class BoardVariant {
public:
    explicit BoardVariant(const std::string& path = "/etc/board-variant.conf")
    {
        load(path);
    }

    std::string get(const std::string& key,
                    const std::string& default_val = "") const
    {
        auto it = config_.find(key);
        return (it != config_.end()) ? it->second : default_val;
    }

    int getInt(const std::string& key, int default_val = 0) const
    {
        auto it = config_.find(key);
        if (it == config_.end()) return default_val;
        try { return std::stoi(it->second); }
        catch (...) { return default_val; }
    }

    bool getBool(const std::string& key, bool default_val = false) const
    {
        std::string v = get(key, "");
        std::transform(v.begin(), v.end(), v.begin(), ::tolower);
        if (v == "true" || v == "1" || v == "yes") return true;
        if (v == "false" || v == "0" || v == "no") return false;
        return default_val;
    }

    void dump() const
    {
        std::cout << "┌─ Board Variant Configuration ──────────────┐\n";
        for (const auto& [k, v] : config_)
            std::cout << "│  " << k << " = " << v << "\n";
        std::cout << "└────────────────────────────────────────────┘\n";
    }

private:
    void load(const std::string& path)
    {
        std::ifstream f(path);
        if (!f.is_open())
            throw std::runtime_error("Cannot open " + path);

        std::string line;
        while (std::getline(f, line)) {
            // strip comments
            auto comment = line.find('#');
            if (comment != std::string::npos)
                line = line.substr(0, comment);

            // trim whitespace
            auto start = line.find_first_not_of(" \t\r\n");
            if (start == std::string::npos) continue;
            auto end = line.find_last_not_of(" \t\r\n");
            line = line.substr(start, end - start + 1);

            auto eq = line.find('=');
            if (eq == std::string::npos) continue;

            std::string key = line.substr(0, eq);
            std::string val = line.substr(eq + 1);

            // trim key/val
            auto trim = [](std::string& s) {
                auto a = s.find_first_not_of(" \t");
                auto b = s.find_last_not_of(" \t");
                s = (a == std::string::npos) ? "" : s.substr(a, b - a + 1);
            };
            trim(key); trim(val);
            if (!key.empty())
                config_[key] = val;
        }
    }

    std::map<std::string, std::string> config_;
};

// ---- Application using board variant config ----

class Application {
public:
    explicit Application(const BoardVariant& bv) : bv_(bv) {}

    void configure()
    {
        std::string name = bv_.get("BOARD_NAME", "unknown");
        int leds         = bv_.getInt("NUM_LEDS", 0);
        bool has_wifi    = bv_.getBool("HAS_WIFI", false);
        int baud         = bv_.getInt("UART_BAUD", 115200);

        std::cout << "Configuring for board: " << name << "\n"
                  << "  LEDs     : " << leds      << "\n"
                  << "  WiFi     : " << (has_wifi ? "yes" : "no") << "\n"
                  << "  UART baud: " << baud       << "\n";
    }

private:
    const BoardVariant& bv_;
};

int main()
{
    try {
        BoardVariant bv("/etc/board-variant.conf");
        bv.dump();

        Application app(bv);
        app.configure();
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }
    return 0;
}
```

Overlay file `board/product-A/rootfs-overlay/etc/board-variant.conf`:

```ini
# Product A Rev 1 board variant
BOARD_NAME    = product-a-rev1
HW_REVISION   = 1.0
NUM_LEDS      = 4
HAS_WIFI      = true
UART_BAUD     = 115200
DISPLAY_WIDTH = 800
DISPLAY_HEIGHT= 480
```

---

### Example 5: Runtime overlayfs stat utility (C++)

```cpp
// overlay_stat.cpp
// Reports filesystem type and overlayfs details of a given path.
//
// Compile: g++ -std=c++17 -o overlay_stat overlay_stat.cpp
//
// ASCII view of what this tool reveals:
//
//   /  (overlay)
//   ├── /proc/mounts entry:
//   │     overlay / overlay rw,lowerdir=/rom,upperdir=/ov/u,workdir=/ov/w 0 0
//   └── statfs type = 0x794c7630  (OVERLAY_SUPER_MAGIC)

#include <iostream>
#include <fstream>
#include <string>
#include <sys/statfs.h>
#include <linux/magic.h>

#ifndef OVERLAYFS_SUPER_MAGIC
#define OVERLAYFS_SUPER_MAGIC  0x794c7630
#endif

struct FsInfo {
    std::string type_name;
    bool is_overlay;
    bool is_readonly;
    long long block_size;
    long long total_blocks;
    long long free_blocks;
};

static FsInfo stat_fs(const std::string& path)
{
    struct statfs sfs {};
    FsInfo info {};

    if (statfs(path.c_str(), &sfs) != 0) {
        throw std::runtime_error("statfs failed for " + path);
    }

    info.block_size   = sfs.f_bsize;
    info.total_blocks = sfs.f_blocks;
    info.free_blocks  = sfs.f_bfree;

    switch (sfs.f_type) {
        case OVERLAYFS_SUPER_MAGIC: info.type_name = "overlayfs"; info.is_overlay = true; break;
        case TMPFS_MAGIC:           info.type_name = "tmpfs";     break;
        case EXT2_SUPER_MAGIC:      info.type_name = "ext2/3/4";  break;
        case SQUASHFS_MAGIC:        info.type_name = "squashfs";  info.is_readonly = true; break;
        default:
            info.type_name = "unknown(0x" +
                std::to_string(sfs.f_type) + ")";
    }

    return info;
}

static void show_overlay_mounts()
{
    std::ifstream mf("/proc/mounts");
    std::string line;
    bool found = false;

    std::cout << "\n  OverlayFS mount entries from /proc/mounts:\n";
    std::cout << "  ┌────────────────────────────────────────────────────┐\n";

    while (std::getline(mf, line)) {
        if (line.find("overlay") != std::string::npos) {
            std::cout << "  │ " << line << "\n";
            found = true;
        }
    }

    if (!found)
        std::cout << "  │  (none found)\n";

    std::cout << "  └────────────────────────────────────────────────────┘\n";
}

int main(int argc, char* argv[])
{
    std::string path = (argc > 1) ? argv[1] : "/";

    try {
        auto info = stat_fs(path);

        std::cout << "Filesystem info for: " << path << "\n";
        std::cout << "  Type       : " << info.type_name << "\n";
        std::cout << "  Is overlay : " << (info.is_overlay  ? "YES" : "no") << "\n";
        std::cout << "  Read-only  : " << (info.is_readonly ? "YES" : "no") << "\n";
        std::cout << "  Block size : " << info.block_size << " bytes\n";
        std::cout << "  Total      : "
                  << (info.total_blocks * info.block_size / 1024 / 1024)
                  << " MiB\n";
        std::cout << "  Free       : "
                  << (info.free_blocks * info.block_size / 1024 / 1024)
                  << " MiB\n";

        if (info.is_overlay)
            show_overlay_mounts();

    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }
    return 0;
}
```

---

## Rust Code Examples

### Example 6: Board config reader in Rust (reads overlay-provided `/etc/board.toml`)

```toml
# Cargo.toml
[package]
name    = "board-config"
version = "0.1.0"
edition = "2021"

[dependencies]
serde       = { version = "1", features = ["derive"] }
toml        = "0.8"
```

```rust
// src/main.rs
//! Reads /etc/board.toml placed via BR2_ROOTFS_OVERLAY
//! and reports board configuration.
//!
//! ASCII flow:
//!
//!   Build time:
//!     board/product-A/rootfs-overlay/etc/board.toml ──► target/etc/board.toml
//!
//!   Runtime:
//!     /etc/board.toml ──► BoardConfig struct ──► application logic

use serde::Deserialize;
use std::fs;
use std::path::Path;
use std::process;

#[derive(Debug, Deserialize)]
struct BoardConfig {
    board: BoardInfo,
    hardware: HardwareInfo,
    network: Option<NetworkInfo>,
}

#[derive(Debug, Deserialize)]
struct BoardInfo {
    name: String,
    revision: String,
    serial_prefix: String,
}

#[derive(Debug, Deserialize)]
struct HardwareInfo {
    num_leds: u8,
    has_wifi: bool,
    uart_baud: u32,
    display: Option<DisplayInfo>,
}

#[derive(Debug, Deserialize)]
struct DisplayInfo {
    width: u32,
    height: u32,
    interface: String,   // "hdmi", "mipi-dsi", "lvds"
}

#[derive(Debug, Deserialize)]
struct NetworkInfo {
    hostname: String,
    dhcp: bool,
    fallback_ip: Option<String>,
}

fn load_config<P: AsRef<Path>>(path: P) -> Result<BoardConfig, String> {
    let content = fs::read_to_string(&path)
        .map_err(|e| format!("Cannot read {:?}: {}", path.as_ref(), e))?;

    toml::from_str(&content)
        .map_err(|e| format!("Parse error in {:?}: {}", path.as_ref(), e))
}

fn print_config(cfg: &BoardConfig) {
    println!("┌─ Board Configuration ──────────────────────────┐");
    println!("│ Board    : {} rev {}",
             cfg.board.name, cfg.board.revision);
    println!("│ S/N pfx  : {}", cfg.board.serial_prefix);
    println!("│ LEDs     : {}", cfg.hardware.num_leds);
    println!("│ WiFi     : {}", if cfg.hardware.has_wifi { "yes" } else { "no" });
    println!("│ UART baud: {}", cfg.hardware.uart_baud);

    if let Some(disp) = &cfg.hardware.display {
        println!("│ Display  : {}x{} ({})",
                 disp.width, disp.height, disp.interface);
    }

    if let Some(net) = &cfg.network {
        println!("│ Hostname : {}", net.hostname);
        println!("│ DHCP     : {}", if net.dhcp { "yes" } else { "no" });
        if let Some(ip) = &net.fallback_ip {
            println!("│ Fallback : {}", ip);
        }
    }

    println!("└────────────────────────────────────────────────┘");
}

fn main() {
    let config_path = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "/etc/board.toml".to_string());

    match load_config(&config_path) {
        Ok(cfg) => {
            print_config(&cfg);
        }
        Err(e) => {
            eprintln!("Error: {}", e);
            process::exit(1);
        }
    }
}
```

Overlay file `board/product-A/rootfs-overlay/etc/board.toml`:

```toml
[board]
name          = "product-a"
revision      = "1.0"
serial_prefix = "PA1"

[hardware]
num_leds  = 4
has_wifi  = true
uart_baud = 115200

[hardware.display]
width     = 800
height    = 480
interface = "lvds"

[network]
hostname    = "product-a"
dhcp        = true
fallback_ip = "192.168.1.100"
```

---

### Example 7: OverlayFS mount helper in Rust

```toml
# Cargo.toml
[package]
name    = "overlay-mount"
version = "0.1.0"
edition = "2021"

[dependencies]
nix = { version = "0.27", features = ["mount", "fs"] }
```

```rust
// src/main.rs
//! Mounts an OverlayFS programmatically from Rust.
//!
//! Requires root. Typical use: called from initramfs before pivot_root.
//!
//! ASCII structure built by this program:
//!
//!   /overlay  (tmpfs)
//!   ├── upper/    <─── writable copy-on-write layer
//!   └── work/     <─── overlayfs internal scratch dir
//!
//!   /mnt/root  (merged view, overlay mount point)
//!   ├── [lower layer contents — read-only]
//!   └── [upper layer files — writable, win on conflict]

use nix::mount::{mount, MsFlags};
use std::fs;
use std::path::Path;
use std::process;

const TMPFS_BASE : &str = "/overlay";
const UPPER_DIR  : &str = "/overlay/upper";
const WORK_DIR   : &str = "/overlay/work";
const LOWER_DIR  : &str = "/rom";
const MERGED_DIR : &str = "/mnt/root";

fn mkdir_p(path: &str) -> Result<(), String> {
    fs::create_dir_all(Path::new(path))
        .map_err(|e| format!("mkdir_p {}: {}", path, e))
}

fn do_mount(
    src: Option<&str>,
    target: &str,
    fstype: Option<&str>,
    flags: MsFlags,
    data: Option<&str>,
) -> Result<(), String> {
    mount(src, target, fstype, flags, data)
        .map_err(|e| format!("mount {} -> {}: {}", src.unwrap_or("none"), target, e))
}

fn main() {
    // 1. Mount tmpfs to hold upper + work
    if let Err(e) = mkdir_p(TMPFS_BASE) {
        eprintln!("{}", e);
        process::exit(1);
    }

    if let Err(e) = do_mount(
        Some("tmpfs"), TMPFS_BASE, Some("tmpfs"),
        MsFlags::empty(), Some("size=32m,mode=755"),
    ) {
        eprintln!("{}", e);
        process::exit(1);
    }
    println!("[overlay-mount] tmpfs mounted at {}", TMPFS_BASE);

    // 2. Create upper, work, and merged directories
    for dir in &[UPPER_DIR, WORK_DIR, MERGED_DIR] {
        if let Err(e) = mkdir_p(dir) {
            eprintln!("{}", e);
            process::exit(1);
        }
    }

    // 3. Build overlay options string
    let opts = format!(
        "lowerdir={},upperdir={},workdir={}",
        LOWER_DIR, UPPER_DIR, WORK_DIR
    );

    // 4. Mount overlayfs
    if let Err(e) = do_mount(
        Some("overlay"), MERGED_DIR, Some("overlay"),
        MsFlags::empty(), Some(opts.as_str()),
    ) {
        eprintln!("{}", e);
        process::exit(1);
    }

    println!("[overlay-mount] OverlayFS mounted:");
    println!("  lower  = {}  (read-only squashfs base)", LOWER_DIR);
    println!("  upper  = {}  (writable tmpfs layer)",    UPPER_DIR);
    println!("  work   = {}  (scratch space)",           WORK_DIR);
    println!("  merged = {}  (live filesystem)",         MERGED_DIR);
}
```

---

### Example 8: Overlay diff reporter in Rust

```rust
// overlay_diff.rs — standalone script (no external deps)
//! Compares an overlay directory to the target rootfs and reports
//! which files will be added, replaced, or new.
//!
//! Usage: overlay_diff <overlay_dir> <target_dir>
//!
//! ASCII output example:
//!
//!   Overlay: board/product-A/rootfs-overlay
//!   Target:  output/target
//!
//!   [NEW    ] etc/hostname
//!   [REPLACE] etc/inittab
//!   [NEW    ] usr/local/bin/monitor
//!   [REPLACE] lib/firmware/wifi.bin

use std::env;
use std::fs;
use std::path::{Path, PathBuf};
use std::process;

#[derive(Debug, PartialEq)]
enum FileStatus {
    New,
    Replace,
}

struct OverlayEntry {
    rel_path: PathBuf,
    status:   FileStatus,
}

fn walk_overlay(
    overlay_root: &Path,
    target_root:  &Path,
    current:      &Path,
    entries:      &mut Vec<OverlayEntry>,
) -> std::io::Result<()> {
    for entry in fs::read_dir(current)? {
        let entry = entry?;
        let path  = entry.path();
        let rel   = path.strip_prefix(overlay_root).unwrap().to_path_buf();
        let in_target = target_root.join(&rel);

        if path.is_dir() {
            walk_overlay(overlay_root, target_root, &path, entries)?;
        } else {
            let status = if in_target.exists() {
                FileStatus::Replace
            } else {
                FileStatus::New
            };
            entries.push(OverlayEntry { rel_path: rel, status });
        }
    }
    Ok(())
}

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() != 3 {
        eprintln!("Usage: {} <overlay_dir> <target_dir>", args[0]);
        process::exit(1);
    }

    let overlay_dir = Path::new(&args[1]);
    let target_dir  = Path::new(&args[2]);

    println!("Overlay: {}", overlay_dir.display());
    println!("Target:  {}", target_dir.display());
    println!();

    let mut entries: Vec<OverlayEntry> = Vec::new();

    if let Err(e) = walk_overlay(overlay_dir, target_dir, overlay_dir, &mut entries) {
        eprintln!("Error: {}", e);
        process::exit(1);
    }

    entries.sort_by(|a, b| a.rel_path.cmp(&b.rel_path));

    let mut new_count     = 0usize;
    let mut replace_count = 0usize;

    for e in &entries {
        let tag = match e.status {
            FileStatus::New     => { new_count     += 1; "NEW    " }
            FileStatus::Replace => { replace_count += 1; "REPLACE" }
        };
        println!("[{}] {}", tag, e.rel_path.display());
    }

    println!();
    println!("Summary: {} new file(s), {} replacement(s), {} total",
             new_count, replace_count, entries.len());
}
```

---

## Summary

```
Buildroot Filesystem Overlays — Complete Picture
================================================

BUILD TIME (static overlay)
────────────────────────────

  Source tree                            Buildroot target
  ──────────                             ────────────────
  board/common/rootfs-overlay/    ──┐
  board/myboard/rootfs-overlay/   ──┼──► rsync -a (left-to-right) ──► output/target/
  board/myboard-v2/rootfs-overlay ──┘

  Controlled by: BR2_ROOTFS_OVERLAY="path1 path2 path3"
  Later paths win on conflict. Permissions preserved by rsync.


RUNTIME (dynamic overlay — OverlayFS)
──────────────────────────────────────

  /rom  (squashfs, read-only)           Kernel OverlayFS driver
  /overlay/upper  (tmpfs, writable)  ──► merged view at /
  /overlay/work   (tmpfs, scratch)

  Operations:
    Read  → upper first, then lower (transparent to processes)
    Write → copy-on-write into upper; lower untouched
    Delete→ whiteout file in upper hides lower file


PER-BOARD MANAGEMENT STRATEGIES
──────────────────────────────────

  Strategy               Use when
  ──────────────────────────────────────────────────────────────
  Separate overlay dirs  Clean split; different defconfigs
  Post-build script      Need scripted logic / template vars
  Template substitution  Parameterised config files
  BR2_EXTERNAL           Large multi-product external trees


KEY RULES
──────────────────────────────────

  1. Files in later overlays REPLACE (not merge) earlier files
  2. Use rsync-friendly permissions before committing overlay files
  3. Device nodes → BR2_ROOTFS_DEVICE_TABLE, not overlay dirs
  4. OverlayFS upper dir needs a work dir on the SAME filesystem
  5. Multiple lower dirs (kernel ≥4.0): lowerdir=high:mid:low
  6. Whiteout files (.wh.<name>) mark lower-layer deletions


LANGUAGE INTEGRATION SUMMARY
─────────────────────────────

  Language  | Overlay use-case
  ──────────────────────────────────────────────────────────────
  C         | Read /etc configs, mount overlayfs via syscall
  C++       | Typed config manager, statfs/overlayfs diagnostics
  Rust      | Safe config parsing (TOML/INI), nix-based mount
```

The Buildroot filesystem overlay system is elegant in its simplicity: at build
time, `BR2_ROOTFS_OVERLAY` applies static, board-specific content with no
runtime overhead. At runtime, Linux OverlayFS extends the same layering concept
dynamically, enabling writable roots on top of read-only base images — essential
for OTA updates, factory reset capability, and flash longevity. Together they
form a robust two-tier customisation strategy that scales cleanly from a single
board to dozens of product variants managed from a single Buildroot tree.