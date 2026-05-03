# 30. OTA Update Strategies (SWUpdate, RAUC, Mender)

**Architecture & Concepts** — atomic update principles, A/B partition layout (ASCII block diagram), bootloader decision flow, and naming conventions across all three frameworks.

**SWUpdate** — `.swu` CPIO bundle structure, `sw-description` in libconfig and Lua, shell script for signing with OpenSSL CMS, and Lua-based A/B slot detection.

**RAUC** — `system.conf` slot mapping, `manifest.raucm`, bundle creation script, HSM signing pipeline diagram, and a full D-Bus client in C.

**Mender** — artifact format internals, `mender-artifact` CLI usage, and `mender.conf` configuration.

**Buildroot Integration** — `menuconfig` paths, defconfig snippet, post-image shell script, and `genimage` A/B disk layout config.

**Code Examples:**
- **C** — `libubootenv` A/B slot switching, watchdog-guarded update thread, SHA-256 verified block device writer
- **C++** — Full A/B OTA state machine with entry actions, `sdbus-c++` RAUC progress monitor
- **Rust** — Async HTTPS downloader with SHA-256 + progress bar, safe A/B slot manager with health checks, SWUpdate IPC socket monitor

**Security** — hardware root-of-trust chain diagram, PKI key hierarchy, and a hardening checklist.


