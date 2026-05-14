# Read-Only Root & Persistent Data Strategies

**Structure of the document:**

- **Architecture overview** — ASCII partition map showing eMMC layout (bootloader, Kernel A/B, Rootfs A/B, Data) and how SquashFS + OverlayFS + tmpfs assemble into one unified VFS tree
- **SquashFS** — Buildroot config, fstab snippets, dm-verity integration
- **OverlayFS** — ASCII copy-on-write diagram, mount options, workdir rules
- **tmpfs** — fstab entries for `/var/log`, `/tmp`, `/run`, size budgeting advice
- **Persistent /data partition** — ASCII directory tree, first-boot init script
- **A/B update slots** — full ASCII state machine (Slot A active → OTA → Slot B pending → commit or rollback), U-Boot env scripting
- **Buildroot config** — annotated defconfig fragment + post-build shell script
- **C examples** — overlay pivot_root setup, config read with template fallback, A/B slot management via libubootenv
- **C++ examples** — RAII `Mount` wrapper, atomic `rename()`-based config write, OTA `ABUpdater` class
- **Rust examples** — `/proc/mounts` overlay verifier, `serde_json` atomic config persistence, boot watchdog/commit daemon
- **Init integration** — BusyBox inittab + rcS approach and systemd `.mount` units
- **Security** — dm-verity, threat model summary table
- **Summary** — ASCII comparison table (component / role / flash writes) + key takeaways


> `squashfs` + `overlayfs` · `tmpfs` mounts for `/var` · data partition mounting · A/B update slot design

---

## Table of Contents

