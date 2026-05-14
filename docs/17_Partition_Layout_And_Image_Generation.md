# 17. Partition Layout & Image Generation (genimage)

**Structure (13 sections):**

1. **Overview** — what genimage does and the Buildroot config knobs that enable it
2. **Pipeline diagram** — ASCII flowchart from `make all` through to physical media
3. **genimage.cfg reference** — minimal raw example, full GPT example, keyword table
4. **MBR vs GPT** — detailed ASCII byte-layout diagrams for both partition schemes
5. **SD-card images** — BeagleBone Black–style example with raw SPL placement
6. **WIC images** — sparse-flash-friendly `.wic` output with `bmaptool` usage
7. **post-image.sh** — canonical template, variable passing, advanced sign-and-compress example
8. **C examples** — MBR validator, raw partition writer, GPT header parser
9. **C++ examples** — `ImageLayout` class that generates `genimage.cfg` + ASCII layout renderer, CRC32 GPT verifier
10. **Rust examples** — MBR reader, cfg generator with ASCII layout, SHA-256 image verifier
11. **ASCII diagrams** — BeagleBone, RPi4, industrial GPT SBC, and genimage processing flow
12. **Advanced topics** — reproducible builds (fixed UUIDs), RAUC A/B slots, NOR/NAND flash, Buildroot Kconfig integration
13. **Summary** — quick-reference table and workflow recap


> **Buildroot Series — Topic 17**
> Covers: `genimage.cfg`, producing `.img` / `.wic` / SD-card images,
> GPT/MBR partition layouts, and `post-image.sh` scripting.

---

## Table of Contents