## A/B Partition Switching, SWUpdate `.swu` Bundles, RAUC Bundle Signing, and Buildroot Integration Packages

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Core Concepts: Reliable OTA Updates](#2-core-concepts-reliable-ota-updates)
3. [A/B Partition Switching Architecture](#3-ab-partition-switching-architecture)
4. [SWUpdate — Software Update Framework](#4-swupdate--software-update-framework)
5. [RAUC — Robust Auto-Update Controller](#5-rauc--robust-auto-update-controller)
6. [Mender — Full-Stack OTA Platform](#6-mender--full-stack-ota-platform)
7. [Buildroot Integration](#7-buildroot-integration)
8. [C Programming Examples](#8-c-programming-examples)
9. [C++ Programming Examples](#9-c-programming-examples-1)
10. [Rust Programming Examples](#10-rust-programming-examples)
11. [Security Considerations](#11-security-considerations)
12. [Comparison and Selection Guide](#12-comparison-and-selection-guide)
13. [Summary](#13-summary)

---

## 1. Introduction

Over-the-Air (OTA) update mechanisms are critical components of any production embedded Linux system. The ability to remotely update firmware, applications, and system software is essential for:

- **Security patching** — fixing vulnerabilities discovered post-deployment
- **Feature delivery** — adding capabilities to already-shipped devices
- **Bug remediation** — correcting defects in the field
- **Compliance** — maintaining regulatory certifications over time

Buildroot provides first-class integration packages for three major OTA frameworks: **SWUpdate**, **RAUC**, and **Mender**. Each has distinct philosophies, strengths, and trade-offs, but all three build upon the same foundational concept: safe, atomic, rollback-capable system updates via **A/B partition switching**.

---

## 2. Core Concepts: Reliable OTA Updates

### 2.1 The Problem with In-Place Updates

Traditional in-place updates (overwriting a running system) are inherently risky:

```
DANGER: In-place update failure scenario
─────────────────────────────────────────
  [Flash]     Power loss or error
  Writing...  ───────────────────>  BRICK!
  [Corrupt]
```

A power failure, network interruption, or write error mid-update leaves the device in an inconsistent, potentially unbootable state with no recovery path.

### 2.2 Atomic Update Principles

Reliable OTA frameworks enforce **atomicity** — an update either fully succeeds or the system rolls back cleanly:

```
ATOMIC UPDATE PRINCIPLE
═══════════════════════
  Before:  [Slot A: v1.0 ACTIVE]  [Slot B: empty]
  During:  [Slot A: v1.0 ACTIVE]  [Slot B: writing v2.0...]
  After:   [Slot A: v1.0 standby] [Slot B: v2.0 ACTIVE]
  Fail:    [Slot A: v1.0 ACTIVE]  [Slot B: dirty, ignored]
```

---

## 3. A/B Partition Switching Architecture

### 3.1 Physical Layout

The A/B (or "dual-copy") scheme maintains two complete, bootable system images on separate partitions:

```
DISK / FLASH LAYOUT — A/B Partitioned System
╔═════════════════════════════════════════════════════════╗
║  BLOCK DEVICE  /dev/mmcblk0  (or /dev/sda, /dev/mtd*)   ║
╠════════╦═══════════╦═══════════╦═════════╦══════════════╣
║ MBR /  ║  Slot A   ║  Slot B   ║ /data   ║  Bootloader  ║
║ GPT    ║ rootfs    ║ rootfs    ║ (persis)║  env / vars  ║
║ 512B   ║ p2 ~500MB ║ p3 ~500MB ║ p4      ║  p1          ║
╚════════╩═══════════╩═══════════╩═════════╩══════════════╝
            ^
            Active slot (current boot)
```

### 3.2 Bootloader Role (U-Boot / GRUB)

The bootloader is the arbiter of which slot boots. It reads metadata variables to determine:

- Which slot is currently active
- How many boot attempts remain (retry counter)
- Whether the active slot has been confirmed as "good"

```
U-BOOT A/B BOOT DECISION FLOW
══════════════════════════════
  Power On
      |
      v
  [Read uboot env]
      |
      +---> boot_slot = "A" or "B"
      |
      +---> upgrade_available = "0" or "1"
      |
      v
  upgrade_available == 1 ?
      |
     YES ──────────────────────────────> Boot NEW slot
      |                                      |
     NO                               bootcount++
      |                                      |
      v                               bootcount > LIMIT ?
  Boot ACTIVE slot                           |
  (confirmed good)                          YES ──> Fallback to OLD slot
                                             |
                                            NO ──> Continue boot
                                                       |
                                            Application confirms good
                                                       |
                                            Set upgrade_available=0
                                            Reset bootcount=0
```

### 3.3 Partition Naming Conventions

```
GPT PARTITION LABEL CONVENTIONS
────────────────────────────────
  Framework    Slot A Label     Slot B Label
  ─────────    ────────────     ────────────
  SWUpdate     rootfs           rootfs_1
  RAUC         rootfs.0         rootfs.1
  Mender       primary          secondary
  Generic      system_a         system_b
```

---

## 4. SWUpdate — Software Update Framework

### 4.1 Overview

SWUpdate is a Linux-native, highly configurable update agent developed by Stefano Babic at Denx. It accepts `.swu` (Software Update) bundles — CPIO archives containing images and a descriptor file (`sw-description`).

```
SWUPDATE ARCHITECTURE
═════════════════════════════════════════════════════════
  ┌─────────────────────────────────────────────────────┐
  │                  SWUpdate Process                   │
  │                                                     │
  │  ┌──────────┐   ┌──────────┐   ┌────────────────┐   │
  │  │  Input   │──>│  Parser  │──>│   Installer    │   │
  │  │ Handler  │   │  (Lua /  │   │  (bootloader   │   │
  │  │          │   │ libconfig│   │  + flash ops)  │   │
  │  │ HTTP/    │   └──────────┘   └────────────────┘   │
  │  │ USB/TFTP │                                       │
  │  └──────────┘   ┌──────────┐   ┌────────────────┐   │
  │                 │ Notifier │   │   Suricatta    │   │
  │                 │ (IPC/Web │   │ (hawkBit/OTA   │   │
  │                 │  socket) │   │  daemon mode)  │   │
  │                 └──────────┘   └────────────────┘   │
  └─────────────────────────────────────────────────────┘
```

### 4.2 The `.swu` Bundle Format

A `.swu` file is a CPIO archive:

```
.swu BUNDLE INTERNAL STRUCTURE
───────────────────────────────
  update_1.0.swu  (CPIO archive)
  ├── sw-description         <- Lua/libconfig descriptor (SIGNED)
  ├── sw-description.sig     <- Detached CMS signature
  ├── rootfs.ext4.gz         <- Compressed rootfs image
  ├── kernel.itb             <- FIT image (kernel+dtb)
  └── bootloader.env         <- U-Boot environment patch
```

### 4.3 `sw-description` Format (libconfig syntax)

```
software =
{
    version = "2.0.0";
    description = "Production firmware release 2.0.0";

    hardware-compatibility: ["1.0", "1.1", "1.2"];

    images: (
        {
            filename = "rootfs.ext4.gz";
            type = "raw";
            device = "/dev/mmcblk0p3";   /* Slot B */
            compressed = "zlib";
            sha256 = "b94d27b9934d3e08a52e52d7da7dabfac...";
        },
        {
            filename = "kernel.itb";
            type = "raw";
            device = "/dev/mmcblk0p1";
            sha256 = "a87ff679a2f3e71d9181a67b7542122c...";
        }
    );

    scripts: (
        {
            filename = "post-install.lua";
            type = "lua";
        }
    );

    uboot: (
        {
            name = "upgrade_available";
            value = "1";
        },
        {
            name = "boot_slot";
            value = "B";
        }
    );
}
```

### 4.4 Creating and Signing a `.swu` Bundle

```bash
#!/bin/bash
# build_swu.sh — Build and sign a SWUpdate bundle

# Generate signing keypair (RSA 4096)
openssl genrsa -out sw-description.key 4096
openssl req -new -x509 -key sw-description.key \
    -out sw-description.cert -days 3650

# Sign the sw-description
openssl cms -sign \
    -in sw-description \
    -out sw-description.sig \
    -signer sw-description.cert \
    -inkey sw-description.key \
    -outform DER \
    -nodetach

# Create the CPIO archive (sw-description MUST be first)
echo sw-description       > /tmp/swu_files
echo sw-description.sig  >> /tmp/swu_files
echo rootfs.ext4.gz      >> /tmp/swu_files
echo kernel.itb          >> /tmp/swu_files

cat /tmp/swu_files | cpio -ov -H newc > update_2.0.0.swu

echo "Bundle size: $(du -sh update_2.0.0.swu)"
```

### 4.5 SWUpdate `sw-description` with A/B Logic (Lua)

```lua
-- sw-description using Lua for A/B slot detection
function get_inactive_slot()
    local f = io.open("/sys/firmware/devicetree/base/chosen/active-slot", "r")
    if f then
        local slot = f:read("*l")
        f:close()
        if slot == "A" then return "B" end
    end
    return "A"   -- default to A if unreadable
end

local inactive = get_inactive_slot()
local rootfs_dev = (inactive == "A") and "/dev/mmcblk0p2" or "/dev/mmcblk0p3"

software = {
    version = "2.0.0",
    images = {
        {
            filename  = "rootfs.ext4.gz",
            device    = rootfs_dev,
            type      = "raw",
            compressed = "zlib",
        }
    }
}
```

---

## 5. RAUC — Robust Auto-Update Controller

### 5.1 Overview

RAUC (Robust Auto-Update Controller) takes a more opinionated, D-Bus-oriented approach. It enforces strong security via cryptographic bundle signing and integrates deeply with systemd and bootloader backends (U-Boot, GRUB, EFI, Barebox).

```
RAUC SYSTEM ARCHITECTURE
═════════════════════════════════════════════════════════
  ┌─────────────────────────────────────────────────────┐
  │                  Target Device                      │
  │                                                     │
  │  ┌─────────┐   D-Bus    ┌──────────────────────┐    │
  │  │ rauc    │<──────────>│   rauc service       │    │
  │  │ CLI tool│            │   (rauc.service)     │    │
  │  └─────────┘            │                      │    │
  │                         │  ┌────────────────┐  │    │
  │  ┌─────────┐            │  │ Bundle Verify  │  │    │
  │  │ Custom  │            │  │ (CMS signature)│  │    │
  │  │ App     │            │  └────────────────┘  │    │
  │  └─────────┘            │  ┌────────────────┐  │    │
  │      |                  │  │ Slot Handler   │  │    │
  │      | GLib/D-Bus       │  │ (raw/ext4/tar) │  │    │
  │      v                  │  └────────────────┘  │    │
  │  [org.rauc.installer]   │  ┌────────────────┐  │    │
  │                         │  │ Boot Backend   │  │    │
  │                         │  │ (u-boot/grub)  │  │    │
  │                         │  └────────────────┘  │    │
  │                         └──────────────────────┘    │
  └─────────────────────────────────────────────────────┘
```

### 5.2 RAUC System Configuration (`system.conf`)

```ini
[system]
compatible=MyDevice-v1
bootloader=uboot

[keyring]
path=/etc/rauc/keyring.pem
use-bundle-signing-time=true

[slot.bootloader.0]
device=/dev/mmcblk0p1
type=raw
bootname=bootloader

[slot.rootfs.0]
device=/dev/mmcblk0p2
type=ext4
bootname=A
parent=bootloader.0

[slot.rootfs.1]
device=/dev/mmcblk0p3
type=ext4
bootname=B
parent=bootloader.0

[slot.appfs.0]
device=/dev/mmcblk0p4
type=ext4
parent=rootfs.0

[slot.appfs.1]
device=/dev/mmcblk0p5
type=ext4
parent=rootfs.1
```

### 5.3 RAUC Bundle `manifest.raucm`

```ini
[update]
compatible=MyDevice-v1
version=2.0.0
description=Production Release 2.0.0 - Security patch CVE-2024-XXXX
build=20240915120000

[bundle]
format=verity

[image.rootfs]
filename=rootfs.ext4
sha256=3d90d3a71fe...
size=524288000

[image.appfs]
filename=appfs.ext4
sha256=8f14e45fce...
size=134217728
```

### 5.4 Building and Signing a RAUC Bundle

```bash
#!/bin/bash
# build_rauc_bundle.sh

BUNDLE_DIR="bundle_staging"
CERT="rauc-developer.cert.pem"
KEY="rauc-developer.key.pem"

# Create certificate (development — use HSM in production)
openssl req -x509 -newkey rsa:4096 \
    -keyout "$KEY" \
    -out "$CERT" \
    -days 365 \
    -nodes \
    -subj "/CN=RAUC Developer Certificate"

mkdir -p "$BUNDLE_DIR"
cp manifest.raucm    "$BUNDLE_DIR/"
cp rootfs.ext4       "$BUNDLE_DIR/"
cp appfs.ext4        "$BUNDLE_DIR/"

# Build signed bundle
rauc bundle \
    --cert="$CERT" \
    --key="$KEY" \
    "$BUNDLE_DIR" \
    release_2.0.0.raucb

# Inspect result
rauc info \
    --keyring="$CERT" \
    release_2.0.0.raucb
```

### 5.5 RAUC Bundle Signing with HSM (Production)

```
RAUC SIGNING PIPELINE — PRODUCTION
═════════════════════════════════════════════════════════
  Build Server                    HSM (e.g. SoftHSM/YubiHSM)
  ─────────────                   ────────────────────────
  rootfs.ext4                          ┌──────────┐
  appfs.ext4       ─── PKCS#11 ──────> │ Sign CMS │
  manifest.raucm                       │ bundle   │
       |                               └──────────┘
       |  rauc bundle                       |
       |  --signing-keyring=pkcs11:...      |
       v                                    v
  release.raucb  <──────────── Signed bundle returned
       |
       v
  [Upload to OTA Server]
```

### 5.6 RAUC D-Bus API Integration

```c
/* rauc_dbus_client.c — Trigger RAUC install via D-Bus */
#include <gio/gio.h>
#include <stdio.h>

#define RAUC_BUS_NAME     "de.pengutronix.rauc"
#define RAUC_OBJECT_PATH  "/"
#define RAUC_INTERFACE    "de.pengutronix.rauc.Installer"

static void on_install_completed(GDBusConnection *conn,
                                  const gchar *sender,
                                  const gchar *path,
                                  const gchar *iface,
                                  const gchar *signal,
                                  GVariant *params,
                                  gpointer user_data)
{
    gint32 result;
    g_variant_get(params, "(i)", &result);

    if (result == 0)
        g_print("RAUC install succeeded.\n");
    else
        g_print("RAUC install FAILED with code %d\n", result);

    g_main_loop_quit((GMainLoop *)user_data);
}

int rauc_install_bundle(const char *bundle_path)
{
    GError          *error   = NULL;
    GDBusConnection *conn    = NULL;
    GMainLoop       *loop    = NULL;
    GDBusProxy      *proxy   = NULL;
    GVariant        *result  = NULL;
    int              ret     = -1;

    conn = g_bus_get_sync(G_BUS_TYPE_SYSTEM, NULL, &error);
    if (!conn) {
        g_printerr("D-Bus connection failed: %s\n", error->message);
        goto out;
    }

    proxy = g_dbus_proxy_new_sync(conn,
                                   G_DBUS_PROXY_FLAGS_NONE,
                                   NULL,
                                   RAUC_BUS_NAME,
                                   RAUC_OBJECT_PATH,
                                   RAUC_INTERFACE,
                                   NULL, &error);
    if (!proxy) {
        g_printerr("Proxy creation failed: %s\n", error->message);
        goto out;
    }

    loop = g_main_loop_new(NULL, FALSE);

    /* Subscribe to Completed signal */
    g_dbus_connection_signal_subscribe(conn,
                                        RAUC_BUS_NAME,
                                        RAUC_INTERFACE,
                                        "Completed",
                                        RAUC_OBJECT_PATH,
                                        NULL,
                                        G_DBUS_SIGNAL_FLAGS_NONE,
                                        on_install_completed,
                                        loop, NULL);

    /* Invoke InstallBundle method */
    result = g_dbus_proxy_call_sync(proxy,
                                     "InstallBundle",
                                     g_variant_new("(sa{sv})",
                                                    bundle_path,
                                                    NULL),
                                     G_DBUS_CALL_FLAGS_NONE,
                                     -1, NULL, &error);
    if (!result) {
        g_printerr("InstallBundle call failed: %s\n", error->message);
        goto out;
    }

    g_main_loop_run(loop);
    ret = 0;

out:
    if (error)  g_error_free(error);
    if (result) g_variant_unref(result);
    if (proxy)  g_object_unref(proxy);
    if (conn)   g_object_unref(conn);
    if (loop)   g_main_loop_unref(loop);

    return ret;
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <bundle.raucb>\n", argv[0]);
        return 1;
    }
    return rauc_install_bundle(argv[1]);
}
```

---

## 6. Mender — Full-Stack OTA Platform

### 6.1 Overview

Mender provides the most complete OTA solution, including a cloud-hosted or self-hosted server, device authentication, deployment management, and a rich artifact system beyond rootfs images.

```
MENDER FULL-STACK ARCHITECTURE
═════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────┐
  │               Mender Server (Cloud)              │
  │                                                  │
  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
  │  │ Device   │  │Deployment│  │  Artifact    │    │
  │  │ Auth     │  │ Manager  │  │  Storage     │    │
  │  └──────────┘  └──────────┘  └──────────────┘    │
  └──────────────────────┬───────────────────────────┘
                         │ HTTPS + JWT
              ┌──────────┴──────────┐
              │                     │
  ┌───────────▼────────┐ ┌──────────▼─────────┐
  │    Device A        │ │    Device B        │
  │  ┌─────────────┐   │ │  ┌─────────────┐   │
  │  │mender-client│   │ │  │mender-client│   │
  │  │ (daemon)    │   │ │  │ (daemon)    │   │
  │  └─────────────┘   │ │  └─────────────┘   │
  │  [A | B | data]    │ │  [A | B | data]    │
  └────────────────────┘ └────────────────────┘
```

### 6.2 Mender Artifact Format

```
MENDER ARTIFACT INTERNAL STRUCTURE
───────────────────────────────────
  update.mender  (TAR archive)
  ├── version                      <- Artifact format version (3)
  ├── manifest                     <- File checksums
  ├── manifest.sig                 <- RSA/ECDSA signature
  ├── header.tar.gz
  │   ├── header-info              <- Device type + artifact name
  │   ├── type-info                <- Payload type (rootfs-image)
  │   └── meta-data                <- Custom key-value metadata
  └── data/
      └── 0000.tar.gz
          └── rootfs.img           <- The actual payload
```

### 6.3 Building a Mender Artifact

```bash
#!/bin/bash
# build_mender_artifact.sh

ARTIFACT_NAME="release-2.0.0"
DEVICE_TYPE="my-embedded-device"
ROOTFS_IMAGE="rootfs.ext4"
OUTPUT="release-2.0.0.mender"

# Using mender-artifact CLI tool
mender-artifact write rootfs-image \
    --artifact-name  "$ARTIFACT_NAME" \
    --device-type    "$DEVICE_TYPE" \
    --file           "$ROOTFS_IMAGE" \
    --output         "$OUTPUT" \
    --sign           /etc/mender/private.key

# Verify the artifact
mender-artifact read "$OUTPUT"
mender-artifact validate \
    --key /etc/mender/public.key \
    "$OUTPUT"
```

### 6.4 Mender Client Configuration (`mender.conf`)

```json
{
    "ServerURL": "https://hosted.mender.io",
    "TenantToken": "eyJhbGciOiJSUzI1NiIs...",
    "UpdatePollIntervalSeconds": 1800,
    "InventoryPollIntervalSeconds": 28800,
    "RetryPollIntervalSeconds": 300,
    "RootfsPartA": "/dev/mmcblk0p2",
    "RootfsPartB": "/dev/mmcblk0p3",
    "BootUtilitiesGetNextActivePart": "fw_printenv",
    "BootUtilitiesSetActivePart": "fw_setenv"
}
```

---

## 7. Buildroot Integration

### 7.1 Enabling OTA Packages in Buildroot

```
BUILDROOT MENUCONFIG NAVIGATION
════════════════════════════════
  make menuconfig
      │
      └─> Target packages
              │
              └─> System tools
                      │
                      ├─> [*] rauc          (BR2_PACKAGE_RAUC)
                      ├─> [*]   rauc-hawkbit-updater
                      │
                      ├─> [*] swupdate      (BR2_PACKAGE_SWUPDATE)
                      ├─> [*]   swupdate webserver
                      ├─> [*]   suricatta (hawkBit backend)
                      │
                      └─> [*] mender-client (BR2_PACKAGE_MENDER)
```

### 7.2 Buildroot Defconfig for RAUC

```makefile
# configs/my_device_rauc_defconfig

BR2_arm=y
BR2_cortex_a53=y
BR2_TOOLCHAIN_BUILDROOT_GLIBC=y

# Filesystem layout — two rootfs partitions
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
BR2_TARGET_ROOTFS_EXT2_SIZE="512M"

# RAUC
BR2_PACKAGE_RAUC=y
BR2_PACKAGE_RAUC_DBUS=y

# Boot support
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="my_board"

# Systemd (RAUC recommends systemd)
BR2_INIT_SYSTEMD=y
BR2_PACKAGE_SYSTEMD=y
```

### 7.3 Custom RAUC Post-Image Script

```bash
# board/my_device/post-image.sh
# Called by Buildroot after images are built

#!/bin/bash
set -e

BINARIES_DIR="${BINARIES_DIR}"
BOARD_DIR="$(dirname $0)"

# Install RAUC system config into rootfs overlay
mkdir -p "${TARGET_DIR}/etc/rauc"
cp "${BOARD_DIR}/rauc/system.conf"  "${TARGET_DIR}/etc/rauc/"
cp "${BOARD_DIR}/rauc/keyring.pem"  "${TARGET_DIR}/etc/rauc/"

# Create final disk image with A/B layout
genimage \
    --config "${BOARD_DIR}/genimage-ab.cfg" \
    --rootpath "${TARGET_DIR}" \
    --tmppath "${BUILD_DIR}/genimage.tmp" \
    --inputpath "${BINARIES_DIR}" \
    --outputpath "${BINARIES_DIR}"

echo "[post-image] A/B disk image written to ${BINARIES_DIR}/sdcard.img"
```

### 7.4 `genimage` Configuration for A/B Layout

```
# board/my_device/genimage-ab.cfg
image rootfs_a.ext4 {
    ext4 {}
    size = 512M
    mountpoint = "/"
}

image rootfs_b.ext4 {
    ext4 {}
    size = 512M
}

image sdcard.img {
    hdimage {}

    partition bootloader {
        partition-type = 0x0C
        bootable = true
        image = "u-boot.img"
        size = 4M
    }

    partition rootfs_a {
        partition-type = 0x83
        image = "rootfs_a.ext4"
        size = 512M
    }

    partition rootfs_b {
        partition-type = 0x83
        image = "rootfs_b.ext4"
        size = 512M
    }

    partition data {
        partition-type = 0x83
        image = "data.ext4"
        size = 128M
    }
}
```

---

## 8. C Programming Examples

### 8.1 U-Boot Environment Manipulation for A/B Switching

```c
/* ab_uboot_env.c
 * Read/write U-Boot environment to control A/B slot selection.
 * Links against libubootenv: -lubootenv
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <uboot/uboot_env.h>   /* from libubootenv */

#define UBOOT_ENV_DEV   "/dev/mmcblk0"
#define UBOOT_ENV_OFFSET 0x800000   /* 8 MB offset */
#define UBOOT_ENV_SIZE   0x20000    /* 128 KB */

typedef enum {
    SLOT_A = 0,
    SLOT_B = 1
} ab_slot_t;

/* ─── Open U-Boot environment context ─────────────────────────── */
static struct uboot_ctx *open_uboot_env(void)
{
    struct uboot_ctx *ctx = NULL;
    int rc;

    rc = libuboot_initialize(&ctx, NULL);
    if (rc < 0) {
        fprintf(stderr, "libuboot_initialize failed: %d\n", rc);
        return NULL;
    }

    rc = libuboot_open(ctx);
    if (rc < 0) {
        fprintf(stderr, "libuboot_open failed: %d\n", rc);
        libuboot_exit(ctx);
        return NULL;
    }

    return ctx;
}

/* ─── Get current active slot ──────────────────────────────────── */
ab_slot_t ab_get_active_slot(void)
{
    struct uboot_ctx *ctx = open_uboot_env();
    if (!ctx) return SLOT_A;    /* safe default */

    const char *val = libuboot_get_env(ctx, "boot_slot");
    ab_slot_t   slot = (val && strcmp(val, "B") == 0) ? SLOT_B : SLOT_A;

    libuboot_close(ctx);
    libuboot_exit(ctx);
    return slot;
}

/* ─── Switch to the other slot ─────────────────────────────────── */
int ab_switch_slot(void)
{
    struct uboot_ctx *ctx = open_uboot_env();
    if (!ctx) return -1;

    ab_slot_t current = ab_get_active_slot();
    const char *new_slot_str = (current == SLOT_A) ? "B" : "A";

    int rc = libuboot_set_env(ctx, "boot_slot",          new_slot_str);
    rc    |= libuboot_set_env(ctx, "upgrade_available",  "1");
    rc    |= libuboot_set_env(ctx, "bootcount",          "0");

    if (rc == 0)
        rc = libuboot_env_store(ctx);

    printf("[A/B] Switched active slot to %s\n", new_slot_str);

    libuboot_close(ctx);
    libuboot_exit(ctx);
    return rc;
}

/* ─── Confirm current boot as healthy ──────────────────────────── */
int ab_confirm_boot(void)
{
    struct uboot_ctx *ctx = open_uboot_env();
    if (!ctx) return -1;

    int rc = libuboot_set_env(ctx, "upgrade_available", "0");
    rc    |= libuboot_set_env(ctx, "bootcount",         "0");

    if (rc == 0)
        rc = libuboot_env_store(ctx);

    if (rc == 0)
        printf("[A/B] Boot confirmed good — rollback protection active.\n");
    else
        fprintf(stderr, "[A/B] Failed to confirm boot!\n");

    libuboot_close(ctx);
    libuboot_exit(ctx);
    return rc;
}

int main(void)
{
    ab_slot_t slot = ab_get_active_slot();
    printf("Current active slot: %s\n", slot == SLOT_A ? "A" : "B");

    /* After successful startup checks, confirm the boot */
    if (ab_confirm_boot() == 0)
        printf("System is stable — no rollback will occur.\n");

    return 0;
}
```

### 8.2 Watchdog-Integrated Update Guard

```c
/* update_watchdog_guard.c
 * During an OTA update, kick the hardware watchdog to prevent
 * spurious resets. Stop kicking on completion to allow timeout
 * detection if the update hangs.
 */

#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <pthread.h>
#include <stdatomic.h>
#include <sys/ioctl.h>
#include <linux/watchdog.h>

#define WDT_DEVICE      "/dev/watchdog"
#define WDT_TIMEOUT_SEC 30
#define WDT_KICK_INTERVAL_SEC 10

static atomic_int  g_update_active = 0;
static int         g_wdt_fd        = -1;

/* ─── Watchdog keeper thread ───────────────────────────────────── */
static void *watchdog_thread(void *arg)
{
    (void)arg;

    g_wdt_fd = open(WDT_DEVICE, O_WRONLY);
    if (g_wdt_fd < 0) {
        perror("open watchdog");
        return NULL;
    }

    /* Set watchdog timeout */
    int timeout = WDT_TIMEOUT_SEC;
    if (ioctl(g_wdt_fd, WDIOC_SETTIMEOUT, &timeout) < 0)
        perror("WDIOC_SETTIMEOUT (non-fatal)");

    while (atomic_load(&g_update_active)) {
        /* Kick the watchdog */
        if (write(g_wdt_fd, "1", 1) != 1)
            perror("watchdog kick");

        sleep(WDT_KICK_INTERVAL_SEC);
    }

    /* Send magic close character — stops WDT if CONFIG_WATCHDOG_NOWAYOUT=n */
    write(g_wdt_fd, "V", 1);
    close(g_wdt_fd);
    g_wdt_fd = -1;

    printf("[WDT] Watchdog keeper stopped.\n");
    return NULL;
}

/* ─── Public API ───────────────────────────────────────────────── */
static pthread_t g_wdt_thread;

int update_guard_start(void)
{
    atomic_store(&g_update_active, 1);
    if (pthread_create(&g_wdt_thread, NULL, watchdog_thread, NULL) != 0) {
        perror("pthread_create");
        return -1;
    }
    printf("[WDT] Watchdog guard started (kick every %ds).\n",
           WDT_KICK_INTERVAL_SEC);
    return 0;
}

void update_guard_stop(void)
{
    atomic_store(&g_update_active, 0);
    pthread_join(g_wdt_thread, NULL);
}

int main(void)
{
    printf("Starting update process...\n");
    update_guard_start();

    /* Simulate a long-running flash write operation */
    sleep(25);

    printf("Update complete. Releasing watchdog guard.\n");
    update_guard_stop();

    return 0;
}
```

### 8.3 Flash Write with Progress and Verification

```c
/* flash_writer.c
 * Write an update image to a block device with progress reporting
 * and SHA-256 integrity verification.
 */

#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/stat.h>
#include <openssl/sha.h>

#define BLOCK_SIZE   (64 * 1024)    /* 64 KB write chunks */

typedef struct {
    unsigned char expected[SHA256_DIGEST_LENGTH];
    int           verify_enabled;
} flash_write_opts_t;

/* ─── Progress bar (ASCII) ─────────────────────────────────────── */
static void print_progress(const char *label,
                            size_t written, size_t total)
{
    int   pct   = (total > 0) ? (int)((written * 100) / total) : 0;
    int   bars  = pct / 5;    /* 20 chars wide */

    printf("\r  %s [", label);
    for (int i = 0; i < 20; i++)
        putchar(i < bars ? '#' : '-');
    printf("] %3d%%  (%zu / %zu MB)",
           pct,
           written / (1024 * 1024),
           total   / (1024 * 1024));
    fflush(stdout);
}

/* ─── Write image to device ────────────────────────────────────── */
int flash_write_image(const char *src_path,
                      const char *dst_dev,
                      const flash_write_opts_t *opts)
{
    int    src_fd  = -1, dst_fd  = -1;
    char  *buf     = NULL;
    int    ret     = -1;
    SHA256_CTX sha_ctx;

    /* Stat source */
    struct stat st;
    if (stat(src_path, &st) < 0) { perror("stat src"); goto out; }
    size_t total_bytes = (size_t)st.st_size;

    src_fd = open(src_path, O_RDONLY);
    if (src_fd < 0) { perror("open src"); goto out; }

    dst_fd = open(dst_dev, O_WRONLY | O_SYNC);
    if (dst_fd < 0) { perror("open dst"); goto out; }

    buf = malloc(BLOCK_SIZE);
    if (!buf) { perror("malloc"); goto out; }

    SHA256_Init(&sha_ctx);
    size_t written = 0;
    ssize_t nr;

    printf("\n  Writing %s -> %s\n", src_path, dst_dev);

    while ((nr = read(src_fd, buf, BLOCK_SIZE)) > 0) {
        ssize_t nw = write(dst_fd, buf, (size_t)nr);
        if (nw != nr) { fprintf(stderr, "\nWrite error!\n"); goto out; }

        SHA256_Update(&sha_ctx, buf, (size_t)nr);
        written += (size_t)nr;
        print_progress("Writing", written, total_bytes);
    }

    if (nr < 0) { perror("\nread"); goto out; }

    /* Flush to device */
    fsync(dst_fd);
    printf("\n");

    /* Verify SHA-256 */
    if (opts && opts->verify_enabled) {
        unsigned char digest[SHA256_DIGEST_LENGTH];
        SHA256_Final(digest, &sha_ctx);

        if (memcmp(digest, opts->expected, SHA256_DIGEST_LENGTH) == 0) {
            printf("  [SHA256] Integrity check PASSED.\n");
            ret = 0;
        } else {
            fprintf(stderr, "  [SHA256] Integrity check FAILED!\n");
            ret = -1;
        }
    } else {
        ret = 0;
    }

out:
    free(buf);
    if (src_fd >= 0) close(src_fd);
    if (dst_fd >= 0) close(dst_fd);
    return ret;
}

int main(void)
{
    flash_write_opts_t opts = {
        .verify_enabled = 0,   /* Set to 1 with real SHA-256 digest */
    };
    return flash_write_image("rootfs.ext4",
                             "/dev/mmcblk0p3",
                             &opts);
}
```

---

## 9. C++ Programming Examples

### 9.1 OTA Update State Machine

```cpp
// ota_state_machine.cpp
// A C++ state machine managing the full OTA lifecycle

#include <iostream>
#include <functional>
#include <unordered_map>
#include <string>
#include <stdexcept>
#include <chrono>
#include <thread>

// ─── States ────────────────────────────────────────────────────────
enum class OtaState {
    IDLE,
    DOWNLOADING,
    VERIFYING,
    FLASHING,
    REBOOTING,
    CONFIRMING,
    ROLLING_BACK,
    FAILED,
    UPDATED
};

std::string state_name(OtaState s) {
    switch (s) {
        case OtaState::IDLE:          return "IDLE";
        case OtaState::DOWNLOADING:   return "DOWNLOADING";
        case OtaState::VERIFYING:     return "VERIFYING";
        case OtaState::FLASHING:      return "FLASHING";
        case OtaState::REBOOTING:     return "REBOOTING";
        case OtaState::CONFIRMING:    return "CONFIRMING";
        case OtaState::ROLLING_BACK:  return "ROLLING_BACK";
        case OtaState::FAILED:        return "FAILED";
        case OtaState::UPDATED:       return "UPDATED";
        default:                       return "UNKNOWN";
    }
}

// ─── Events ────────────────────────────────────────────────────────
enum class OtaEvent {
    UPDATE_AVAILABLE,
    DOWNLOAD_OK,
    DOWNLOAD_FAIL,
    VERIFY_OK,
    VERIFY_FAIL,
    FLASH_OK,
    FLASH_FAIL,
    REBOOT_OK,
    CONFIRM_OK,
    CONFIRM_TIMEOUT,
    ROLLBACK_OK
};

// ─── Transition table ───────────────────────────────────────────────
struct Transition {
    OtaState from;
    OtaEvent event;
    OtaState to;
};

static const Transition TRANSITIONS[] = {
    { OtaState::IDLE,         OtaEvent::UPDATE_AVAILABLE, OtaState::DOWNLOADING  },
    { OtaState::DOWNLOADING,  OtaEvent::DOWNLOAD_OK,      OtaState::VERIFYING    },
    { OtaState::DOWNLOADING,  OtaEvent::DOWNLOAD_FAIL,    OtaState::FAILED       },
    { OtaState::VERIFYING,    OtaEvent::VERIFY_OK,        OtaState::FLASHING     },
    { OtaState::VERIFYING,    OtaEvent::VERIFY_FAIL,      OtaState::FAILED       },
    { OtaState::FLASHING,     OtaEvent::FLASH_OK,         OtaState::REBOOTING    },
    { OtaState::FLASHING,     OtaEvent::FLASH_FAIL,       OtaState::ROLLING_BACK },
    { OtaState::REBOOTING,    OtaEvent::REBOOT_OK,        OtaState::CONFIRMING   },
    { OtaState::CONFIRMING,   OtaEvent::CONFIRM_OK,       OtaState::UPDATED      },
    { OtaState::CONFIRMING,   OtaEvent::CONFIRM_TIMEOUT,  OtaState::ROLLING_BACK },
    { OtaState::ROLLING_BACK, OtaEvent::ROLLBACK_OK,      OtaState::IDLE         },
};

// ─── State machine class ────────────────────────────────────────────
class OtaStateMachine {
public:
    using Action = std::function<void()>;

    explicit OtaStateMachine(OtaState initial = OtaState::IDLE)
        : m_state(initial) {}

    void on_entry(OtaState s, Action action) {
        m_entry_actions[s] = std::move(action);
    }

    bool send_event(OtaEvent event) {
        for (const auto &t : TRANSITIONS) {
            if (t.from == m_state && t.event == event) {
                std::cout << "  [FSM] " << state_name(m_state)
                          << " ---> " << state_name(t.to) << "\n";
                m_state = t.to;

                auto it = m_entry_actions.find(t.to);
                if (it != m_entry_actions.end())
                    it->second();

                return true;
            }
        }
        std::cerr << "  [FSM] No transition from "
                  << state_name(m_state) << " for this event!\n";
        return false;
    }

    OtaState state() const { return m_state; }

private:
    OtaState m_state;
    std::unordered_map<OtaState, Action> m_entry_actions;
};

// ─── Demo ───────────────────────────────────────────────────────────
int main()
{
    OtaStateMachine fsm;

    fsm.on_entry(OtaState::DOWNLOADING,  []{ std::cout << "  Starting download...\n"; });
    fsm.on_entry(OtaState::VERIFYING,    []{ std::cout << "  Verifying signature...\n"; });
    fsm.on_entry(OtaState::FLASHING,     []{ std::cout << "  Writing to inactive slot...\n"; });
    fsm.on_entry(OtaState::REBOOTING,    []{ std::cout << "  Setting U-Boot env, rebooting...\n"; });
    fsm.on_entry(OtaState::CONFIRMING,   []{ std::cout << "  Running post-boot health checks...\n"; });
    fsm.on_entry(OtaState::UPDATED,      []{ std::cout << "  Update COMPLETE. Slot confirmed good.\n"; });
    fsm.on_entry(OtaState::ROLLING_BACK, []{ std::cout << "  ROLLING BACK to previous slot!\n"; });
    fsm.on_entry(OtaState::FAILED,       []{ std::cout << "  Update FAILED.\n"; });

    std::cout << "\n=== OTA Update Simulation ===\n\n";

    // Happy path
    fsm.send_event(OtaEvent::UPDATE_AVAILABLE);
    fsm.send_event(OtaEvent::DOWNLOAD_OK);
    fsm.send_event(OtaEvent::VERIFY_OK);
    fsm.send_event(OtaEvent::FLASH_OK);
    fsm.send_event(OtaEvent::REBOOT_OK);
    fsm.send_event(OtaEvent::CONFIRM_OK);

    std::cout << "\nFinal state: " << state_name(fsm.state()) << "\n";
    return 0;
}
```

### 9.2 RAUC D-Bus Client in C++ (using sdbus-c++)

```cpp
// rauc_cpp_client.cpp
// Monitor RAUC installation progress using sdbus-c++
// Compile: g++ -std=c++17 -lsdbus-c++ rauc_cpp_client.cpp -o rauc_client

#include <sdbus-c++/sdbus-c++.h>
#include <iostream>
#include <string>
#include <iomanip>

constexpr auto RAUC_SERVICE    = "de.pengutronix.rauc";
constexpr auto RAUC_PATH       = "/";
constexpr auto RAUC_IFACE      = "de.pengutronix.rauc.Installer";
constexpr auto PROPS_IFACE     = "org.freedesktop.DBus.Properties";

class RaucClient {
public:
    explicit RaucClient(const std::string &bundle_path)
        : m_bundle_path(bundle_path)
        , m_conn(sdbus::createSystemBusConnection())
        , m_proxy(sdbus::createProxy(*m_conn, RAUC_SERVICE, RAUC_PATH))
    {
        // Subscribe to PropertiesChanged for progress
        m_proxy->uponSignal("PropertiesChanged")
               .onInterface(PROPS_IFACE)
               .call([this](const std::string &iface,
                            const std::map<std::string,
                                           sdbus::Variant> &changed,
                            const std::vector<std::string>&) {
                   if (iface != RAUC_IFACE) return;
                   this->on_properties_changed(changed);
               });

        // Subscribe to Completed signal
        m_proxy->uponSignal("Completed")
               .onInterface(RAUC_IFACE)
               .call([this](int32_t result) {
                   m_completed = true;
                   m_result    = result;
               });

        m_proxy->finishRegistration();
    }

    int install_and_wait()
    {
        std::cout << "Installing bundle: " << m_bundle_path << "\n\n";

        // Kick off install
        m_proxy->callMethod("InstallBundle")
               .onInterface(RAUC_IFACE)
               .withArguments(m_bundle_path,
                              std::map<std::string, sdbus::Variant>{})
               .dontExpectReply();

        // Event loop until Completed fires
        while (!m_completed) {
            m_conn->processPendingEvent();
        }

        std::cout << "\n\nInstall " << (m_result == 0 ? "SUCCEEDED" : "FAILED")
                  << " (code=" << m_result << ")\n";
        return m_result;
    }

private:
    void on_properties_changed(
            const std::map<std::string, sdbus::Variant> &changed)
    {
        auto it = changed.find("Progress");
        if (it == changed.end()) return;

        // Progress is (percentage, message, nesting_depth)
        auto [pct, msg, depth] = it->second.get<
            sdbus::Struct<int32_t, std::string, int32_t>>();

        // Simple ASCII progress bar
        int bars = pct / 5;
        std::cout << "\r  [";
        for (int i = 0; i < 20; ++i)
            std::cout << (i < bars ? '#' : '-');
        std::cout << "] " << std::setw(3) << pct << "%  " << msg
                  << "          " << std::flush;
    }

    std::string                     m_bundle_path;
    std::unique_ptr<sdbus::IConnection> m_conn;
    std::unique_ptr<sdbus::IProxy>  m_proxy;
    bool                            m_completed = false;
    int32_t                         m_result    = -1;
};

int main(int argc, char *argv[])
{
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <bundle.raucb>\n";
        return 1;
    }
    return RaucClient(argv[1]).install_and_wait();
}
```

---

## 10. Rust Programming Examples

### 10.1 OTA HTTP Downloader with Integrity Check

```rust
// ota_downloader/src/main.rs
// Download an OTA bundle over HTTPS with SHA-256 verification
// Cargo.toml deps: reqwest = { features = ["blocking","stream"] }
//                  sha2, hex, indicatif, thiserror

use sha2::{Sha256, Digest};
use std::{
    fs::{self, File},
    io::{self, Write, BufWriter},
    path::Path,
};
use thiserror::Error;

#[derive(Error, Debug)]
pub enum OtaError {
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),
    #[error("I/O error: {0}")]
    Io(#[from] io::Error),
    #[error("Integrity check failed — expected {expected}, got {actual}")]
    IntegrityFail { expected: String, actual: String },
}

pub type Result<T> = std::result::Result<T, OtaError>;

/// Download a bundle, verify its SHA-256, and save to disk.
pub fn download_bundle(
    url:      &str,
    dest:     &Path,
    expected_sha256: &str,
) -> Result<()> {
    println!("Downloading: {}", url);

    let client   = reqwest::blocking::Client::new();
    let mut resp = client.get(url).send()?;
    let total    = resp.content_length().unwrap_or(0);

    // Progress bar
    let bar = indicatif::ProgressBar::new(total);
    bar.set_style(
        indicatif::ProgressStyle::default_bar()
            .template("  [{bar:40}] {bytes}/{total_bytes} ({eta})")
            .unwrap()
            .progress_chars("##-"),
    );

    let tmp_path = dest.with_extension("part");
    let file     = File::create(&tmp_path)?;
    let mut writer = BufWriter::new(file);
    let mut hasher = Sha256::new();
    let mut buf    = vec![0u8; 65536];

    loop {
        let n = io::Read::read(&mut resp, &mut buf)?;
        if n == 0 { break; }
        writer.write_all(&buf[..n])?;
        hasher.update(&buf[..n]);
        bar.inc(n as u64);
    }
    bar.finish();
    writer.flush()?;

    // Verify digest
    let actual_hex = format!("{:x}", hasher.finalize());
    if actual_hex != expected_sha256 {
        fs::remove_file(&tmp_path)?;
        return Err(OtaError::IntegrityFail {
            expected: expected_sha256.to_string(),
            actual:   actual_hex,
        });
    }

    println!("  [SHA256] Integrity OK.");
    fs::rename(&tmp_path, dest)?;
    Ok(())
}

fn main() {
    let url      = "https://ota.example.com/releases/2.0.0/update.raucb";
    let dest     = Path::new("/tmp/update.raucb");
    let expected = "3d90d3a71fe..."; // real SHA-256 here

    match download_bundle(url, dest, expected) {
        Ok(())  => println!("Bundle ready at {:?}", dest),
        Err(e)  => eprintln!("Download failed: {}", e),
    }
}
```

### 10.2 A/B Slot Manager in Rust

```rust
// ab_slot_manager/src/lib.rs
// Safe Rust wrapper for A/B slot management via U-Boot env
// or via RAUC status file parsing

use std::{
    fs,
    io::{self, BufRead},
    path::Path,
    process::Command,
};
use thiserror::Error;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Slot { A, B }

impl Slot {
    pub fn name(&self)    -> &'static str { match self { Slot::A => "A", Slot::B => "B" } }
    pub fn other(&self)   -> Slot         { match self { Slot::A => Slot::B, Slot::B => Slot::A } }
    pub fn device(&self)  -> &'static str {
        match self {
            Slot::A => "/dev/mmcblk0p2",
            Slot::B => "/dev/mmcblk0p3",
        }
    }
}

#[derive(Error, Debug)]
pub enum SlotError {
    #[error("I/O error: {0}")]
    Io(#[from] io::Error),
    #[error("fw_printenv failed: {0}")]
    EnvRead(String),
    #[error("fw_setenv failed: {0}")]
    EnvWrite(String),
}

pub type Result<T> = std::result::Result<T, SlotError>;

// ─── Read current active slot ────────────────────────────────────
pub fn get_active_slot() -> Result<Slot> {
    let output = Command::new("fw_printenv")
        .arg("boot_slot")
        .output()
        .map_err(|e| SlotError::EnvRead(e.to_string()))?;

    if !output.status.success() {
        return Err(SlotError::EnvRead(
            String::from_utf8_lossy(&output.stderr).into_owned()
        ));
    }

    let line = String::from_utf8_lossy(&output.stdout);
    // fw_printenv returns "boot_slot=B\n"
    Ok(if line.contains("=B") { Slot::B } else { Slot::A })
}

// ─── Commit a newly flashed slot ────────────────────────────────
pub fn commit_slot(slot: Slot) -> Result<()> {
    let vars = [
        ("boot_slot",          slot.name()),
        ("upgrade_available",  "1"),
        ("bootcount",          "0"),
    ];

    for (key, val) in &vars {
        let status = Command::new("fw_setenv")
            .arg(key).arg(val)
            .status()
            .map_err(|e| SlotError::EnvWrite(e.to_string()))?;

        if !status.success() {
            return Err(SlotError::EnvWrite(
                format!("fw_setenv {} {} failed", key, val)
            ));
        }
    }

    println!("[Slot] Committed slot {} as next boot target.", slot.name());
    Ok(())
}

// ─── Confirm successful boot (called by health-check daemon) ────
pub fn confirm_boot() -> Result<()> {
    let vars = [
        ("upgrade_available", "0"),
        ("bootcount",          "0"),
    ];

    for (key, val) in &vars {
        let status = Command::new("fw_setenv")
            .arg(key).arg(val)
            .status()
            .map_err(|e| SlotError::EnvWrite(e.to_string()))?;

        if !status.success() {
            return Err(SlotError::EnvWrite(format!("fw_setenv {} {}", key, val)));
        }
    }

    println!("[Slot] Boot confirmed. Rollback protection is active.");
    Ok(())
}

// ─── Health check before confirmation ───────────────────────────
pub fn run_health_checks() -> bool {
    let checks: &[(&str, &[&str])] = &[
        ("systemctl", &["is-active", "--quiet", "networking.service"]),
        ("systemctl", &["is-active", "--quiet", "myapp.service"]),
    ];

    for (cmd, args) in checks {
        match Command::new(cmd).args(*args).status() {
            Ok(s) if s.success() => println!("  [OK] {} {:?}", cmd, args),
            _ => {
                eprintln!("  [FAIL] Health check failed: {} {:?}", cmd, args);
                return false;
            }
        }
    }
    true
}

// ─── Main ────────────────────────────────────────────────────────
fn main() {
    match get_active_slot() {
        Ok(slot) => println!("Active slot: {}", slot.name()),
        Err(e)   => eprintln!("Could not read slot: {}", e),
    }

    if run_health_checks() {
        match confirm_boot() {
            Ok(())  => println!("System confirmed healthy."),
            Err(e)  => eprintln!("Confirm failed: {}", e),
        }
    } else {
        eprintln!("Health checks FAILED — system will roll back on next reboot!");
        // Do NOT call confirm_boot — bootloader will roll back automatically
    }
}
```

### 10.3 SWUpdate IPC Progress Monitor in Rust

```rust
// swupdate_monitor/src/main.rs
// Connect to SWUpdate's progress IPC socket and display real-time progress
// SWUpdate exposes a UNIX socket at /tmp/swupdateprog

use std::{
    io::{self, Read},
    os::unix::net::UnixStream,
};

/// Must match SWUpdate's progress_msg struct (from progress_ipc.h)
#[repr(C, packed)]
#[derive(Debug, Clone, Copy)]
struct ProgressMsg {
    magic:     u32,
    status:    u32,    // IDLE=0, START=1, RUN=2, SUCCESS=3, FAILURE=4, DOWNLOAD=5
    dwl_pct:   u32,    // Download progress 0-100
    nsteps:    u32,    // Total handler steps
    cur_step:  u32,    // Current step
    cur_pct:   u32,    // Current step progress 0-100
    cur_image: [u8; 256],
    hnd_name:  [u8; 64],
    source:    u32,
    infolen:   u32,
    info:      [u8; 2048],
}

const PROGRESS_MAGIC: u32 = 0xDEADBEEF;
const PROGRESS_SOCKET: &str = "/tmp/swupdateprog";

fn c_str_to_string(bytes: &[u8]) -> String {
    let end = bytes.iter().position(|&b| b == 0).unwrap_or(bytes.len());
    String::from_utf8_lossy(&bytes[..end]).to_string()
}

fn draw_progress(label: &str, pct: u32) {
    let bars = (pct / 5) as usize;
    print!("\r  {} [", label);
    for i in 0..20 {
        print!("{}", if i < bars { '#' } else { '-' });
    }
    print!("] {:3}%   ", pct);
    io::Write::flush(&mut io::stdout()).ok();
}

fn main() -> io::Result<()> {
    println!("Connecting to SWUpdate progress socket...");
    let mut stream = UnixStream::connect(PROGRESS_SOCKET)?;
    println!("Connected. Waiting for update activity.\n");

    let msg_size = std::mem::size_of::<ProgressMsg>();
    let mut buf  = vec![0u8; msg_size];

    loop {
        stream.read_exact(&mut buf)?;

        // SAFETY: repr(C, packed) struct with same layout as SWUpdate's C struct
        let msg: ProgressMsg = unsafe {
            std::ptr::read_unaligned(buf.as_ptr() as *const ProgressMsg)
        };

        if msg.magic != PROGRESS_MAGIC { continue; }

        let image = c_str_to_string(&msg.cur_image);

        match msg.status {
            1 => println!("\n[SWUpdate] Update STARTED"),
            2 => {
                let label = format!("Step {}/{} {}", msg.cur_step, msg.nsteps, image);
                draw_progress(&label, msg.cur_pct);
            }
            3 => { println!("\n[SWUpdate] Update SUCCEEDED!"); break; }
            4 => { println!("\n[SWUpdate] Update FAILED!"); break; }
            5 => draw_progress("Downloading", msg.dwl_pct),
            _ => {}
        }
    }

    println!();
    Ok(())
}
```

---

## 11. Security Considerations

### 11.1 Key Management and Trust Chain

```
COMPLETE OTA SECURITY TRUST CHAIN
══════════════════════════════════════════════════════════════
  ┌─────────────────────────────────────────────────────────┐
  │              Hardware Root of Trust                     │
  │  [e-fuse / OTP: Device Identity Key Hash]               │
  └────────────────────────┬────────────────────────────────┘
                           │ Verifies
                           v
  ┌─────────────────────────────────────────────────────────┐
  │              Secure Boot Chain                          │
  │  SPL ──> U-Boot ──> Kernel (signed FIT image)           │
  └────────────────────────┬────────────────────────────────┘
                           │ Trusts keyring at
                           v
  ┌─────────────────────────────────────────────────────────┐
  │         OTA Bundle Verification (RAUC/SWUpdate)         │
  │                                                         │
  │  Bundle ──[CMS/PKCS#7 sig]──> Keyring in /etc/rauc/     │
  │                               (verified at boot by      │
  │                                secure boot chain)       │
  └─────────────────────────────────────────────────────────┘
```

### 11.2 Key Hierarchy Recommendation

```
PKI KEY HIERARCHY — OTA SIGNING
──────────────────────────────────────────────────────────
  [ROOT CA]  (offline, HSM, air-gapped)
      |
      +── [INTERMEDIATE CA]  (online signing infra)
              |
              +── [RELEASE SIGNING CERT]  (CI/CD pipeline)
              |       Signs production bundles
              |
              +── [DEVELOPER CERT]  (developer machines)
                      Signs development bundles
                      NOT trusted in production keyring
```

### 11.3 Security Hardening Checklist

```
OTA SECURITY HARDENING CHECKLIST
══════════════════════════════════
  [ ] Bundle signing with RSA-4096 or EC P-384 minimum
  [ ] Keyring baked into rootfs (not updateable via OTA)
  [ ] Version anti-rollback counter in hardware (OP-TEE / e-fuse)
  [ ] HTTPS with certificate pinning for download transport
  [ ] Artifact authenticity verified BEFORE any flash write
  [ ] Update partition non-executable (NX bit / noexec mount)
  [ ] Watchdog active during flash operations
  [ ] Post-update health check before boot confirmation
  [ ] Audit log of all update events (journal / secure element)
  [ ] Signed U-Boot environment (libubootenv with HMAC)
```

---

## 12. Comparison and Selection Guide

```
OTA FRAMEWORK COMPARISON MATRIX
═══════════════════════════════════════════════════════════════════
  Feature              SWUpdate      RAUC          Mender
  ───────────────────  ────────────  ────────────  ─────────────
  Protocol             Push/Pull     Push/Pull     Pull (cloud)
  Server required      No            No            Yes (optional)
  D-Bus integration    Optional      Yes (native)  No
  Lua scripting        Yes           No            No
  Bundle signing       OpenSSL CMS   CMS (strong)  Mender CA
  Payload types        Many          Many          rootfs only*
  Bootloader support   U-Boot,Barbox U-Boot,GRUB,  U-Boot, GRUB
                       GRUB, EFI     Barebox, EFI
  Buildroot package    BR2_PKG_SWUPD BR2_PKG_RAUC  BR2_PKG_MENDER
  Complexity           Medium        Low-Medium    Low (SaaS)
  Best for             Flexible/     Systemd/      Managed fleet
                       industrial    Embedded      / cloud OTA
  ───────────────────  ────────────  ────────────  ─────────────
  * Mender supports application updates via Update Modules
```

---

## 13. Summary

OTA update reliability in embedded Linux systems built with Buildroot hinges on three pillars: **safe A/B partition switching**, **cryptographically signed bundles**, and **atomic commit with automatic rollback**.

**A/B partition switching** stores two complete system images on separate partitions. The bootloader maintains retry counters and only marks a new slot as permanently active once the application explicitly confirms it is healthy — otherwise it reverts automatically. The U-Boot environment (managed via `libubootenv` in C or the `fw_setenv`/`fw_printenv` utilities in Rust/shell) is the central mechanism for communicating slot state between application and bootloader.

**SWUpdate** is the most flexible framework, accepting `.swu` CPIO bundles described by a `sw-description` file in libconfig or Lua. It supports a wide range of input sources (HTTP, USB, TFTP, hawkBit cloud) and many payload types. Bundle integrity is enforced via OpenSSL CMS signatures. Its Lua scripting engine enables complex A/B detection and conditional install logic.

**RAUC** takes a stricter, more opinionated approach with deep systemd and D-Bus integration. Its `system.conf` declaratively maps slots to devices, and its signing pipeline is considered the most robust of the three for production environments, especially when combined with HSM-based signing. RAUC's D-Bus API makes it straightforward to integrate update logic into C, C++, or Python applications.

**Mender** provides the most complete managed OTA experience with a cloud backend, device authentication, deployment groups, and artifact management. Its client daemon polls a Mender server and handles the entire update lifecycle automatically. While the rootfs update mechanism is simpler than RAUC's, Mender Update Modules extend it to application and configuration updates.

All three frameworks are available as first-class Buildroot packages (enabled via `make menuconfig` under **Target packages → System tools**), and all three integrate with `genimage` for generating dual-partition disk images suitable for manufacturing and field deployment.

The selection decision reduces to: choose **SWUpdate** for maximum flexibility and industrial/special-case payloads, **RAUC** for clean systemd-based embedded Linux platforms where strong security guarantees are paramount, and **Mender** when a managed cloud fleet with minimal custom integration work is the priority.

---

*Document generated for Buildroot Embedded Linux Topic Series — Chapter 30*
*All code examples target Linux 6.x kernels on ARMv7/AArch64 targets.*