1. [Introduction & Motivation](#1-introduction--motivation)
2. [Filesystem Architecture Overview](#2-filesystem-architecture-overview)
3. [SquashFS — The Read-Only Root](#3-squashfs--the-read-only-root)
4. [OverlayFS — Writable Union Layer](#4-overlayfs--writable-union-layer)
5. [tmpfs Mounts for /var and /tmp](#5-tmpfs-mounts-for-var-and-tmp)
6. [Persistent Data Partition](#6-persistent-data-partition)
7. [A/B Update Slot Design](#7-ab-update-slot-design)
8. [Buildroot Configuration](#8-buildroot-configuration)
9. [C Code Examples](#9-c-code-examples)
10. [C++ Code Examples](#10-c-code-examples-1)
11. [Rust Code Examples](#11-rust-code-examples)
12. [Init System Integration](#12-init-system-integration)
13. [Security Considerations](#13-security-considerations)
14. [Summary](#14-summary)

---

## 1. Introduction & Motivation

Embedded Linux systems — industrial controllers, routers, medical devices, automotive ECUs — run from flash storage with limited write endurance and exposure to unexpected power loss at any moment. A conventional read-write root filesystem is a liability in this environment: every write accelerates flash wear, and an interrupted write can leave the filesystem corrupt and unbootable.

The industry-standard answer combines three complementary technologies:

- A compressed, optionally integrity-verified, **read-only root filesystem** (SquashFS).
- A **copy-on-write overlay** (OverlayFS) that makes the union appear writable to processes without ever touching the original image.
- **Explicit, separate storage** for data that genuinely needs to survive reboots.

Adding **A/B partition slots** on top gives atomic, fail-safe OTA updates with automatic rollback — no downtime, no bricks.

---

## 2. Filesystem Architecture Overview

### 2.1 eMMC / SD Card Partition Layout

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │ Bootloader │ Kernel A  │ Rootfs A   │ Rootfs B   │ Data             │
  │ (U-Boot)   │ zImage    │ squashfs   │ squashfs   │ ext4             │
  │ ~512 KB    │ ~8 MB     │ ~64 MB     │ ~64 MB     │ remainder        │
  └─────────────────────────────────────────────────────────────────────┘
        ▲             ▲           ▲                         ▲
        │             │           │                         │
     MTD/MMC       /boot     /dev/mmcblk0p3           /dev/mmcblk0p5
```

### 2.2 Linux VFS — Mount-Time Assembly

```
  squashfs (ro)              tmpfs (rw, RAM)            ext4 /data (rw)
  ─────────────────          ──────────────────         ───────────────
  /bin                       /tmp                       /data/
  /lib                       /run                       /data/config/
  /usr                       /var/log                   /data/db/
  /etc  (ro templates)       /var/tmp                   /data/certs/
  /sbin                      /var/cache
       │                          │
       └───────── overlayfs ──────┘
                      │
               /  (as seen by all processes)
```

### 2.3 OverlayFS Layer Stack

```
  ┌────────────────────────────────────────────────────────────┐
  │  upperdir  ──►  tmpfs or /data partition  (read-write)     │  ◄── writes land here
  │  workdir   ──►  same filesystem as upper  (bookkeeping)    │
  │  lowerdir  ──►  squashfs mount            (read-only)      │  ◄── reads fall through
  └────────────────────────────────────────────────────────────┘
                              │
                    unified mount at  /
```

---

## 3. SquashFS — The Read-Only Root

SquashFS is a compressed, read-only filesystem built into the mainline Linux kernel since 2.6.29. Key properties:

| Property | Detail |
|---|---|
| Compression | LZ4 / ZSTD / XZ / GZIP — typically 40–60 % of original size |
| Write path | None — physically impossible to corrupt at runtime |
| Integrity | dm-verity Merkle tree for per-block cryptographic verification |
| Determinism | `mksquashfs` output is byte-for-byte reproducible (suitable for signing) |

### 3.1 Buildroot Configuration

```makefile
# In Buildroot .config:
BR2_TARGET_ROOTFS_SQUASHFS=y
BR2_TARGET_ROOTFS_SQUASHFS_ZSTD=y

# Output: output/images/rootfs.squashfs
```

Via `menuconfig`:  
`Filesystem images → SquashFS root filesystem image [*] → Compression: zstd`

### 3.2 Mounting at Boot (fstab)

```
/dev/mmcblk0p3   /mnt/rootfs   squashfs   ro,noatime,nodiratime   0 0
```

### 3.3 Host Inspection

```bash
# Mount on host for inspection
sudo mount -o loop,ro output/images/rootfs.squashfs /mnt/rootfs

# Verify compression ratio
du -sh output/images/rootfs.squashfs
du -sh /mnt/rootfs
```

---

## 4. OverlayFS — Writable Union Layer

OverlayFS (merged in Linux 3.18) implements a union mount: reads that miss the **upper** layer fall through to the **lower** read-only layer. Writes create copy-up inodes or whiteout files in the upper layer. The lower directory is never modified.

### 4.1 Copy-on-Write Behaviour (ASCII)

```
  Operation: process writes to /etc/hostname

  BEFORE write                         AFTER write
  ─────────────────────────────────────────────────────────────────
  upper/ (tmpfs)         [empty]        upper/etc/hostname  [rw]
  lower/etc/hostname     [ro, visible]  lower/etc/hostname  [ro, hidden by upper]

  ► Process reads /etc/hostname  →  upper copy returned
  ► System reboots               →  tmpfs upper vanishes, lower visible again
```

### 4.2 Mount Command

```bash
mount -t overlay overlay \
  -o lowerdir=/mnt/rootfs,upperdir=/run/overlay/upper,workdir=/run/overlay/work \
  /newroot

# Rules:
# - workdir MUST be on the same filesystem as upperdir
# - workdir MUST be empty before mounting
# - lowerdir may be a colon-separated list (stacked, right=bottom)
```

### 4.3 Stacked Lower Directories

```bash
# lowerdir layers (left = top, right = bottom):
#   /run/patches  ← small runtime patch layer (optional)
#   /mnt/rootfs   ← squashfs base

mount -t overlay overlay \
  -o lowerdir=/run/patches:/mnt/rootfs,upperdir=/run/upper,workdir=/run/work \
  /newroot
```

---

## 5. tmpfs Mounts for `/var` and `/tmp`

`tmpfs` lives entirely in RAM. It is ideal for ephemeral runtime data — log files, PID files, UNIX sockets, lock files — that must not survive a reboot and must never reach the flash.

### 5.1 fstab Entries

```
tmpfs   /tmp         tmpfs   defaults,nosuid,nodev,size=32m           0 0
tmpfs   /run         tmpfs   defaults,nosuid,nodev,mode=0755,size=16m 0 0
tmpfs   /var/log     tmpfs   defaults,nosuid,nodev,size=16m           0 0
tmpfs   /var/tmp     tmpfs   defaults,nosuid,nodev,size=8m            0 0
tmpfs   /var/cache   tmpfs   defaults,nosuid,nodev,size=8m            0 0
```

### 5.2 RAM Budget (ASCII)

```
  Total RAM: 256 MB (example)

  ┌──────────────────────────────────────────────────┐
  │  Kernel + drivers           ~32 MB               │
  │  Userspace processes        ~80 MB               │
  │  OverlayFS upper (tmpfs)    ~64 MB  ← ceiling    │
  │  /tmp                       ~32 MB  ← ceiling    │
  │  /var/log + /var/tmp        ~24 MB  ← ceiling    │
  │  Free headroom              ~24 MB               │
  └──────────────────────────────────────────────────┘

  NOTE: tmpfs only allocates pages actually written.
        size= is a ceiling, not a reservation.
        Over-committing causes OOM when the ceiling is hit.
```

---

## 6. Persistent Data Partition

Some data must survive reboots: user configuration, TLS certificates, calibration data, databases. This lives on a separate, writable partition — typically **ext4** or **f2fs** — mounted at `/data`.

### 6.1 Directory Layout (ASCII)

```
  /data/                        ← ext4, /dev/mmcblk0p5
  ├── .initialised              ← sentinel: first-boot done
  ├── config/
  │   └── app.conf              ← runtime-editable config
  ├── certs/
  │   ├── device.key            ← TLS private key (mode 0600)
  │   └── device.crt            ← TLS certificate
  ├── db/
  │   └── metrics.db            ← SQLite database
  └── firmware/
      └── pending.squashfs      ← staged OTA payload (A/B)
```

### 6.2 First-Boot Initialisation Script

```sh
#!/bin/sh
# /etc/init.d/S10data-init

DATA=/data
TEMPLATES=/etc/factory-defaults

case "$1" in
  start)
    if [ ! -f "$DATA/.initialised" ]; then
        echo "First boot: populating $DATA ..."
        cp -a "$TEMPLATES/." "$DATA/"
        touch "$DATA/.initialised"
        sync
        echo "Done."
    fi
    ;;
  stop) : ;;
esac
```

### 6.3 Formatting on First Flash (post-install script)

```bash
#!/bin/sh
# Run once from factory provisioning script
DATA_DEV=/dev/mmcblk0p5

mkfs.ext4 -L data -O ^has_journal "$DATA_DEV"
# Remove journal to save writes on flash.
# Use -O has_journal if data integrity under power loss is critical.
```

---

## 7. A/B Update Slot Design

A/B ("dual-copy") updates eliminate the risk of a failed OTA leaving the device unbootable. Two complete root filesystem images coexist. The bootloader tracks which slot is active and how many boot attempts remain.

### 7.1 Slot State Machine (ASCII)

```
  ┌────────────────────────────────────────────────────────────────┐
  │                     NORMAL OPERATION                           │
  │   Slot A: ACTIVE        Slot B: IDLE                           │
  └────────────────────────────────────────────────────────────────┘
                    │
                    │  OTA download begins
                    ▼
  ┌─────────────┐              ┌─────────────┐
  │   Slot A    │              │   Slot B    │
  │   ACTIVE    │──────────►   │  UPDATING   │
  └─────────────┘              └─────────────┘
                                      │
                               write + verify
                               set boot_slot=b
                               set slot_b_tries=3
                                      │
                                   reboot
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                  boot succeeds             boot fails (3x)
                         │                         │
                  health checks pass        tries → 0
                  commit: tries=0                  │
                         │                  bootloader rolls back
                         ▼                  boot_slot=a
                  ┌─────────────┐                  │
                  │   Slot B    │           ┌─────────────┐
                  │   ACTIVE    │           │   Slot A    │
                  └─────────────┘           │   ACTIVE    │
                                            └─────────────┘
```

### 7.2 U-Boot Environment Variables

```bash
# Stored in dedicated MTD partition (u-boot-env):
boot_slot=a
slot_a_tries=3
slot_b_tries=0

# U-Boot boot script:
if test ${boot_slot} = "a"; then
    if test ${slot_a_tries} -gt 0; then
        setenv slot_a_tries $((${slot_a_tries} - 1))
        saveenv
        run boot_a
    else
        setenv boot_slot b
        setenv slot_b_tries 3
        saveenv
        run boot_b
    fi
fi
```

### 7.3 RAUC Integration (Buildroot)

```makefile
BR2_PACKAGE_RAUC=y

# RAUC system.conf on device:
# [system]
# compatible=myboard-1.0
# bootloader=uboot
#
# [slot.rootfs.0]
# device=/dev/mmcblk0p3
# type=squashfs
# bootname=a
#
# [slot.rootfs.1]
# device=/dev/mmcblk0p4
# type=squashfs
# bootname=b
```

---

## 8. Buildroot Configuration

### 8.1 Annotated defconfig Fragment

```makefile
# ── Architecture ────────────────────────────────────────────────
BR2_arm=y
BR2_cortex_a53=y

# ── Toolchain ───────────────────────────────────────────────────
BR2_TOOLCHAIN_BUILDROOT_UCLIBC=y

# ── Read-only root ──────────────────────────────────────────────
BR2_TARGET_ROOTFS_SQUASHFS=y
BR2_TARGET_ROOTFS_SQUASHFS_ZSTD=y
# Do NOT enable ext2/ext4 root — squashfs IS the root image

# ── Kernel must include OverlayFS built-in (not module) ─────────
BR2_LINUX_KERNEL_CONFIG_USING_FRAGMENT=y
# kernel fragment:
#   CONFIG_OVERLAY_FS=y
#   CONFIG_SQUASHFS=y
#   CONFIG_SQUASHFS_ZSTD=y
#   CONFIG_DM_VERITY=y        # optional but recommended

# ── Init ────────────────────────────────────────────────────────
BR2_INIT_BUSYBOX=y            # or BR2_INIT_SYSTEMD=y

# ── Tools ───────────────────────────────────────────────────────
BR2_PACKAGE_E2FSPROGS=y       # mkfs.ext4 for data partition
BR2_PACKAGE_RAUC=y            # A/B OTA orchestration
BR2_PACKAGE_LIBUBOOTENV=y     # fw_printenv / fw_setenv
```

### 8.2 Post-Build Script

```bash
#!/bin/bash
# board/myboard/post-build.sh  — called by Buildroot after rootfs assembly

TARGET="$1"

# Remove write permission from /etc — it is a read-only template store
chmod -R a-w "$TARGET/etc"

# Strip package manager state (saves space, prevents accidental writes)
rm -rf "$TARGET/var/lib/opkg" "$TARGET/var/cache/opkg"

# Create mount-point stubs (they must exist in the squashfs)
install -d "$TARGET/data"
install -d "$TARGET/run"
install -d "$TARGET/var/log"
install -d "$TARGET/var/tmp"
install -d "$TARGET/var/cache"
install -d "$TARGET/mnt/rootfs"
install -d "$TARGET/run/overlay/upper"
install -d "$TARGET/run/overlay/work"
```

---

## 9. C Code Examples

### 9.1 Overlay Mount + `pivot_root` at Boot

A minimal, statically-linked C program that can serve as an early-init stage: mounts tmpfs, squashfs, overlayfs, then pivots into the new root.

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mount.h>
#include <sys/stat.h>
#include <sys/syscall.h>
#include <errno.h>

#define SQFS_DEV  "/dev/mmcblk0p3"
#define LOWER     "/mnt/lower"
#define UPPER     "/run/upper"
#define WORK      "/run/work"
#define NEWROOT   "/newroot"
#define OLDROOT   NEWROOT "/mnt/oldroot"

static int xmount(const char *src, const char *tgt,
                  const char *type, unsigned long flags,
                  const char *opts)
{
    if (mount(src, tgt, type, flags, opts) < 0) {
        fprintf(stderr, "mount %s -> %s: %s\n",
                src, tgt, strerror(errno));
        return -1;
    }
    return 0;
}

int main(void)
{
    /* 1. Minimal early mounts */
    xmount("proc",   "/proc", "proc",   0, NULL);
    xmount("sysfs",  "/sys",  "sysfs",  0, NULL);

    /* 2. tmpfs for overlay workspace */
    xmount("tmpfs", "/run", "tmpfs",
           MS_NODEV | MS_NOSUID, "size=64m,mode=0755");
    mkdir(UPPER,   0755);
    mkdir(WORK,    0700);
    mkdir(NEWROOT, 0755);

    /* 3. SquashFS read-only lower layer */
    mkdir(LOWER, 0755);
    if (xmount(SQFS_DEV, LOWER, "squashfs",
               MS_RDONLY | MS_NOATIME, NULL) < 0)
        return 1;

    /* 4. OverlayFS union */
    char opts[512];
    snprintf(opts, sizeof(opts),
             "lowerdir=%s,upperdir=%s,workdir=%s",
             LOWER, UPPER, WORK);
    if (xmount("overlay", NEWROOT, "overlay", 0, opts) < 0)
        return 1;

    /* 5. Move early mounts into new root */
    xmount("/proc", NEWROOT "/proc", NULL, MS_MOVE, NULL);
    xmount("/sys",  NEWROOT "/sys",  NULL, MS_MOVE, NULL);
    xmount("/run",  NEWROOT "/run",  NULL, MS_MOVE, NULL);

    /* 6. pivot_root: swap / and NEWROOT */
    mkdir(OLDROOT, 0700);
    if (syscall(SYS_pivot_root, NEWROOT, OLDROOT) < 0) {
        perror("pivot_root");
        return 1;
    }
    chdir("/");

    /* 7. Unmount old root (optional, keeps namespace clean) */
    umount2("/mnt/oldroot", MNT_DETACH);

    /* 8. Hand off to real init */
    execl("/sbin/init", "init", NULL);
    perror("exec /sbin/init");
    return 1;
}
```

### 9.2 Config Read with Template Fallback

Reads from `/data/config/app.conf`; if absent, falls back to the read-only template at `/etc/factory-defaults/app.conf`.

```c
#include <stdio.h>
#include <string.h>

#define DATA_CONF  "/data/config/app.conf"
#define TMPL_CONF  "/etc/factory-defaults/app.conf"

static FILE *open_config(void)
{
    FILE *f = fopen(DATA_CONF, "r");
    if (!f)
        f = fopen(TMPL_CONF, "r");   /* fall back to ro template */
    return f;
}

/* Returns 0 on success, -1 if key not found. */
int config_get(const char *key, char *out, size_t outlen)
{
    FILE *f = open_config();
    if (!f) return -1;

    char line[512];
    int  found = -1;

    while (fgets(line, sizeof(line), f)) {
        /* Skip comments and blank lines */
        if (line[0] == '#' || line[0] == '\n') continue;

        char *eq = strchr(line, '=');
        if (!eq) continue;
        *eq = '\0';

        if (strcmp(line, key) == 0) {
            char *val = eq + 1;
            size_t vlen = strlen(val);
            if (vlen && val[vlen - 1] == '\n') val[--vlen] = '\0';
            snprintf(out, outlen, "%s", val);
            found = 0;
            break;
        }
    }
    fclose(f);
    return found;
}

int main(void)
{
    char hostname[128] = "embedded-device";
    char loglevel[32]  = "info";

    config_get("hostname",  hostname,  sizeof(hostname));
    config_get("log_level", loglevel,  sizeof(loglevel));

    printf("hostname  = %s\n", hostname);
    printf("log_level = %s\n", loglevel);
    return 0;
}
```

### 9.3 A/B Slot Management via libubootenv

```c
#include <stdio.h>
#include <string.h>
#include <uboot_env.h>     /* libubootenv — BR2_PACKAGE_LIBUBOOTENV */

/* Returns 0 and writes slot name ("a" or "b") into *slot. */
int ab_get_active_slot(char *slot, size_t len)
{
    struct uboot_ctx *ctx = NULL;
    if (uboot_initialize(&ctx, NULL)) return -1;

    const char *val = uboot_get(ctx, "boot_slot");
    if (!val) { uboot_destroy(ctx); return -1; }

    snprintf(slot, len, "%s", val);
    uboot_destroy(ctx);
    return 0;
}

/* Commit the given slot as good (reset retry counter to 0). */
int ab_commit_slot(const char *slot)
{
    struct uboot_ctx *ctx = NULL;
    if (uboot_initialize(&ctx, NULL)) return -1;

    char key[32];
    snprintf(key, sizeof(key), "slot_%s_tries", slot);
    uboot_set(ctx, key, "0");

    int rc = uboot_close(ctx);   /* writes to flash */
    uboot_destroy(ctx);
    return rc;
}

/* Stage the inactive slot for next boot. */
int ab_stage_update(const char *new_slot, int retry_count)
{
    struct uboot_ctx *ctx = NULL;
    if (uboot_initialize(&ctx, NULL)) return -1;

    char key[32], val[8];
    snprintf(key, sizeof(key), "slot_%s_tries", new_slot);
    snprintf(val, sizeof(val), "%d", retry_count);

    uboot_set(ctx, key, val);
    uboot_set(ctx, "boot_slot", new_slot);

    int rc = uboot_close(ctx);
    uboot_destroy(ctx);
    return rc;
}

int main(void)
{
    char slot[4];
    if (ab_get_active_slot(slot, sizeof(slot)) == 0)
        printf("Active slot: %s\n", slot);

    /* After confirming system is healthy: */
    ab_commit_slot(slot);
    return 0;
}
```

---

## 10. C++ Code Examples

### 10.1 RAII Mount Wrapper

```cpp
#define _GNU_SOURCE
#include <stdexcept>
#include <string>
#include <system_error>
#include <cerrno>
#include <sys/mount.h>

class Mount {
public:
    Mount(std::string src, std::string target,
          std::string fstype,
          unsigned long flags = 0,
          std::string opts = "")
        : target_(std::move(target))
    {
        if (::mount(src.c_str(),
                    target_.c_str(),
                    fstype.empty() ? nullptr : fstype.c_str(),
                    flags,
                    opts.empty()   ? nullptr : opts.c_str()) < 0)
        {
            throw std::system_error(errno,
                std::generic_category(),
                "mount " + src + " -> " + target_);
        }
    }

    ~Mount() noexcept {
        if (!target_.empty())
            ::umount2(target_.c_str(), MNT_DETACH);
    }

    /* Non-copyable */
    Mount(const Mount&)            = delete;
    Mount& operator=(const Mount&) = delete;

    /* Moveable */
    Mount(Mount&& o) noexcept : target_(std::move(o.target_)) {
        o.target_.clear();
    }

private:
    std::string target_;
};

void setup_overlay()
{
    // If any mount throws, all previously constructed mounts are
    // automatically unmounted via their destructors (RAII).

    Mount sqfs("/dev/mmcblk0p3", "/mnt/lower",
               "squashfs", MS_RDONLY | MS_NOATIME);

    Mount ram("tmpfs", "/run", "tmpfs",
              MS_NODEV | MS_NOSUID, "size=64m,mode=0755");

    std::string opts =
        "lowerdir=/mnt/lower,upperdir=/run/upper,workdir=/run/work";
    Mount overlay("overlay", "/newroot", "overlay", 0, opts);

    // pivot_root, exec init ...
}
```

### 10.2 Atomic Config Write to `/data`

Power-loss-safe write: write to a `.tmp` sibling, `fsync`, then `rename`. The rename is atomic on POSIX — the file either has old or new content, never partial.

```cpp
#include <filesystem>
#include <fstream>
#include <stdexcept>
#include <string>
#include <fcntl.h>
#include <unistd.h>

namespace fs = std::filesystem;

void atomic_write(const fs::path& dest, const std::string& content)
{
    fs::path tmp = dest;
    tmp += ".tmp";

    /* Step 1: write + fsync the temp file */
    {
        std::ofstream out(tmp, std::ios::trunc | std::ios::binary);
        if (!out)
            throw std::runtime_error("Cannot open: " + tmp.string());

        out << content;
        out.flush();
        if (!out)
            throw std::runtime_error("Write failed: " + tmp.string());

        int fd = ::open(tmp.c_str(), O_WRONLY);
        if (fd >= 0) { ::fsync(fd); ::close(fd); }
    }

    /* Step 2: atomic rename */
    fs::rename(tmp, dest);

    /* Step 3: fsync parent directory (commits directory entry) */
    int dfd = ::open(dest.parent_path().c_str(),
                     O_RDONLY | O_DIRECTORY);
    if (dfd >= 0) { ::fsync(dfd); ::close(dfd); }
}

int main()
{
    atomic_write("/data/config/app.conf",
                 "hostname=prod-sensor-01\n"
                 "log_level=warn\n"
                 "sensor_interval_ms=500\n");
}
```

### 10.3 A/B OTA Update Orchestrator

```cpp
#include <filesystem>
#include <fstream>
#include <iostream>
#include <stdexcept>
#include <string>
#include <cstdlib>

namespace fs = std::filesystem;

struct SlotInfo {
    std::string device;     // block device path
    std::string env_slot;   // "a" or "b"
    std::string tries_key;  // U-Boot env key
};

class ABUpdater {
public:
    explicit ABUpdater(std::string active_slot)
        : active_(std::move(active_slot)) {}

    void apply(const fs::path& payload)
    {
        auto target = inactive();
        std::cout << "[OTA] Writing " << payload
                  << " -> " << target.device << '\n';

        /* Raw block write (production: use dd or libblkid) */
        std::ifstream src(payload, std::ios::binary);
        std::ofstream dst(target.device,
                          std::ios::binary | std::ios::trunc);
        if (!src || !dst)
            throw std::runtime_error("Cannot open source or target device");

        dst << src.rdbuf();
        dst.flush();
        if (!dst)
            throw std::runtime_error("Write failed");

        /* Set the inactive slot as next boot target */
        fw_setenv("boot_slot",       target.env_slot);
        fw_setenv(target.tries_key,  "3");

        std::cout << "[OTA] Staged. Reboot to activate slot '"
                  << target.env_slot << "'.\n";
    }

    void commit()
    {
        /* Called after successful boot health checks */
        std::string key = "slot_" + active_ + "_tries";
        fw_setenv(key, "0");
        std::cout << "[OTA] Slot '" << active_ << "' committed.\n";
    }

private:
    std::string active_;

    SlotInfo inactive() const {
        if (active_ == "a")
            return {"/dev/mmcblk0p4", "b", "slot_b_tries"};
        return     {"/dev/mmcblk0p3", "a", "slot_a_tries"};
    }

    void fw_setenv(const std::string& key, const std::string& val) {
        std::string cmd = "fw_setenv " + key + " " + val;
        if (std::system(cmd.c_str()) != 0)
            throw std::runtime_error("fw_setenv failed: " + cmd);
    }
};

int main()
{
    ABUpdater upd("a");
    upd.apply("/data/firmware/pending.squashfs");
    // ... reboot ...
    // After reboot and health checks:
    // upd.commit();
}
```

---

## 11. Rust Code Examples

### 11.1 Verify OverlayFS is Mounted at Root

Parses `/proc/mounts` and confirms the overlay union is active before the application starts.

```rust
use std::fs;
use std::io::{self, BufRead};

#[derive(Debug)]
struct MountEntry {
    device:      String,
    mount_point: String,
    fs_type:     String,
    options:     String,
}

fn parse_mounts() -> io::Result<Vec<MountEntry>> {
    let file = fs::File::open("/proc/mounts")?;
    let reader = io::BufReader::new(file);
    let mut entries = Vec::new();

    for line in reader.lines() {
        let line = line?;
        let parts: Vec<&str> = line.splitn(6, ' ').collect();
        if parts.len() < 4 { continue; }
        entries.push(MountEntry {
            device:      parts[0].to_owned(),
            mount_point: parts[1].to_owned(),
            fs_type:     parts[2].to_owned(),
            options:     parts[3].to_owned(),
        });
    }
    Ok(entries)
}

fn verify_overlay_root() -> bool {
    parse_mounts()
        .unwrap_or_default()
        .iter()
        .any(|e| e.fs_type == "overlay" && e.mount_point == "/")
}

fn verify_data_mounted() -> bool {
    parse_mounts()
        .unwrap_or_default()
        .iter()
        .any(|e| e.mount_point == "/data" && e.fs_type == "ext4")
}

fn main() {
    let overlay_ok = verify_overlay_root();
    let data_ok    = verify_data_mounted();

    println!("overlayfs at /  : {}", if overlay_ok { "OK" } else { "MISSING" });
    println!("/data (ext4)    : {}", if data_ok    { "OK" } else { "MISSING" });

    if !overlay_ok || !data_ok {
        eprintln!("ERROR: required mounts absent — aborting.");
        std::process::exit(1);
    }
}
```

### 11.2 Atomic Config Persistence with `serde_json`

```toml
# Cargo.toml
[dependencies]
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};
use std::fs;
use std::io::{self, Write};
use std::os::unix::fs::OpenOptionsExt;
use std::path::Path;

#[derive(Debug, Serialize, Deserialize, Default)]
struct AppConfig {
    pub hostname:            String,
    pub log_level:           String,
    pub sensor_interval_ms:  u32,
}

const CONFIG_PATH:   &str = "/data/config/app.json";
const TEMPLATE_PATH: &str = "/etc/factory-defaults/app.json";

fn load_config() -> AppConfig {
    let path = if Path::new(CONFIG_PATH).exists() {
        CONFIG_PATH
    } else {
        TEMPLATE_PATH   /* read-only fallback */
    };

    let data = fs::read_to_string(path).unwrap_or_default();
    serde_json::from_str(&data).unwrap_or_default()
}

fn save_config(cfg: &AppConfig) -> io::Result<()> {
    let tmp = format!("{CONFIG_PATH}.tmp");

    let json = serde_json::to_string_pretty(cfg)
        .map_err(|e| io::Error::new(io::ErrorKind::Other, e))?;

    /* Write + fsync tmp file */
    {
        let mut f = fs::OpenOptions::new()
            .write(true)
            .create(true)
            .truncate(true)
            .mode(0o600)
            .open(&tmp)?;
        f.write_all(json.as_bytes())?;
        f.sync_all()?;
    }

    /* Atomic rename */
    fs::rename(&tmp, CONFIG_PATH)?;

    /* Sync parent directory */
    let parent = Path::new(CONFIG_PATH).parent().unwrap();
    let dir = fs::File::open(parent)?;
    dir.sync_all()?;

    Ok(())
}

fn main() {
    let mut cfg = load_config();
    println!("Loaded: {:?}", cfg);

    cfg.hostname           = "prod-sensor-01".into();
    cfg.log_level          = "warn".into();
    cfg.sensor_interval_ms = 500;

    save_config(&cfg).expect("Failed to save config");
    println!("Config saved.");
}
```

### 11.3 Boot Watchdog / Slot Commit Daemon

After a successful OTA reboot into the new slot, this daemon performs health checks and commits the slot by resetting the bootloader retry counter. If it never runs (early crash, watchdog timeout), the bootloader decrements the counter until rollback is triggered automatically.

```rust
use std::process::Command;
use std::time::{Duration, Instant};
use std::thread;

const HEALTH_TIMEOUT:    Duration = Duration::from_secs(120);
const CHECK_INTERVAL:    Duration = Duration::from_secs(5);
const REQUIRED_SUCCESSES: u32     = 3;

fn current_slot() -> Option<String> {
    let out = Command::new("fw_printenv")
        .arg("boot_slot")
        .output()
        .ok()?;
    // Output: "boot_slot=a\n"
    String::from_utf8(out.stdout)
        .ok()?
        .split('=')
        .nth(1)
        .map(|v| v.trim().to_owned())
}

fn commit_slot(slot: &str) -> bool {
    let key = format!("slot_{slot}_tries");
    Command::new("fw_setenv")
        .args([key.as_str(), "0"])
        .status()
        .map(|s| s.success())
        .unwrap_or(false)
}

fn run_health_checks() -> bool {
    // Extend with real checks:
    //   - ping default gateway
    //   - ensure critical systemd units are active
    //   - verify sensor data stream is live
    true
}

fn main() {
    let start    = Instant::now();
    let mut ok_count = 0u32;

    println!("Boot watchdog started (timeout {}s).",
             HEALTH_TIMEOUT.as_secs());

    loop {
        if start.elapsed() >= HEALTH_TIMEOUT {
            eprintln!("Health timeout! Rollback will occur on next boot.");
            std::process::exit(1);
        }

        if run_health_checks() {
            ok_count += 1;
            println!("Health check {}/{REQUIRED_SUCCESSES} passed.",
                     ok_count);

            if ok_count >= REQUIRED_SUCCESSES {
                if let Some(slot) = current_slot() {
                    if commit_slot(&slot) {
                        println!("Slot '{slot}' committed successfully.");
                        return;
                    } else {
                        eprintln!("fw_setenv failed — will retry.");
                        ok_count = 0;
                    }
                }
            }
        } else {
            eprintln!("Health check failed — resetting counter.");
            ok_count = 0;
        }

        thread::sleep(CHECK_INTERVAL);
    }
}
```

---

## 12. Init System Integration

### 12.1 BusyBox inittab + rcS

```
# /etc/inittab
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty -L ttyS0 115200 vt100
::shutdown:/etc/init.d/rcK
```

```sh
#!/bin/sh
# /etc/init.d/S05overlay — sets up overlay before any other service

case "$1" in
  start)
    echo "Setting up overlayfs ..."
    mount -t tmpfs tmpfs /run \
        -o size=64m,mode=0755,nosuid,nodev

    mkdir -p /run/overlay/upper /run/overlay/work

    mount -t overlay overlay / \
        -o lowerdir=/mnt/rootfs,\
upperdir=/run/overlay/upper,\
workdir=/run/overlay/work
    ;;
  stop) : ;;
esac
```

```sh
#!/bin/sh
# /etc/init.d/S10mounts — tmpfs ephemeral dirs + /data

case "$1" in
  start)
    mount -t tmpfs tmpfs /tmp     -o size=32m,nosuid,nodev
    mount -t tmpfs tmpfs /var/log -o size=16m,nosuid,nodev
    mount -t tmpfs tmpfs /var/tmp -o size=8m,nosuid,nodev
    mount /dev/mmcblk0p5 /data    -t ext4 -o rw,noatime
    ;;
  stop)
    umount /data
    ;;
esac
```

### 12.2 systemd Mount Units

```ini
# /etc/systemd/system/data.mount
[Unit]
Description=Persistent Data Partition
DefaultDependencies=no
Before=local-fs.target

[Mount]
What=/dev/mmcblk0p5
Where=/data
Type=ext4
Options=rw,noatime,data=ordered

[Install]
WantedBy=local-fs.target
```

```ini
# /etc/systemd/system/var-log.mount
[Unit]
Description=Volatile /var/log (tmpfs)
DefaultDependencies=no

[Mount]
What=tmpfs
Where=/var/log
Type=tmpfs
Options=size=16m,nosuid,nodev,mode=0755

[Install]
WantedBy=local-fs.target
```

```ini
# /etc/systemd/system/data-init.service
[Unit]
Description=First-Boot Data Partition Initialisation
After=data.mount
Requires=data.mount
ConditionPathExists=!/data/.initialised

[Service]
Type=oneshot
ExecStart=/usr/sbin/data-init.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

## 13. Security Considerations

### 13.1 dm-verity — Block-Level Integrity

```
  SquashFS image with dm-verity

  ┌──────────────────────────────────────────────────┐
  │  Data blocks  (4 KB each)                        │
  │  ┌──────┬──────┬──────┬──────┬──────┬──────┐     │
  │  │ blk0 │ blk1 │ blk2 │ blk3 │ blk4 │ ...  │     │
  │  └──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──────┘     │
  │     │      │      │      │      │                │
  │  Hash tree (Merkle)                              │
  │  ┌──────────────────────┐                        │
  │  │  H(blk0) H(blk1)...  │  ← leaf hashes         │
  │  │    H(H0+H1) ...      │  ← intermediate        │
  │  │       Root Hash      │  ← signed in kernel    │
  │  └──────────────────────┘                        │
  └──────────────────────────────────────────────────┘

  Any bit-flip in any block  →  hash mismatch  →  I/O error
  Root hash embedded in signed kernel or stored in secure OTP.
```

Enable in kernel: `CONFIG_DM_VERITY=y`  
Kernel cmdline: `root=/dev/dm-0 dm-mod.create="vroot,,,ro, 0 131072 verity 1 /dev/mmcblk0p3 /dev/mmcblk0p3 4096 4096 16384 16384 sha256 <roothash> <salt>"`

### 13.2 Threat Model Summary

| Threat | Mitigation |
|---|---|
| Physical flash attacker | squashfs + dm-verity: silent modification detected |
| Remote RCE writing files | `/` is ro; writes land in tmpfs, lost on reboot |
| Persistent backdoor via `/data` | Separate mount, can be wiped independently |
| Failed OTA bricks device | A/B slots + retry countdown → automatic rollback |
| Data confidentiality | dm-crypt on `/data` with device-unique key from TPM/eFuse |
| Supply-chain image tampering | Sign squashfs image in CI, verify root hash at boot |

---

## 14. Summary

### Component Roles at a Glance

```
  Component        Role                                   Flash writes
  ──────────────────────────────────────────────────────────────────────
  squashfs         Compressed, verified read-only root        NONE
  overlayfs        Union layer — / appears writable           NONE *
  tmpfs            RAM-backed ephemeral storage               NONE
  /data  (ext4)    Persistent user & application data         YES
  A-slot rootfs    Current active OS image                    YES (OTA)
  B-slot rootfs    Staged candidate / rollback target         YES (OTA)
  U-Boot env       Boot slot state & retry counters           YES (small)

  * overlayfs upper on tmpfs: writes are in RAM, lost on reboot.
    overlayfs upper on /data: writes persist but are intentional.
```

### Design Principles

The read-only root strategy rests on three pillars:

**Reliability** — The root filesystem can never be corrupted by a power loss or a rogue process. SquashFS has no write path in the kernel driver. Even if a process calls `open("/bin/ls", O_WRONLY)`, the kernel returns `EROFS` before any flash I/O occurs.

**Security** — dm-verity verifies each 4 KB block of squashfs against a cryptographic Merkle tree whose root hash is embedded in the signed kernel image. Any modification — whether from flash wear, cosmic ray, or an attacker with physical access — is detected before the data is used, not after.

**Maintainability** — A/B update slots make OTA updates atomic and automatically rolled back on failure. The bootloader's retry countdown means a device that crashes on every boot in the new slot will silently revert to the last known good slot within minutes, with no manual intervention.

### Buildroot Integration in a Nutshell

```
  .config        →  BR2_TARGET_ROOTFS_SQUASHFS=y + compression choice
  kernel config  →  CONFIG_OVERLAY_FS=y, CONFIG_SQUASHFS=y, CONFIG_DM_VERITY=y
  post-build.sh  →  chmod -R a-w etc/, install -d /data /run /var/log
  init script    →  mount tmpfs, squashfs, overlayfs; then pivot_root
  /data init     →  copy /etc/factory-defaults → /data on first boot
  RAUC / swupd   →  orchestrate A/B slot writes and bootloader env updates
```

The C, C++, and Rust examples in this chapter provide battle-tested primitives that can be dropped directly into any Buildroot-based embedded project: boot-time overlay orchestration, power-loss-safe atomic writes to the data partition, A/B slot management through the U-Boot environment, and a boot watchdog that guarantees automatic rollback without any cloud dependency.

---

*End of Chapter 18 — Read-Only Root & Persistent Data Strategies*