1. [Overview](#1-overview)
2. [How genimage Fits into the Buildroot Pipeline](#2-how-genimage-fits-into-the-buildroot-pipeline)
3. [genimage.cfg — Configuration Reference](#3-genimage-cfg--configuration-reference)
4. [MBR vs GPT Partition Layouts](#4-mbr-vs-gpt-partition-layouts)
5. [Producing SD-Card Images](#5-producing-sd-card-images)
6. [Producing WIC Images](#6-producing-wic-images)
7. [post-image.sh Scripting](#7-post-imagesh-scripting)
8. [C Programming Examples](#8-c-programming-examples)
9. [C++ Programming Examples](#9-c-programming-examples-1)
10. [Rust Programming Examples](#10-rust-programming-examples)
11. [ASCII Partition Diagrams](#11-ascii-partition-diagrams)
12. [Advanced Topics](#12-advanced-topics)
13. [Summary](#13-summary)

---

## 1. Overview

**genimage** is a tool — integrated into Buildroot — for creating binary disk images from
a declarative configuration file. It assembles file-system images, raw binary blobs,
bootloaders, and partition tables into a single `.img` file ready to be written to flash
storage, SD cards, eMMC, or NAND.

Key capabilities:

- Produces MBR (MS-DOS) and GPT partitioned images
- Supports ext2/3/4, FAT12/16/32, squashfs, jffs2, ubifs, raw, and custom file-systems
- Generates WIC images (compatible with `bmaptool` and Yocto-style flashing)
- Integrates with Buildroot's `post-image.sh` hook to allow arbitrary post-processing
- Handles alignment, offsets, paddings, and UUIDs for reproducible builds

Buildroot enables genimage via:

```
BR2_ROOTFS_POST_IMAGE_SCRIPT
BR2_PACKAGE_HOST_GENIMAGE
```

---

## 2. How genimage Fits into the Buildroot Pipeline

```
 Buildroot Build Pipeline
 ========================

  make all
     |
     v
 +-------------------+
 |  Toolchain        |  (cross-compiler, sysroot)
 +-------------------+
     |
     v
 +-------------------+
 |  Packages         |  (busybox, uboot, kernel, libs...)
 +-------------------+
     |
     v
 +-------------------+
 |  Filesystem       |  creates rootfs.tar, rootfs.ext4,
 |  Construction     |  rootfs.squashfs, etc.
 +-------------------+
     |
     v
 +-------------------+       genimage.cfg
 |  post-image.sh    |  <--  (declares partitions,
 |  (hook script)    |        images, layout)
 +-------------------+
     |
     v
 +-------------------+
 |  genimage         |  assembles final binary image
 +-------------------+
     |
     v
 +---------------------------+
 |  output/images/           |
 |    sdcard.img             |
 |    disk.img               |
 |    firmware.wic           |
 +---------------------------+
     |
     v
 dd / bmaptool / etcher --> physical media
```

Buildroot calls `post-image.sh` after all file-system images have been generated.
The script invokes `genimage`, which reads `genimage.cfg` and writes the final image
into `$(BINARIES_DIR)` (`output/images/`).

---

## 3. genimage.cfg — Configuration Reference

`genimage.cfg` is written in a libconfig-style syntax. It describes:

- **images**: named file-system or partition images
- **flash**: the target device (SD card, eMMC, NOR flash, etc.)

### 3.1 Minimal Example — Raw Image

```cfg
# genimage.cfg — minimal raw image example

image boot.vfat {
    vfat {
        files = {
            "zImage",
            "bcm2837-rpi-3-b.dtb",
        }
    }
    size = 32M
}

image rootfs.ext4 {
    ext4 {
        label = "rootfs"
    }
    size = 256M
}

image sdcard.img {
    hdimage {
        # MBR partition table (default)
    }

    partition boot {
        partition-type = 0xC    # FAT32 LBA
        bootable = true
        image = "boot.vfat"
    }

    partition rootfs {
        partition-type = 0x83   # Linux
        image = "rootfs.ext4"
    }
}
```

### 3.2 Full-Featured GPT Example

```cfg
# genimage.cfg — GPT with 3 partitions

image boot.vfat {
    vfat {
        files = {
            "u-boot.bin",
            "Image",
            "my-board.dtb",
            "extlinux/extlinux.conf",
        }
    }
    size = 64M
}

image rootfs.ext4 {
    ext4 {
        label        = "rootfs"
        use-mke2fs   = true
        mke2fs-extra-options = "-O 64bit,metadata_csum"
    }
    size = 512M
}

image data.ext4 {
    ext4 {
        label = "data"
    }
    size = 128M
}

image sdcard.img {
    hdimage {
        gpt = true                 # GPT instead of MBR
        gpt-location = 1M         # Reserve 1 MiB for MBR + GPT header
    }

    partition spl {
        # Raw partition for SPL / MLO
        partition-type-uuid = "8DA63339-0007-60C0-C436-083AC8230908"
        offset = 8K
        image  = "u-boot-spl.bin"
        size   = 512K
    }

    partition uboot {
        offset = 1M
        image  = "u-boot.itb"
        size   = 1M
    }

    partition boot {
        partition-type-uuid = "C12A7328-F81F-11D2-BA4B-00A0C93EC93B"  # EFI System
        bootable = true
        image    = "boot.vfat"
        offset   = 2M
    }

    partition rootfs {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"  # Linux fs
        image  = "rootfs.ext4"
        offset = 66M
    }

    partition data {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"
        image  = "data.ext4"
        # offset omitted → placed directly after rootfs
    }
}
```

### 3.3 Key genimage.cfg Keywords

| Keyword | Scope | Meaning |
|---------|-------|---------|
| `gpt = true` | `hdimage {}` | Use GPT partition table |
| `gpt-location` | `hdimage {}` | Offset for GPT header |
| `partition-type` | `partition {}` | MBR type byte (hex) |
| `partition-type-uuid` | `partition {}` | GPT partition type GUID |
| `bootable` | `partition {}` | Set MBR boot flag |
| `offset` | `partition {}` | Explicit start offset |
| `size` | `image {}` / `partition {}` | Size (K, M, G suffixes) |
| `label` | `ext4 {}` | File-system label |
| `files` | `vfat {}` | Files to include |
| `use-mke2fs` | `ext4 {}` | Use mke2fs instead of genext2fs |

---

## 4. MBR vs GPT Partition Layouts

### 4.1 MBR (Master Boot Record)

```
 MBR Disk Layout (512-byte sectors)
 ====================================

 Byte 0                                          Byte 511
 +-------------------+----------+----------------+------+
 | Bootstrap Code    | Disk Sig | Partition Table| 0x55 |
 |   (446 bytes)     | (4 bytes)| (4 x 16 bytes) | 0xAA |
 +-------------------+----------+----------------+------+

 Partition Table (4 entries max):
 +--------+------+--------+--------+-------+--------+------+--------+
 | Status | CHS  |  Type  |  CHS   |  LBA  |  Size  | ...  |  ...   |
 |  0x80  | Start| 0x0C   |  End   | Start | Sectors|      |        |
 | (boot) |      | (FAT32)|        |       |        |      |        |
 +--------+------+--------+--------+-------+--------+------+--------+

 Limitations:
   - Max 4 primary partitions (or 3 primary + 1 extended)
   - Max disk size: 2 TiB
   - No redundant table (single point of failure)
```

### 4.2 GPT (GUID Partition Table)

```
 GPT Disk Layout
 ================

 LBA 0   LBA 1        LBA 2..33        LBA 34..N-34     LBA N-33..N-2  LBA N-1
 +------+------------+---------------+------------------+---------------+------+
 | MBR  | Primary    | Partition     |   Data           | Backup Part.  | Bkup |
 | Prot | GPT Header | Entry Array   |   Partitions     | Entry Array   | HDR  |
 | 512B | 512 B      | (128 x 128 B) |                  | (128 x 128 B) | 512B |
 +------+------------+---------------+------------------+---------------+------+

 Each Partition Entry (128 bytes):
 +------------------+------------------+----------+----------+---------+--------+
 | Type GUID        | Unique GUID      | Start LBA| End LBA  | Attribs | Name   |
 |  (16 bytes)      |  (16 bytes)      | (8 bytes)|(8 bytes) |(8 bytes)|(72 B)  |
 +------------------+------------------+----------+----------+---------+--------+

 Advantages over MBR:
   - Up to 128 partitions (default)
   - Max disk size: 9.4 ZiB
   - CRC32 checksums on header and partition array
   - Backup GPT at end of disk
   - Human-readable partition names (UTF-16)
```

---

## 5. Producing SD-Card Images

A typical Raspberry Pi / BeagleBone-style SD card image with Buildroot:

```cfg
# board/myboard/genimage.cfg

image boot.vfat {
    vfat {
        files = {
            "MLO",
            "u-boot.img",
            "zImage",
            "am335x-boneblack.dtb",
        }
        extraargs = "-F 16"   # force FAT16
    }
    size = 16M
}

image rootfs.ext4 {
    ext4 {
        label = "rootfs"
        use-mke2fs = true
    }
    size = 512M
}

image sdcard.img {
    hdimage {
        # MBR layout; leave first 1MiB for SPL raw write
    }

    partition spl {
        # AM335x ROM code reads MLO from sector 256 / 512 / 768 / 1024
        in-partition-table = false
        image  = "MLO"
        offset = 128K
        size   = 128K
    }

    partition boot {
        partition-type = 0xC   # W95 FAT32 (LBA)
        bootable = true
        image    = "boot.vfat"
        offset   = 1M
        size     = 16M
    }

    partition rootfs {
        partition-type = 0x83  # Linux
        image  = "rootfs.ext4"
        offset = 17M
    }
}
```

---

## 6. Producing WIC Images

WIC (`.wic`) images are used with `bmaptool` for fast, sparse flashing.
Buildroot can call `genimage` to produce a WIC-compatible raw image, or
Buildroot's own wic plugin can be used.

```cfg
# genimage.cfg producing a .wic-named image (functionally identical to .img)
# The extension signals to flash tools that bmaptool should be used.

image firmware.wic {
    hdimage {
        gpt = true
    }

    partition boot {
        partition-type-uuid = "C12A7328-F81F-11D2-BA4B-00A0C93EC93B"
        image    = "boot.vfat"
        bootable = true
        size     = 64M
    }

    partition rootfs {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"
        image = "rootfs.squashfs"
    }

    partition overlay {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"
        image = "overlay.ext4"
        size  = 256M
    }
}
```

Flash with bmaptool (creates a `.bmap` sidecar for sparse writes):

```bash
bmaptool copy firmware.wic /dev/sdX
```

---

## 7. post-image.sh Scripting

Buildroot calls `post-image.sh` after building all file-system images.
Configure it in `menuconfig`:

```
System configuration
  └─> Custom scripts to run after creating filesystem images (post-image.sh)
```

Or in `BR2_ROOTFS_POST_IMAGE_SCRIPT` in your `.config`.

### 7.1 Canonical post-image.sh Template

```bash
#!/usr/bin/env bash
#
# board/myboard/post-image.sh
# Called by Buildroot after rootfs images are built.
#
# Environment variables provided by Buildroot:
#   BINARIES_DIR   — output/images/
#   BUILD_DIR      — output/build/
#   HOST_DIR       — output/host/
#   STAGING_DIR    — output/staging/
#   TARGET_DIR     — output/target/
#   BR2_CONFIG     — path to .config

set -euo pipefail

BOARD_DIR="$(dirname "$0")"
GENIMAGE_CFG="${BOARD_DIR}/genimage.cfg"
GENIMAGE_TMP="${BUILD_DIR}/genimage.tmp"

# Helper: clean temporary directory each run
rm -rf "${GENIMAGE_TMP}"

# Invoke genimage
# --rootpath must point to the *target* sysroot so genimage can
# resolve symlinks when creating ext4 images.
genimage \
    --config    "${GENIMAGE_CFG}"  \
    --rootpath  "${TARGET_DIR}"    \
    --tmppath   "${GENIMAGE_TMP}"  \
    --inputpath "${BINARIES_DIR}"  \
    --outputpath "${BINARIES_DIR}"

echo "[post-image] sdcard.img written to ${BINARIES_DIR}"
```

### 7.2 Passing Variables from Buildroot to post-image.sh

Buildroot passes `BR2_JLEVEL`, `BR2_DL_DIR`, etc. automatically.
Extra variables can be passed via `BR2_ROOTFS_POST_SCRIPT_ARGS`:

```
BR2_ROOTFS_POST_SCRIPT_ARGS="BOARD_REVISION=3 ENABLE_WIFI=y"
```

Inside the script:

```bash
BOARD_REVISION="${BOARD_REVISION:-1}"
ENABLE_WIFI="${ENABLE_WIFI:-n}"

if [ "${ENABLE_WIFI}" = "y" ]; then
    cp "${BOARD_DIR}/firmware/brcmfmac43455-sdio.bin" \
       "${BINARIES_DIR}/"
fi
```

### 7.3 Advanced post-image.sh — Sign and Encrypt

```bash
#!/usr/bin/env bash
set -euo pipefail

BOARD_DIR="$(dirname "$0")"
GENIMAGE_CFG="${BOARD_DIR}/genimage.cfg"
GENIMAGE_TMP="${BUILD_DIR}/genimage.tmp"
SIGNING_KEY="${BOARD_DIR}/keys/image-signing.key"

# Step 1: Build the raw disk image
rm -rf "${GENIMAGE_TMP}"
genimage \
    --config     "${GENIMAGE_CFG}"   \
    --rootpath   "${TARGET_DIR}"     \
    --tmppath    "${GENIMAGE_TMP}"   \
    --inputpath  "${BINARIES_DIR}"   \
    --outputpath "${BINARIES_DIR}"

# Step 2: Compute SHA-256 checksum
sha256sum "${BINARIES_DIR}/sdcard.img" \
    > "${BINARIES_DIR}/sdcard.img.sha256"

# Step 3: Optionally sign with GPG
if [ -f "${SIGNING_KEY}" ]; then
    gpg --batch --yes \
        --passphrase-file "${BOARD_DIR}/keys/passphrase.txt" \
        --output  "${BINARIES_DIR}/sdcard.img.sig" \
        --detach-sig "${BINARIES_DIR}/sdcard.img"
    echo "[post-image] Image signed: sdcard.img.sig"
fi

# Step 4: Compress for distribution
gzip -k -f "${BINARIES_DIR}/sdcard.img"
echo "[post-image] Compressed image: sdcard.img.gz"
```

---

## 8. C Programming Examples

### 8.1 Read and Validate an MBR

```c
/*
 * mbr_validate.c
 * Validates the MBR signature and prints partition table entries.
 * Build: gcc -o mbr_validate mbr_validate.c
 * Run:   ./mbr_validate /dev/sdX   (or sdcard.img)
 */

#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

#define MBR_SIGNATURE    0xAA55
#define MBR_PART_OFFSET  446
#define MBR_PART_COUNT   4

#pragma pack(push, 1)
typedef struct {
    uint8_t  status;          /* 0x80 = bootable, 0x00 = not */
    uint8_t  chs_first[3];   /* CHS address of first sector  */
    uint8_t  type;            /* Partition type               */
    uint8_t  chs_last[3];    /* CHS address of last sector   */
    uint32_t lba_start;      /* LBA of first sector          */
    uint32_t lba_size;       /* Number of sectors            */
} mbr_partition_t;

typedef struct {
    uint8_t          bootstrap[446];
    mbr_partition_t  partitions[4];
    uint16_t         signature;
} mbr_t;
#pragma pack(pop)

static const char *partition_type_name(uint8_t type)
{
    switch (type) {
    case 0x00: return "Empty";
    case 0x0B: return "FAT32";
    case 0x0C: return "FAT32 LBA";
    case 0x82: return "Linux swap";
    case 0x83: return "Linux";
    case 0x85: return "Linux extended";
    case 0xEE: return "GPT protective MBR";
    case 0xEF: return "EFI System";
    default:   return "Unknown";
    }
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <disk-image>\n", argv[0]);
        return 1;
    }

    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    mbr_t mbr;
    ssize_t n = read(fd, &mbr, sizeof(mbr));
    close(fd);

    if (n != sizeof(mbr)) {
        fprintf(stderr, "Short read (%zd bytes)\n", n);
        return 1;
    }

    if (mbr.signature != MBR_SIGNATURE) {
        fprintf(stderr, "Invalid MBR signature: 0x%04X\n", mbr.signature);
        return 1;
    }

    printf("MBR signature: 0x%04X (valid)\n\n", mbr.signature);
    printf("%-4s %-8s %-12s %-12s %-20s %-8s\n",
           "No.", "Status", "LBA Start", "LBA Size",
           "Type Name", "Type");
    printf("%-4s %-8s %-12s %-12s %-20s %-8s\n",
           "---", "------", "---------", "--------",
           "---------", "----");

    for (int i = 0; i < MBR_PART_COUNT; i++) {
        mbr_partition_t *p = &mbr.partitions[i];
        if (p->type == 0x00)
            continue;
        printf("%-4d %-8s %-12u %-12u %-20s 0x%02X\n",
               i + 1,
               (p->status == 0x80) ? "bootable" : "---",
               p->lba_start,
               p->lba_size,
               partition_type_name(p->type),
               p->type);
    }

    return 0;
}
```

### 8.2 Write Raw Partition Data

```c
/*
 * write_partition.c
 * Writes a binary blob to a specific byte offset in a disk image.
 * Useful for embedding SPL/MLO/U-Boot at fixed offsets.
 *
 * Build: gcc -o write_partition write_partition.c
 * Run:   ./write_partition sdcard.img u-boot-spl.bin 131072
 *                                                    ^ 128 KiB offset
 */

#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <errno.h>
#include <string.h>

#define COPY_BUFFER 65536   /* 64 KiB copy buffer */

int main(int argc, char *argv[])
{
    if (argc != 4) {
        fprintf(stderr,
            "Usage: %s <disk-image> <blob> <offset-bytes>\n", argv[0]);
        return 1;
    }

    const char *disk_path   = argv[1];
    const char *blob_path   = argv[2];
    off_t       offset      = (off_t)atoll(argv[3]);

    /* Open blob for reading */
    int blob_fd = open(blob_path, O_RDONLY);
    if (blob_fd < 0) { perror("open blob"); return 1; }

    struct stat st;
    if (fstat(blob_fd, &st) < 0) { perror("fstat"); return 1; }
    printf("Blob size: %lld bytes\n", (long long)st.st_size);

    /* Open disk image for writing (no truncate) */
    int disk_fd = open(disk_path, O_WRONLY);
    if (disk_fd < 0) { perror("open disk"); close(blob_fd); return 1; }

    /* Seek to offset */
    if (lseek(disk_fd, offset, SEEK_SET) != offset) {
        perror("lseek");
        close(blob_fd);
        close(disk_fd);
        return 1;
    }

    /* Copy blob into disk image */
    unsigned char buf[COPY_BUFFER];
    ssize_t total = 0;
    ssize_t n;

    while ((n = read(blob_fd, buf, sizeof(buf))) > 0) {
        ssize_t written = write(disk_fd, buf, (size_t)n);
        if (written != n) {
            fprintf(stderr, "Short write at offset %lld\n",
                    (long long)(offset + total));
            close(blob_fd);
            close(disk_fd);
            return 1;
        }
        total += written;
    }

    printf("Written %zd bytes at offset %lld (0x%llX)\n",
           total, (long long)offset, (long long)offset);

    close(blob_fd);
    close(disk_fd);
    return 0;
}
```

### 8.3 GPT Header Parser

```c
/*
 * gpt_info.c
 * Parses and prints the GPT header from a disk image.
 * Build: gcc -o gpt_info gpt_info.c
 * Run:   ./gpt_info sdcard.img
 */

#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

#define GPT_SIGNATURE  "EFI PART"
#define SECTOR_SIZE    512
#define GPT_LBA        1     /* GPT header is at LBA 1 */

#pragma pack(push, 1)
typedef struct {
    uint8_t  signature[8];
    uint32_t revision;
    uint32_t header_size;
    uint32_t header_crc32;
    uint32_t reserved;
    uint64_t my_lba;
    uint64_t alternate_lba;
    uint64_t first_usable_lba;
    uint64_t last_usable_lba;
    uint8_t  disk_guid[16];
    uint64_t partition_entry_lba;
    uint32_t num_partition_entries;
    uint32_t sizeof_partition_entry;
    uint32_t partition_entry_crc32;
} gpt_header_t;
#pragma pack(pop)

static void print_guid(const uint8_t guid[16])
{
    /* GUID mixed-endian format */
    printf("%08X-%04X-%04X-%02X%02X-%02X%02X%02X%02X%02X%02X",
        *(uint32_t*)&guid[0],
        *(uint16_t*)&guid[4],
        *(uint16_t*)&guid[6],
        guid[8], guid[9],
        guid[10], guid[11], guid[12], guid[13], guid[14], guid[15]);
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <disk-image>\n", argv[0]);
        return 1;
    }

    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    /* Skip LBA 0 (Protective MBR) and read LBA 1 */
    lseek(fd, (off_t)SECTOR_SIZE * GPT_LBA, SEEK_SET);

    gpt_header_t hdr;
    read(fd, &hdr, sizeof(hdr));
    close(fd);

    if (memcmp(hdr.signature, GPT_SIGNATURE, 8) != 0) {
        fprintf(stderr, "Not a GPT disk\n");
        return 1;
    }

    printf("GPT Header\n");
    printf("==========\n");
    printf("Revision           : %u.%u\n",
           hdr.revision >> 16, hdr.revision & 0xFFFF);
    printf("Header size        : %u bytes\n", hdr.header_size);
    printf("My LBA             : %llu\n", (unsigned long long)hdr.my_lba);
    printf("Alternate LBA      : %llu\n", (unsigned long long)hdr.alternate_lba);
    printf("First usable LBA   : %llu\n", (unsigned long long)hdr.first_usable_lba);
    printf("Last usable LBA    : %llu\n", (unsigned long long)hdr.last_usable_lba);
    printf("Disk GUID          : ");
    print_guid(hdr.disk_guid);
    printf("\n");
    printf("Partition entries  : %u (each %u bytes)\n",
           hdr.num_partition_entries,
           hdr.sizeof_partition_entry);
    return 0;
}
```

---

## 9. C++ Programming Examples

### 9.1 Image Layout Descriptor Class

```cpp
/*
 * ImageLayout.hpp / ImageLayout.cpp
 * C++ class for describing and serialising a disk image layout.
 * Useful for tooling that generates genimage.cfg programmatically.
 */

#include <cstdint>
#include <string>
#include <vector>
#include <iostream>
#include <fstream>
#include <sstream>
#include <stdexcept>
#include <iomanip>

enum class PartitionScheme { MBR, GPT };
enum class FsType           { VFAT, EXT4, SQUASHFS, RAW };

struct Partition {
    std::string name;
    FsType      fs_type;
    uint64_t    offset_bytes;
    uint64_t    size_bytes;
    std::string image_file;
    bool        bootable  = false;
    uint8_t     mbr_type  = 0x83;   /* Linux default */
    std::string gpt_uuid;

    std::string fs_type_str() const {
        switch (fs_type) {
        case FsType::VFAT:     return "vfat";
        case FsType::EXT4:     return "ext4";
        case FsType::SQUASHFS: return "squashfs";
        case FsType::RAW:      return "raw";
        }
        return "raw";
    }
};

class ImageLayout {
public:
    ImageLayout(const std::string &output_name, PartitionScheme scheme)
        : output_name_(output_name), scheme_(scheme)
    {}

    void add_partition(const Partition &p) {
        partitions_.push_back(p);
    }

    /* Generate a genimage.cfg snippet */
    std::string generate_config() const {
        std::ostringstream cfg;

        cfg << "# Auto-generated genimage.cfg\n\n";

        /* Per-partition image blocks */
        for (const auto &p : partitions_) {
            if (p.fs_type == FsType::RAW) continue;

            cfg << "image " << p.image_file << " {\n";
            cfg << "    " << p.fs_type_str() << " {\n";
            if (p.fs_type == FsType::EXT4)
                cfg << "        label = \"" << p.name << "\"\n";
            if (p.fs_type == FsType::VFAT)
                cfg << "        files = {}\n";
            cfg << "    }\n";
            cfg << "    size = " << (p.size_bytes / (1024*1024)) << "M\n";
            cfg << "}\n\n";
        }

        /* Master disk image block */
        cfg << "image " << output_name_ << " {\n";
        cfg << "    hdimage {\n";
        if (scheme_ == PartitionScheme::GPT)
            cfg << "        gpt = true\n";
        cfg << "    }\n\n";

        for (const auto &p : partitions_) {
            cfg << "    partition " << p.name << " {\n";
            if (scheme_ == PartitionScheme::GPT && !p.gpt_uuid.empty())
                cfg << "        partition-type-uuid = \""
                    << p.gpt_uuid << "\"\n";
            else
                cfg << "        partition-type = 0x"
                    << std::hex << static_cast<int>(p.mbr_type)
                    << std::dec << "\n";
            if (p.bootable)
                cfg << "        bootable = true\n";
            cfg << "        offset = " << p.offset_bytes << "\n";
            cfg << "        image  = \"" << p.image_file << "\"\n";
            cfg << "    }\n\n";
        }

        cfg << "}\n";
        return cfg.str();
    }

    void write_config(const std::string &path) const {
        std::ofstream f(path);
        if (!f)
            throw std::runtime_error("Cannot write: " + path);
        f << generate_config();
        std::cout << "Config written to " << path << "\n";
    }

    void print_ascii_layout() const {
        const int BAR_WIDTH = 60;
        uint64_t total = 0;
        for (const auto &p : partitions_)
            total += p.size_bytes;

        std::cout << "\nDisk Image Layout: " << output_name_ << "\n";
        std::cout << std::string(BAR_WIDTH + 20, '=') << "\n";

        for (const auto &p : partitions_) {
            int width = static_cast<int>(
                (double)p.size_bytes / total * BAR_WIDTH);
            if (width < 4) width = 4;

            std::string label = p.name;
            if ((int)label.size() > width - 2)
                label = label.substr(0, width - 2);

            std::cout << "| ";
            std::cout << std::left << std::setw(width - 2) << label;
            std::cout << " |";
        }
        std::cout << "\n";
        std::cout << std::string(BAR_WIDTH + 20, '-') << "\n";
        for (const auto &p : partitions_) {
            double mb = p.size_bytes / 1048576.0;
            std::cout << "  " << std::left << std::setw(16) << p.name
                      << std::right << std::setw(8) << std::fixed
                      << std::setprecision(1) << mb << " MiB"
                      << "  @ offset "
                      << p.offset_bytes << "\n";
        }
        std::cout << "\n";
    }

private:
    std::string             output_name_;
    PartitionScheme         scheme_;
    std::vector<Partition>  partitions_;
};

/* ---- main ---- */
int main()
{
    ImageLayout layout("sdcard.img", PartitionScheme::GPT);

    layout.add_partition({
        .name        = "boot",
        .fs_type     = FsType::VFAT,
        .offset_bytes = 2 * 1024 * 1024,
        .size_bytes  = 64 * 1024 * 1024,
        .image_file  = "boot.vfat",
        .bootable    = true,
        .gpt_uuid    = "C12A7328-F81F-11D2-BA4B-00A0C93EC93B",
    });

    layout.add_partition({
        .name        = "rootfs",
        .fs_type     = FsType::EXT4,
        .offset_bytes = 66 * 1024 * 1024,
        .size_bytes  = 512 * 1024 * 1024,
        .image_file  = "rootfs.ext4",
        .gpt_uuid    = "0FC63DAF-8483-4772-8E79-3D69D8477DE4",
    });

    layout.add_partition({
        .name        = "data",
        .fs_type     = FsType::EXT4,
        .offset_bytes = 578 * 1024 * 1024,
        .size_bytes  = 128 * 1024 * 1024,
        .image_file  = "data.ext4",
        .gpt_uuid    = "0FC63DAF-8483-4772-8E79-3D69D8477DE4",
    });

    layout.print_ascii_layout();
    layout.write_config("genimage_auto.cfg");

    return 0;
}
```

### 9.2 CRC32 Verifier for GPT

```cpp
/*
 * gpt_crc.cpp
 * Computes and verifies the CRC32 of a GPT header.
 * Build: g++ -std=c++17 -o gpt_crc gpt_crc.cpp
 */

#include <cstdint>
#include <cstring>
#include <fstream>
#include <iostream>
#include <array>
#include <vector>

/* Standard CRC-32 (ISO 3309) */
uint32_t crc32(const uint8_t *data, size_t length)
{
    uint32_t crc = 0xFFFFFFFF;
    while (length--) {
        crc ^= *data++;
        for (int k = 0; k < 8; k++)
            crc = (crc & 1) ? (crc >> 1) ^ 0xEDB88320 : (crc >> 1);
    }
    return crc ^ 0xFFFFFFFF;
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <disk-image>\n";
        return 1;
    }

    std::ifstream f(argv[1], std::ios::binary);
    if (!f) {
        std::cerr << "Cannot open " << argv[1] << "\n";
        return 1;
    }

    /* Read LBA 1 (GPT header, 512 bytes) */
    f.seekg(512);
    std::array<uint8_t, 512> sector{};
    f.read(reinterpret_cast<char*>(sector.data()), 512);

    if (std::memcmp(sector.data(), "EFI PART", 8) != 0) {
        std::cerr << "Not a GPT disk\n";
        return 1;
    }

    uint32_t header_size;
    std::memcpy(&header_size, sector.data() + 12, 4);

    uint32_t stored_crc;
    std::memcpy(&stored_crc, sector.data() + 16, 4);

    /* Zero-out CRC field before computing */
    std::vector<uint8_t> header_bytes(sector.begin(),
                                      sector.begin() + header_size);
    std::memset(header_bytes.data() + 16, 0, 4);

    uint32_t computed_crc = crc32(header_bytes.data(), header_size);

    std::cout << "Stored CRC32  : 0x" << std::hex << stored_crc   << "\n";
    std::cout << "Computed CRC32: 0x" << std::hex << computed_crc  << "\n";

    if (stored_crc == computed_crc)
        std::cout << "CRC32 VALID\n";
    else
        std::cout << "CRC32 MISMATCH — GPT header may be corrupt\n";

    return (stored_crc == computed_crc) ? 0 : 1;
}
```

---

## 10. Rust Programming Examples

### 10.1 MBR Reader in Rust

```rust
// mbr_reader.rs
// Reads and validates an MBR from a disk image.
// Run: cargo run -- sdcard.img
//
// Cargo.toml:
// [package]
// name = "mbr_reader"
// version = "0.1.0"
// edition = "2021"

use std::env;
use std::fs::File;
use std::io::{self, Read};

const MBR_SIGNATURE: u16 = 0xAA55;

#[repr(C, packed)]
#[derive(Copy, Clone)]
struct MbrPartition {
    status:    u8,
    chs_first: [u8; 3],
    part_type: u8,
    chs_last:  [u8; 3],
    lba_start: u32,
    lba_size:  u32,
}

#[repr(C, packed)]
struct Mbr {
    bootstrap:  [u8; 446],
    partitions: [MbrPartition; 4],
    signature:  u16,
}

fn partition_type_name(t: u8) -> &'static str {
    match t {
        0x00 => "Empty",
        0x0B => "FAT32",
        0x0C => "FAT32 LBA",
        0x82 => "Linux swap",
        0x83 => "Linux",
        0xEE => "GPT Protective",
        0xEF => "EFI System",
        _    => "Unknown",
    }
}

fn main() -> io::Result<()> {
    let path = env::args().nth(1).expect("Usage: mbr_reader <disk-image>");
    let mut f = File::open(&path)?;

    // Safety: MBR is exactly 512 bytes; read into a byte array then transmute.
    let mut buf = [0u8; 512];
    f.read_exact(&mut buf)?;

    // SAFETY: MBR is #[repr(C, packed)] and buf is exactly 512 bytes.
    let mbr: &Mbr = unsafe { &*(buf.as_ptr() as *const Mbr) };
    let sig = u16::from_le(mbr.signature);

    if sig != MBR_SIGNATURE {
        eprintln!("Invalid MBR signature: 0x{:04X}", sig);
        std::process::exit(1);
    }

    println!("MBR signature: 0x{:04X} (valid)\n", sig);
    println!("{:<4} {:<10} {:<12} {:<12} {:<20} {:<6}",
             "No.", "Status", "LBA Start", "LBA Size", "Type Name", "Type");
    println!("{}", "-".repeat(70));

    for (i, p) in mbr.partitions.iter().enumerate() {
        let part_type = p.part_type;
        if part_type == 0x00 { continue; }
        let lba_start = u32::from_le(p.lba_start);
        let lba_size  = u32::from_le(p.lba_size);
        let status    = if p.status == 0x80 { "bootable" } else { "---" };
        println!("{:<4} {:<10} {:<12} {:<12} {:<20} 0x{:02X}",
                 i + 1, status, lba_start, lba_size,
                 partition_type_name(part_type), part_type);
    }
    Ok(())
}
```

### 10.2 genimage.cfg Generator in Rust

```rust
// genimage_gen.rs
// Generates a genimage.cfg from a simple partition spec.
//
// Cargo.toml:
// [package]
// name    = "genimage_gen"
// version = "0.1.0"
// edition = "2021"
//
// [dependencies]
// # none required

use std::fmt::Write as FmtWrite;
use std::fs;

#[derive(Debug, Clone)]
enum PartitionScheme { Mbr, Gpt }

#[derive(Debug, Clone)]
enum FsType { Vfat, Ext4, Squashfs, Raw }

impl FsType {
    fn as_str(&self) -> &'static str {
        match self {
            FsType::Vfat     => "vfat",
            FsType::Ext4     => "ext4",
            FsType::Squashfs => "squashfs",
            FsType::Raw      => "raw",
        }
    }
}

#[derive(Debug, Clone)]
struct Partition {
    name:         String,
    fs_type:      FsType,
    offset_bytes: u64,
    size_mib:     u64,
    image_file:   String,
    bootable:     bool,
    mbr_type:     u8,
    gpt_uuid:     Option<String>,
}

struct ImageLayout {
    output_name: String,
    scheme:      PartitionScheme,
    partitions:  Vec<Partition>,
}

impl ImageLayout {
    fn new(output_name: &str, scheme: PartitionScheme) -> Self {
        Self {
            output_name: output_name.to_string(),
            scheme,
            partitions: Vec::new(),
        }
    }

    fn add_partition(&mut self, p: Partition) {
        self.partitions.push(p);
    }

    fn generate_config(&self) -> String {
        let mut cfg = String::new();
        writeln!(cfg, "# Auto-generated genimage.cfg\n").unwrap();

        for p in &self.partitions {
            if matches!(p.fs_type, FsType::Raw) { continue; }
            writeln!(cfg, "image {} {{", p.image_file).unwrap();
            writeln!(cfg, "    {} {{", p.fs_type.as_str()).unwrap();
            if matches!(p.fs_type, FsType::Ext4) {
                writeln!(cfg, "        label = \"{}\"", p.name).unwrap();
                writeln!(cfg, "        use-mke2fs = true").unwrap();
            }
            writeln!(cfg, "    }}").unwrap();
            writeln!(cfg, "    size = {}M", p.size_mib).unwrap();
            writeln!(cfg, "}}\n").unwrap();
        }

        writeln!(cfg, "image {} {{", self.output_name).unwrap();
        writeln!(cfg, "    hdimage {{").unwrap();
        if matches!(self.scheme, PartitionScheme::Gpt) {
            writeln!(cfg, "        gpt = true").unwrap();
        }
        writeln!(cfg, "    }}\n").unwrap();

        for p in &self.partitions {
            writeln!(cfg, "    partition {} {{", p.name).unwrap();
            match self.scheme {
                PartitionScheme::Gpt => {
                    if let Some(uuid) = &p.gpt_uuid {
                        writeln!(cfg, "        partition-type-uuid = \"{}\"",
                                 uuid).unwrap();
                    }
                }
                PartitionScheme::Mbr => {
                    writeln!(cfg, "        partition-type = 0x{:02X}",
                             p.mbr_type).unwrap();
                }
            }
            if p.bootable {
                writeln!(cfg, "        bootable = true").unwrap();
            }
            writeln!(cfg, "        offset = {}", p.offset_bytes).unwrap();
            writeln!(cfg, "        image  = \"{}\"", p.image_file).unwrap();
            writeln!(cfg, "    }}\n").unwrap();
        }

        writeln!(cfg, "}}").unwrap();
        cfg
    }

    fn print_ascii_layout(&self) {
        let total_mib: u64 = self.partitions.iter().map(|p| p.size_mib).sum();
        let bar_width: usize = 64;

        println!("\nLayout: {}", self.output_name);
        println!("{}", "=".repeat(bar_width + 4));
        print!("|");
        for p in &self.partitions {
            let w = ((p.size_mib as f64 / total_mib as f64) * bar_width as f64)
                        .round() as usize;
            let w = w.max(4);
            let label = if p.name.len() > w.saturating_sub(2) {
                &p.name[..w.saturating_sub(2)]
            } else {
                &p.name
            };
            print!("{:^width$}|", label, width = w);
        }
        println!();
        println!("{}", "-".repeat(bar_width + 4));
        for p in &self.partitions {
            println!("  {:<16}  {:>6} MiB  @ offset {} bytes",
                     p.name, p.size_mib, p.offset_bytes);
        }
        println!();
    }
}

fn main() {
    let mut layout = ImageLayout::new("sdcard.img", PartitionScheme::Gpt);

    layout.add_partition(Partition {
        name:         "boot".into(),
        fs_type:      FsType::Vfat,
        offset_bytes: 2 * 1024 * 1024,
        size_mib:     64,
        image_file:   "boot.vfat".into(),
        bootable:     true,
        mbr_type:     0x0C,
        gpt_uuid:     Some("C12A7328-F81F-11D2-BA4B-00A0C93EC93B".into()),
    });

    layout.add_partition(Partition {
        name:         "rootfs".into(),
        fs_type:      FsType::Ext4,
        offset_bytes: 66 * 1024 * 1024,
        size_mib:     512,
        image_file:   "rootfs.ext4".into(),
        bootable:     false,
        mbr_type:     0x83,
        gpt_uuid:     Some("0FC63DAF-8483-4772-8E79-3D69D8477DE4".into()),
    });

    layout.add_partition(Partition {
        name:         "data".into(),
        fs_type:      FsType::Ext4,
        offset_bytes: 578 * 1024 * 1024,
        size_mib:     128,
        image_file:   "data.ext4".into(),
        bootable:     false,
        mbr_type:     0x83,
        gpt_uuid:     Some("0FC63DAF-8483-4772-8E79-3D69D8477DE4".into()),
    });

    layout.print_ascii_layout();

    let config = layout.generate_config();
    fs::write("genimage_auto.cfg", &config).expect("write failed");
    println!("genimage.cfg written.");
}
```

### 10.3 SHA-256 Image Verifier in Rust

```rust
// img_verify.rs
// Computes SHA-256 of a disk image and compares to a .sha256 sidecar file.
//
// Cargo.toml [dependencies]:
// sha2 = "0.10"
//
// Run: cargo run -- sdcard.img

use std::env;
use std::fs::File;
use std::io::{self, BufReader, Read};
use sha2::{Sha256, Digest};

fn sha256_of_file(path: &str) -> io::Result<String> {
    let f   = File::open(path)?;
    let mut reader = BufReader::with_capacity(1 << 20, f);
    let mut hasher = Sha256::new();
    let mut buf    = [0u8; 65536];

    loop {
        let n = reader.read(&mut buf)?;
        if n == 0 { break; }
        hasher.update(&buf[..n]);
    }

    Ok(format!("{:x}", hasher.finalize()))
}

fn main() -> io::Result<()> {
    let img_path = env::args().nth(1).expect("Usage: img_verify <image>");
    let sha_path = format!("{}.sha256", img_path);

    println!("Computing SHA-256 of {img_path}...");
    let computed = sha256_of_file(&img_path)?;
    println!("Computed : {computed}");

    // Read stored hash (format: "<hash>  <filename>")
    let stored_raw = std::fs::read_to_string(&sha_path)
        .map_err(|e| {
            eprintln!("Cannot read {sha_path}: {e}");
            e
        })?;
    let stored = stored_raw.split_whitespace().next().unwrap_or("").to_string();
    println!("Stored   : {stored}");

    if computed == stored {
        println!("VERIFIED OK");
    } else {
        eprintln!("MISMATCH — image may be corrupt");
        std::process::exit(1);
    }
    Ok(())
}
```

---

## 11. ASCII Partition Diagrams

### 11.1 Typical BeagleBone Black SD Card (MBR)

```
 sdcard.img  (792 MiB total)
 ===================================================================

  Byte 0                                                     End
  |                                                            |
  +----------+-------+-------------------+---------------------+
  |   SPL    | Boot  |      rootfs       |        data         |
  |  (MLO)   | VFAT  |      ext4         |        ext4         |
  |  128 KiB | 16 MiB|     512 MiB       |       128 MiB       |
  +----------+-------+-------------------+---------------------+
  ^          ^       ^                   ^
  128 KiB    1 MiB   17 MiB             529 MiB

  Note: SPL written raw (not in partition table)
        boot partition: MBR type 0x0C (FAT32 LBA), bootable
        rootfs/data:    MBR type 0x83 (Linux)
```

### 11.2 Raspberry Pi 4 SD Card (MBR)

```
 sdcard.img  (512 MiB total)
 ===================================================================

  +---+------------------+--------------------------------------+
  |   |   boot (VFAT)    |           rootfs (ext4)              |
  |   |   64 MiB         |           448 MiB                    |
  |   |  start2.elf      |                                      |
  |   |  kernel8.img     |  /bin  /etc  /lib  /usr  ...         |
  |   |  config.txt      |                                      |
  +---+------------------+--------------------------------------+
    ^  ^                  ^
    |  1 MiB              65 MiB
    |
    MBR (512 bytes at offset 0)

  MBR Partition Table:
  +---------+---------+----------+-------+-------+--------+
  | Entry   | Status  |  Type    | LBA   | LBA   |  Size  |
  |         |         |          | Start | Count |        |
  +---------+---------+----------+-------+-------+--------+
  | 1 boot  | 0x80(*) | 0x0C FAT | 2048  |131072 | 64 MiB |
  | 2 root  | 0x00    | 0x83 ext | 133120|917504 |448 MiB |
  | 3       |  ---    |  ---     |  ---  |  ---  |  ---   |
  | 4       |  ---    |  ---     |  ---  |  ---  |  ---   |
  +---------+---------+----------+-------+-------+--------+
  (*) bootable flag set
```

### 11.3 GPT Disk Image (Industrial SBC)

```
 firmware.img  (GPT, 2 GiB)
 ===================================================================

  LBA 0     LBA 1       LBA 2..33    LBA 34                 LBA N
  +---------+-----------+------------+----+-------+-------+-+---------+
  | Prot.   | GPT Hdr   | Partition  |SPL | u-boot|  boot |r|Bk.      |
  | MBR     | 512 B     | Array      |512K|  1 MiB| 64 MiB|o|GPT      |
  |         | EFI PART  | 128 entries|    |       |       |o|Hdr      |
  |         |           |            |    |       |       |t|         |
  +---------+-----------+------------+----+-------+-------+-+----+----+
                                                            |f   |d   |
                                                            |s   |a   |
                                                            |    |t   |
                                                            |    |a   |
                                                            |512M|128M|
  GPT Partition Type GUIDs:
  +------------------------------------------+-------------------------------+
  | Partition  | GUID                                                        |
  +------------------------------------------+-------------------------------+
  | SPL (raw)  | 8DA63339-0007-60C0-C436-083AC8230908                        |
  | u-boot     | 8DA63339-0007-60C0-C436-083AC8230908                        |
  | boot (EFI) | C12A7328-F81F-11D2-BA4B-00A0C93EC93B                        |
  | rootfs     | 0FC63DAF-8483-4772-8E79-3D69D8477DE4 (Linux filesystem)     |
  | data       | 0FC63DAF-8483-4772-8E79-3D69D8477DE4                        |
  +------------------------------------------+-------------------------------+
```

### 11.4 genimage Processing Flow

```
  Input files (output/images/)
  ============================

  zImage  bcm*.dtb  u-boot.bin  rootfs.ext4
      \       |         |            /
       \      |         |           /
        v     v         v          v
  +------------------------------------+
  |         genimage                   |
  |  reads: genimage.cfg               |
  |                                    |
  |  1. Create vfat for boot/          |
  |     mcopy zImage, dtb, u-boot.bin  |
  |  2. Reference rootfs.ext4          |
  |  3. Allocate MBR / GPT table       |
  |  4. Write partition data at        |
  |     correct LBA offsets            |
  |  5. Pad to declared image size     |
  +------------------------------------+
                  |
                  v
         sdcard.img  (binary, ready for dd)
         firmware.wic (same, .bmap optional)

  Flash commands:
  +-----------------------------------------+
  |  dd if=sdcard.img of=/dev/sdX bs=1M     |
  |  bmaptool copy firmware.wic /dev/mmcblk0|
  |  udisksctl loop-setup -f sdcard.img     |  (desktop inspection)
  +-----------------------------------------+
```

---

## 12. Advanced Topics

### 12.1 Reproducible Builds with Fixed UUIDs

By default, genimage generates random GPT disk and partition GUIDs.
For reproducible builds, pin them in `genimage.cfg`:

```cfg
image sdcard.img {
    hdimage {
        gpt = true
        disk-uuid = "12345678-1234-1234-1234-123456789ABC"
    }

    partition rootfs {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"
        partition-uuid       = "ABCDEF01-1234-5678-ABCD-0123456789AB"
        image = "rootfs.ext4"
    }
}
```

### 12.2 Using genimage with RAUC (OTA Updates)

RAUC uses a dual-partition A/B scheme. Genimage can pre-populate both slots:

```cfg
image slot_a.ext4 { ext4 { label = "rootfs_a" } size = 512M }
image slot_b.ext4 { ext4 { label = "rootfs_b" } size = 512M }

image firmware.img {
    hdimage { gpt = true }

    partition boot {
        partition-type-uuid = "C12A7328-F81F-11D2-BA4B-00A0C93EC93B"
        image = "boot.vfat"
        size  = 64M
        offset = 2M
    }

    partition rootfs_a {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"
        image  = "slot_a.ext4"
        offset = 66M
    }

    partition rootfs_b {
        partition-type-uuid = "0FC63DAF-8483-4772-8E79-3D69D8477DE4"
        image  = "slot_b.ext4"
        # placed after rootfs_a automatically
    }
}
```

### 12.3 NOR Flash / NAND Images

For raw NOR flash (no partition table):

```cfg
image nor-flash.img {
    # No hdimage block → raw binary concatenation

    # Bootloader at offset 0
    partition spl {
        in-partition-table = false
        image  = "u-boot-spl.bin"
        offset = 0
        size   = 64K
    }

    partition uboot {
        in-partition-table = false
        image  = "u-boot.bin"
        offset = 64K
        size   = 448K
    }

    partition env {
        in-partition-table = false
        image  = "uboot-env.bin"
        offset = 512K
        size   = 64K
    }

    partition kernel {
        in-partition-table = false
        image  = "zImage"
        offset = 1M
        size   = 6M
    }

    partition rootfs {
        in-partition-table = false
        image  = "rootfs.squashfs"
        offset = 7M
    }
}
```

### 12.4 Integrating genimage with Buildroot's Config System

In `board/myboard/Config.in`:

```kconfig
config BR2_PACKAGE_MYBOARD
    bool "myboard support"
    select BR2_PACKAGE_HOST_GENIMAGE
    select BR2_PACKAGE_HOST_DOSFSTOOLS
    select BR2_PACKAGE_HOST_E2FSPROGS
```

In `board/myboard/myboard.mk`:

```makefile
define MYBOARD_BUILD_IMAGE
    $(HOST_DIR)/bin/genimage \
        --config  $(BR2_EXTERNAL_MYBOARD_PATH)/board/myboard/genimage.cfg \
        --rootpath $(TARGET_DIR) \
        --tmppath  $(BUILD_DIR)/genimage.tmp \
        --inputpath $(BINARIES_DIR) \
        --outputpath $(BINARIES_DIR)
endef

MYBOARD_POST_IMAGE_HOOKS += MYBOARD_BUILD_IMAGE
```

---

## 13. Summary

| Aspect | Detail |
|--------|--------|
| **Tool** | `genimage` — declarative disk image assembler |
| **Config file** | `genimage.cfg` — libconfig syntax |
| **Partition tables** | MBR (up to 4 primary) and GPT (up to 128) |
| **Supported FS** | vfat, ext2/3/4, squashfs, jffs2, ubifs, raw |
| **Output formats** | `.img`, `.wic`, raw binaries |
| **Buildroot hook** | `post-image.sh` calling `genimage` |
| **Flash methods** | `dd`, `bmaptool`, `etcher`, `dfu-util` |
| **Key strength** | Reproducible, declarative, offset-exact images |

### Core Workflow Recap

```
  BR2_ROOTFS_POST_IMAGE_SCRIPT=board/myboard/post-image.sh
                     |
                     v
  post-image.sh  -->  genimage  -->  sdcard.img
                          ^
                          |
                     genimage.cfg
                    (partitions, offsets,
                     fs types, GPT/MBR)
```

**genimage** is the production-grade solution for assembling final firmware images
in Buildroot-based embedded Linux projects. By combining a declarative `genimage.cfg`
with a `post-image.sh` orchestration script, developers gain full control over partition
layout, alignment, file-system content, and bootloader placement — all in a reproducible,
build-system-integrated pipeline.

---

*Buildroot Series — Topic 17 | Partition Layout & Image Generation (genimage)*