# 15. Root Filesystem Configuration & Skeleton

**Structure covered:**

- **Skeleton** (`BR2_ROOTFS_SKELETON_CUSTOM`) — how it replaces the default BusyBox layout, what to put in it, merge-usr layouts, and ordering gotchas.
- **Device/Permissions Table** (`BR2_ROOTFS_DEVICE_TABLE`) — full syntax table, an annotated example with device nodes, ttyS range expansion, and restricted file modes, plus how to register multiple table files.
- **User Tables** (`BR2_ROOTFS_USERS_TABLES`) — field-by-field breakdown, a realistic example with service/log/dev accounts, and how UIDs must align with the device table.
- **Post-Build Hooks** — the complete execution order timeline, a full `post-build.sh` (metadata injection, stripping, validation), `post-image.sh` (signing), and per-package hook points.

**Code examples:**

| Language | What it demonstrates |
|----------|----------------------|
| **C** | Parsing `/etc/build-info` via syslog; dropping to `myapp` uid with `setuid`/`initgroups` and opening a Unix socket |
| **C++** | Host-side rootfs manifest validator (checks existence + octal modes); ASCII directory tree walker with human-readable file sizes |
| **Rust** | Cross-compile `.mk` scaffold; daemon dropping privileges with `nix` crate + Unix socket; host tool generating device tables from TOML |

The ASCII diagrams illustrate the overall build pipeline, the before/after of using a permissions table, and the complete execution order from skeleton through post-image scripts.


> **Buildroot Series — Topic 15**  
> Covers: `BR2_ROOTFS_SKELETON_CUSTOM`, permissions tables (`BR2_ROOTFS_DEVICE_TABLE`),
> user tables, and post-build hooks — with C, C++, and Rust code examples.

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Root Filesystem Skeleton](#2-the-root-filesystem-skeleton)
3. [BR2_ROOTFS_SKELETON_CUSTOM](#3-br2_rootfs_skeleton_custom)
4. [Permissions & Device Tables (BR2_ROOTFS_DEVICE_TABLE)](#4-permissions--device-tables-br2_rootfs_device_table)
5. [User Tables](#5-user-tables)
6. [Post-Build Hooks](#6-post-build-hooks)
7. [Code Examples — C](#7-code-examples--c)
8. [Code Examples — C++](#8-code-examples--c)
9. [Code Examples — Rust](#9-code-examples--rust)
10. [Summary](#10-summary)

---

## 1. Overview

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                  Buildroot Build Pipeline                       │
  │                                                                 │
  │  menuconfig / .config                                           │
  │       │                                                         │
  │       ▼                                                         │
  │  ┌─────────────┐    ┌──────────────────┐    ┌───────────────┐   │
  │  │  Toolchain  │───▶│  Package Build   │───▶│  Target Dir   │   │
  │  └─────────────┘    └──────────────────┘    │  (output/     │   │
  │                                             │   target/)    │   │
  │  ┌──────────────────────────────────────┐   └──────┬────────┘   │
  │  │  Root FS Configuration               │          │            │
  │  │  ┌────────────────────────────────┐  │          ▼            │
  │  │  │ BR2_ROOTFS_SKELETON_CUSTOM     │──┼──▶  Skeleton Copy     │
  │  │  ├────────────────────────────────┤  │          │            │
  │  │  │ BR2_ROOTFS_DEVICE_TABLE        │──┼──▶  Permissions Set   │
  │  │  ├────────────────────────────────┤  │          │            │
  │  │  │ BR2_ROOTFS_USERS_TABLES        │──┼──▶  Users Created     │
  │  │  ├────────────────────────────────┤  │          │            │
  │  │  │ BR2_ROOTFS_POST_BUILD_SCRIPT   │──┼──▶  Hook Scripts      │
  │  │  └────────────────────────────────┘  │          │            │
  │  └──────────────────────────────────────┘          ▼            │
  │                                             Final Root FS Image │
  └─────────────────────────────────────────────────────────────────┘
```

When Buildroot assembles the final root filesystem it follows a well-defined
sequence. Understanding each stage lets you:

- Ship a minimal, purpose-built root filesystem without the Buildroot defaults.
- Enforce exact file ownership and permissions across every build.
- Add system users that packages rely on without patching upstream makefiles.
- Inject environment-specific files (certificates, config files, init scripts)
  via post-build hooks without touching package recipes.

The four pillars of this stage are examined in detail below.

---

## 2. The Root Filesystem Skeleton

Before any package installs a single file, Buildroot copies a **skeleton** into
`output/target/`. The skeleton provides the fundamental directory hierarchy
(`/bin`, `/etc`, `/lib`, `/proc`, `/sys`, …) plus mandatory files such as
`/etc/inittab`, `/etc/passwd`, and `/etc/group`.

```
  Default Skeleton Location
  ─────────────────────────
  buildroot/
  └── system/
      └── skeleton/           ← BR2_ROOTFS_SKELETON_DEFAULT
          ├── bin/
          ├── dev/
          ├── etc/
          │   ├── group
          │   ├── hostname
          │   ├── inittab
          │   ├── passwd
          │   ├── profile
          │   └── shadow
          ├── lib/
          ├── media/
          ├── mnt/
          ├── opt/
          ├── proc/
          ├── root/
          ├── run/
          ├── sbin/
          ├── sys/
          ├── tmp/
          ├── usr/
          └── var/
```

Buildroot offers three skeleton modes, selectable in `menuconfig` under
**System configuration → Root filesystem skeleton**:

| Mode | Config Symbol | Description |
|------|--------------|-------------|
| Default (BusyBox) | `BR2_ROOTFS_SKELETON_DEFAULT` | Bundled skeleton, BusyBox-centric |
| Custom | `BR2_ROOTFS_SKELETON_CUSTOM` | Your own skeleton directory |
| None | `BR2_ROOTFS_SKELETON_NONE` | Empty — advanced use only |

---

## 3. BR2_ROOTFS_SKELETON_CUSTOM

### 3.1 Purpose

`BR2_ROOTFS_SKELETON_CUSTOM` points to a directory that **replaces** (not
extends) the default skeleton. Every file and subdirectory in that path is
rsync-copied to `output/target/` before any package installation begins.

Use this when you need:

- A non-BusyBox init system (systemd, s6, runit) with its own directory layout.
- Company-wide default `/etc` files (NTP config, CA certificates, hostname
  templates) baked into every product image.
- A stripped-down hierarchy for a read-only squashfs partition.

### 3.2 Enabling in menuconfig

```
  System configuration
    └─> Root filesystem skeleton (Custom skeleton)
          └─> custom skeleton path:  $(BR2_EXTERNAL_MY_PATH)/board/myboard/skeleton
```

Or directly in `<buildroot>/.config` / a `defconfig` fragment:

```makefile
BR2_ROOTFS_SKELETON_CUSTOM=y
BR2_ROOTFS_SKELETON_CUSTOM_PATH="$(BR2_EXTERNAL)/board/myboard/skeleton"
```

### 3.3 Skeleton Directory Layout Example

```
  my-skeleton/
  ├── etc/
  │   ├── fstab           ← Custom mount table
  │   ├── group           ← Baseline groups
  │   ├── hostname        ← Default hostname
  │   ├── inittab         ← BusyBox init config
  │   ├── network/
  │   │   └── interfaces  ← Static IP defaults
  │   ├── passwd          ← System accounts
  │   ├── profile         ← Shell environment
  │   ├── rc.local        ← Late-boot hook
  │   └── shadow          ← Hashed passwords (mode 0640)
  ├── lib -> usr/lib      ← Merged usr layout symlink
  ├── sbin -> usr/sbin
  └── usr/
      ├── bin/
      ├── lib/
      └── sbin/
```

### 3.4 Gotchas

- The skeleton is copied **before** packages; packages may overwrite files you
  place there.
- If using a merged-usr layout (`/lib → /usr/lib`), make sure your toolchain
  and all packages also support it.
- Buildroot does **not** validate the skeleton — a missing `/proc` will produce
  a broken image silently.
- The skeleton directory itself is **not** cleaned on `make clean`; changes
  take effect on the next build automatically.

---

## 4. Permissions & Device Tables (BR2_ROOTFS_DEVICE_TABLE)

### 4.1 Why Permissions Tables Exist

Linux filesystems encode ownership (`uid`/`gid`) and permission bits in inode
metadata. When Buildroot copies files into `output/target/` it runs as a
regular user, so it **cannot** `chown` to `root:root` or create device nodes
(`/dev/null`, `/dev/ttyS0`, …) at copy-time without elevated privileges.

The device/permissions table defers these operations to the image-generation
phase, where Buildroot's `makedevs` tool applies them without ever needing
`sudo`.

```
  ┌──────────────────────────────────────────────────────────┐
  │  Without device table    │  With device table            │
  ├──────────────────────────┼───────────────────────────────┤
  │  /etc/shadow  -rw-r--r-- │  /etc/shadow  -r--------      │
  │  (world-readable!)       │  root:shadow  640             │
  │                          │                               │
  │  /dev/* — missing or     │  /dev/null    c 666 0 0 1 3   │
  │  require sudo mknod      │  /dev/ttyS0   c 660 0 5 4 64  │
  └──────────────────────────┴───────────────────────────────┘
```

### 4.2 Table Syntax

Each non-comment line uses this whitespace-separated format:

```
<path>  <type>  <mode>  <uid>  <gid>  [<major>  <minor>  <start>  <inc>  <count>]
```

| Field | Meaning |
|-------|---------|
| `path` | Absolute path inside the target rootfs |
| `type` | `f` file, `d` directory, `c` char device, `b` block device, `p` pipe, `l` symlink |
| `mode` | Octal permission bits |
| `uid` | Numeric user ID |
| `gid` | Numeric group ID |
| `major`/`minor` | Device numbers (devices only) |
| `start`/`inc`/`count` | Range expansion: creates `path0 … path<count-1>` |

### 4.3 Example Device Table

```
# /board/myboard/device_table.txt
#
# path                type  mode  uid   gid   major minor start inc count
#---------------------------------------------------------------------------

# Core devices
/dev                  d     755   0     0     -     -     -     -   -
/dev/null             c     666   0     0     1     3     -     -   -
/dev/zero             c     666   0     0     1     5     -     -   -
/dev/full             c     666   0     0     1     7     -     -   -
/dev/random           c     444   0     0     1     8     -     -   -
/dev/urandom          c     444   0     0     1     9     -     -   -
/dev/console          c     600   0     0     5     1     -     -   -
/dev/tty              c     666   0     5     5     0     -     -   -

# Serial ports ttyS0 … ttyS3  (start=0, inc=1, count=4)
/dev/ttyS             c     660   0     5     4     64    0     1   4

# Restrict sensitive files
/etc/shadow           f     640   0     42    -     -     -     -   -
/etc/gshadow          f     640   0     42    -     -     -     -   -
/etc/sudoers          f     440   0     0     -     -     -     -   -

# Application runtime directories
/var/run/myapp        d     750   1000  1000  -     -     -     -   -
/var/log/myapp        d     750   1000  1000  -     -     -     -   -
```

### 4.4 Registering in Buildroot

```makefile
# In your .config / defconfig fragment:
BR2_ROOTFS_DEVICE_TABLE="board/myboard/device_table.txt"

# Multiple tables (space-separated):
BR2_ROOTFS_DEVICE_TABLE="system/device_table.txt board/myboard/extra_devices.txt"
```

### 4.5 Package-Level Device Tables

Individual packages can ship their own device table fragment:

```makefile
# In package/mypackage/mypackage.mk
define MYPACKAGE_INSTALL_DEVICE_TABLE
    $(INSTALL) -m 0644 $(MYPACKAGE_PKGDIR)/mypackage_devices.txt \
        $(TARGET_DIR)/../images/  # Not correct — use the mechanism below
endef

# Correct mechanism: install into $(BR2_ROOTFS_DEVICE_TABLE) via a .mk variable
MYPACKAGE_DEVICES = /dev/mydev c 660 0 0 10 200 - - -
```

---

## 5. User Tables

### 5.1 Purpose

User tables let you declare system accounts in a declarative, repeatable way.
Buildroot's `makeusers` tool processes them during `make` and creates the
accounts in `output/target/etc/passwd`, `shadow`, and `group` — without
requiring the host to have `useradd` at the right uid namespace.

### 5.2 Table Syntax

```
<username>  <uid>  <group>  <gid>  <password>  <home>  <shell>  <groups>  <comment>
```

| Field | Meaning |
|-------|---------|
| `username` | Login name |
| `uid` | Numeric UID (`-1` = auto-assign) |
| `group` | Primary group name |
| `gid` | Numeric GID (`-1` = auto-assign) |
| `password` | `!` = locked, `=` = empty, or a crypt hash |
| `home` | Home directory path |
| `shell` | Login shell (`/sbin/nologin` for system accounts) |
| `groups` | Comma-separated list of supplementary groups |
| `comment` | GECOS field |

### 5.3 Example User Table

```
# /board/myboard/users_table.txt
#
#  name     uid   group    gid   pw    home              shell             groups         comment
#----------------------------------------------------------------------------------------------------

# Service account for the main application daemon
myapp       1000  myapp    1000  !     /var/lib/myapp    /sbin/nologin     gpio,i2c       MyApp daemon

# Log reader account (read-only access to /var/log)
logview     1001  logview  1001  !     /dev/null         /sbin/nologin     -              Log viewer

# Development-only debug account (disable in production defconfig)
devuser     1002  devuser  1002  =     /home/devuser     /bin/sh           myapp,logview  Developer
```

### 5.4 Registering in Buildroot

```makefile
BR2_ROOTFS_USERS_TABLES="board/myboard/users_table.txt"

# Multiple files:
BR2_ROOTFS_USERS_TABLES="board/common/base_users.txt board/myboard/extra_users.txt"
```

### 5.5 Interaction Between User Tables and Device Tables

When a service account owns files (`uid=1000`), you must reference the same
numeric UID in the device table:

```
  User Table Entry:
    myapp  1000  myapp  1000 ...

  Device Table Entry:
    /var/lib/myapp   d  750  1000  1000  -  -  -  -  -
    /var/run/myapp   d  750  1000  1000  -  -  -  -  -
    /etc/myapp.conf  f  640  1000  1000  -  -  -  -  -
```

---

## 6. Post-Build Hooks

### 6.1 Overview

Post-build hooks are shell scripts (or make fragments) that Buildroot executes
**after** all packages are installed into `output/target/` but **before** the
filesystem image is created. They give you a final chance to:

- Inject build-time information (git hash, build date, version strings).
- Patch configuration files that packages install with wrong defaults.
- Remove unwanted files (documentation, test binaries) to reduce image size.
- Validate the image — abort the build if a mandatory file is missing.

```
  Build Stage Timeline
  ─────────────────────────────────────────────────────────────────────
  [skeleton copy] → [package installs] → [POST-BUILD HOOK] → [image gen]
                                               ▲
                                      Your scripts run here.
                                      output/target/ is fully
                                      populated but no image yet.
  ─────────────────────────────────────────────────────────────────────
```

### 6.2 Configuring Post-Build Scripts

```makefile
# .config / defconfig
BR2_ROOTFS_POST_BUILD_SCRIPT="board/myboard/post-build.sh"

# Multiple scripts (space-separated, executed in order):
BR2_ROOTFS_POST_BUILD_SCRIPT="board/common/post-build.sh board/myboard/post-build.sh"
```

Scripts receive two arguments automatically:

- `$1` — path to `output/target/`
- `$2` — path to `output/images/`

### 6.3 Example Post-Build Script

```bash
#!/usr/bin/env bash
# board/myboard/post-build.sh
# Invoked by Buildroot after all packages are installed.
# $1 = TARGET_DIR   $2 = IMAGES_DIR
set -euo pipefail

TARGET_DIR="$1"
IMAGES_DIR="$2"

# ── 1. Embed build metadata ──────────────────────────────────────────
BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
GIT_HASH=$(git -C "$(dirname "$0")/../.." describe --always --dirty 2>/dev/null || echo "unknown")

cat > "${TARGET_DIR}/etc/build-info" <<EOF
BUILD_DATE=${BUILD_DATE}
GIT_HASH=${GIT_HASH}
BUILDROOT_VERSION=$(cat "${BR2_CONFIG%/.config}/Makefile" | grep "^BR2_VERSION := " | cut -d' ' -f3)
EOF
chmod 0444 "${TARGET_DIR}/etc/build-info"

# ── 2. Inject the correct hostname ───────────────────────────────────
HOSTNAME="${BR2_TARGET_GENERIC_HOSTNAME:-myboard}"
echo "${HOSTNAME}" > "${TARGET_DIR}/etc/hostname"

# ── 3. Strip development artefacts (reduces squashfs by ~4 MB) ───────
find "${TARGET_DIR}/usr/share" -name "*.pod" -delete
find "${TARGET_DIR}/usr/lib"   -name "*.la"  -delete
rm -rf "${TARGET_DIR}/usr/share/man"
rm -rf "${TARGET_DIR}/usr/share/doc"
rm -rf "${TARGET_DIR}/usr/include"

# ── 4. Validate mandatory runtime files ──────────────────────────────
MANDATORY=(
    "/etc/passwd"
    "/etc/group"
    "/usr/bin/myapp"
    "/etc/myapp/myapp.conf"
)

for f in "${MANDATORY[@]}"; do
    if [[ ! -e "${TARGET_DIR}${f}" ]]; then
        echo "ERROR: Mandatory file missing from rootfs: ${f}" >&2
        exit 1
    fi
done

echo "[post-build] Done. Target: ${TARGET_DIR}"
```

### 6.4 Post-Image Hooks

A related hook — `BR2_ROOTFS_POST_IMAGE_SCRIPT` — runs after image files have
been generated. Use it to sign images, generate manifests, or copy artefacts to
a deployment server:

```makefile
BR2_ROOTFS_POST_IMAGE_SCRIPT="board/myboard/post-image.sh"
```

```bash
#!/usr/bin/env bash
# board/myboard/post-image.sh
# $1 = IMAGES_DIR
set -euo pipefail

IMAGES_DIR="$1"

# Sign the squashfs image with our private key
openssl dgst -sha256 -sign /secrets/sign.key \
    -out "${IMAGES_DIR}/rootfs.squashfs.sig" \
    "${IMAGES_DIR}/rootfs.squashfs"

echo "[post-image] Signed rootfs.squashfs"
```

### 6.5 Package-Level Hooks

Inside a `.mk` recipe you can add hooks at multiple points:

```makefile
define MYPACKAGE_POST_BUILD_HOOKS_APPLY
    # Runs right after MYPACKAGE installs into $(TARGET_DIR)
    $(INSTALL) -m 0755 $(MYPACKAGE_PKGDIR)/extra-init \
        $(TARGET_DIR)/etc/init.d/S99mypackage
endef
MYPACKAGE_POST_BUILD_HOOKS += MYPACKAGE_POST_BUILD_HOOKS_APPLY
```

Available hook points per package:

```
  MYPACKAGE_POST_PATCH_HOOKS
  MYPACKAGE_POST_CONFIGURE_HOOKS
  MYPACKAGE_POST_BUILD_HOOKS
  MYPACKAGE_POST_INSTALL_STAGING_HOOKS
  MYPACKAGE_POST_INSTALL_TARGET_HOOKS
  MYPACKAGE_POST_INSTALL_HOOKS
```

---

## 7. Code Examples — C

### 7.1 Reading Build Metadata at Runtime

A C program that parses `/etc/build-info` (written by the post-build script)
and logs it via syslog. Useful as a boot-time banner in your init scripts.

```c
/* src/build_info.c
 * Compile (cross): ${CROSS_COMPILE}gcc -Wall -Wextra -o build_info build_info.c
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <syslog.h>
#include <errno.h>

#define BUILD_INFO_PATH "/etc/build-info"
#define MAX_LINE        256

typedef struct {
    char build_date[64];
    char git_hash[64];
    char br_version[32];
} BuildInfo;

static int parse_build_info(BuildInfo *bi)
{
    FILE *f = fopen(BUILD_INFO_PATH, "r");
    if (!f) {
        syslog(LOG_WARNING, "Cannot open %s: %s", BUILD_INFO_PATH, strerror(errno));
        return -1;
    }

    char line[MAX_LINE];
    while (fgets(line, sizeof(line), f)) {
        /* Strip trailing newline */
        line[strcspn(line, "\n")] = '\0';

        char *eq = strchr(line, '=');
        if (!eq) continue;
        *eq = '\0';
        const char *key   = line;
        const char *value = eq + 1;

        if (strcmp(key, "BUILD_DATE") == 0)
            snprintf(bi->build_date,  sizeof(bi->build_date),  "%s", value);
        else if (strcmp(key, "GIT_HASH") == 0)
            snprintf(bi->git_hash,    sizeof(bi->git_hash),    "%s", value);
        else if (strcmp(key, "BUILDROOT_VERSION") == 0)
            snprintf(bi->br_version,  sizeof(bi->br_version),  "%s", value);
    }

    fclose(f);
    return 0;
}

int main(void)
{
    openlog("build_info", LOG_PID | LOG_CONS, LOG_DAEMON);

    BuildInfo bi = {0};
    if (parse_build_info(&bi) == 0) {
        syslog(LOG_INFO, "Build date   : %s", bi.build_date);
        syslog(LOG_INFO, "Git hash     : %s", bi.git_hash);
        syslog(LOG_INFO, "Buildroot    : %s", bi.br_version);
        printf("Build date   : %s\n", bi.build_date);
        printf("Git hash     : %s\n", bi.git_hash);
        printf("Buildroot    : %s\n", bi.br_version);
    }

    closelog();
    return 0;
}
```

### 7.2 Daemon Running as a Restricted Service Account

This snippet shows how a daemon drops privileges to the `myapp` user (uid/gid
1000) declared in the user table, then opens a Unix domain socket in
`/var/run/myapp/` (owned 750 by that account per the device table).

```c
/* src/myapp_daemon.c */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pwd.h>
#include <grp.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <sys/stat.h>
#include <errno.h>
#include <string.h>
#include <syslog.h>

#define SOCKET_PATH "/var/run/myapp/myapp.sock"
#define SERVICE_USER "myapp"

static int drop_privileges(const char *username)
{
    struct passwd *pw = getpwnam(username);
    if (!pw) {
        syslog(LOG_ERR, "User '%s' not found in /etc/passwd", username);
        return -1;
    }

    /* Drop supplementary groups first */
    if (initgroups(username, pw->pw_gid) != 0) {
        syslog(LOG_ERR, "initgroups: %s", strerror(errno));
        return -1;
    }
    if (setgid(pw->pw_gid) != 0) {
        syslog(LOG_ERR, "setgid(%d): %s", pw->pw_gid, strerror(errno));
        return -1;
    }
    if (setuid(pw->pw_uid) != 0) {
        syslog(LOG_ERR, "setuid(%d): %s", pw->pw_uid, strerror(errno));
        return -1;
    }

    syslog(LOG_INFO, "Running as uid=%d gid=%d", getuid(), getgid());
    return 0;
}

static int create_unix_socket(void)
{
    int fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (fd < 0) {
        syslog(LOG_ERR, "socket: %s", strerror(errno));
        return -1;
    }

    unlink(SOCKET_PATH);  /* Remove stale socket */

    struct sockaddr_un addr = { .sun_family = AF_UNIX };
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    if (bind(fd, (struct sockaddr *)&addr, sizeof(addr)) != 0) {
        syslog(LOG_ERR, "bind(%s): %s", SOCKET_PATH, strerror(errno));
        close(fd);
        return -1;
    }

    chmod(SOCKET_PATH, 0660);  /* Group-writable for IPC clients */
    listen(fd, 8);
    return fd;
}

int main(void)
{
    openlog("myapp", LOG_PID | LOG_CONS, LOG_DAEMON);

    /* Drop root privileges before opening the socket */
    if (drop_privileges(SERVICE_USER) != 0) {
        syslog(LOG_CRIT, "Failed to drop privileges — aborting");
        return EXIT_FAILURE;
    }

    int server_fd = create_unix_socket();
    if (server_fd < 0) return EXIT_FAILURE;

    syslog(LOG_INFO, "myapp listening on %s", SOCKET_PATH);

    /* Main accept loop (simplified) */
    while (1) {
        int client = accept(server_fd, NULL, NULL);
        if (client < 0) {
            if (errno == EINTR) continue;
            syslog(LOG_ERR, "accept: %s", strerror(errno));
            break;
        }
        /* Handle client in a real implementation */
        close(client);
    }

    close(server_fd);
    closelog();
    return EXIT_SUCCESS;
}
```

---

## 8. Code Examples — C++

### 8.1 Post-Build Validator (Host Tool)

A C++ host-side tool that validates the target directory against a manifest
file. Integrate it into `BR2_ROOTFS_POST_BUILD_SCRIPT` to fail the build early
if required files are absent or have wrong permissions.

```cpp
// tools/rootfs_validator.cpp
// Build on host: g++ -std=c++17 -Wall -o rootfs_validator rootfs_validator.cpp
// Usage: ./rootfs_validator <target_dir> <manifest.txt>

#include <filesystem>
#include <fstream>
#include <iostream>
#include <sstream>
#include <string>
#include <vector>
#include <cstring>
#include <sys/stat.h>

namespace fs = std::filesystem;

struct ManifestEntry {
    std::string path;
    mode_t      required_mode;   // 0 = do not check
    bool        must_exist;
};

static std::vector<ManifestEntry> load_manifest(const std::string &manifest_path)
{
    std::vector<ManifestEntry> entries;
    std::ifstream f(manifest_path);
    if (!f) throw std::runtime_error("Cannot open manifest: " + manifest_path);

    std::string line;
    while (std::getline(f, line)) {
        if (line.empty() || line[0] == '#') continue;
        std::istringstream iss(line);
        ManifestEntry e{};
        std::string mode_str;
        iss >> e.path >> mode_str;
        e.must_exist    = true;
        e.required_mode = mode_str.empty() ? 0 : static_cast<mode_t>(
                              std::stoul(mode_str, nullptr, 8));
        entries.push_back(std::move(e));
    }
    return entries;
}

static bool validate_entry(const fs::path &target_dir, const ManifestEntry &e)
{
    fs::path full = target_dir / e.path.substr(1);  // strip leading '/'
    bool ok = true;

    if (!fs::exists(full)) {
        std::cerr << "[FAIL] Missing : " << e.path << "\n";
        return false;
    }

    if (e.required_mode != 0) {
        struct stat st{};
        if (::stat(full.c_str(), &st) != 0) {
            std::cerr << "[FAIL] stat()  : " << e.path
                      << " (" << std::strerror(errno) << ")\n";
            return false;
        }
        mode_t actual = st.st_mode & 07777;
        if (actual != e.required_mode) {
            std::cerr << "[FAIL] Mode    : " << e.path
                      << "  expected=" << std::oct << e.required_mode
                      << " actual="   << actual << std::dec << "\n";
            ok = false;
        } else {
            std::cout << "[ OK ] " << e.path
                      << " (mode=" << std::oct << actual << std::dec << ")\n";
        }
    } else {
        std::cout << "[ OK ] " << e.path << "\n";
    }

    return ok;
}

int main(int argc, char *argv[])
{
    if (argc != 3) {
        std::cerr << "Usage: " << argv[0] << " <target_dir> <manifest.txt>\n";
        return EXIT_FAILURE;
    }

    const fs::path target_dir = argv[1];
    const std::string manifest_path = argv[2];

    if (!fs::is_directory(target_dir)) {
        std::cerr << "Error: Not a directory: " << target_dir << "\n";
        return EXIT_FAILURE;
    }

    std::vector<ManifestEntry> entries;
    try {
        entries = load_manifest(manifest_path);
    } catch (const std::exception &ex) {
        std::cerr << "Error: " << ex.what() << "\n";
        return EXIT_FAILURE;
    }

    std::cout << "\nValidating rootfs: " << target_dir << "\n";
    std::cout << std::string(50, '-') << "\n";

    int failures = 0;
    for (const auto &e : entries)
        if (!validate_entry(target_dir, e)) ++failures;

    std::cout << std::string(50, '-') << "\n";
    std::cout << "Result: " << (entries.size() - failures) << "/"
              << entries.size() << " checks passed";
    if (failures > 0)
        std::cout << "  — " << failures << " FAILED\n";
    else
        std::cout << "  — all OK\n";

    return failures > 0 ? EXIT_FAILURE : EXIT_SUCCESS;
}
```

**Example manifest file** (`board/myboard/rootfs_manifest.txt`):

```
# path                     octal-mode
/etc/passwd                0644
/etc/shadow                0640
/etc/sudoers               0440
/usr/bin/myapp             0755
/var/lib/myapp             0750
/var/log/myapp             0750
/etc/build-info            0444
/etc/hostname
/etc/group
```

**Integration in post-build script:**

```bash
# In board/myboard/post-build.sh
"${BR2_EXTERNAL}/tools/rootfs_validator" \
    "${TARGET_DIR}" \
    "${BR2_EXTERNAL}/board/myboard/rootfs_manifest.txt" \
|| { echo "ERROR: rootfs validation failed" >&2; exit 1; }
```

### 8.2 ASCII Filesystem Visualizer (Host Tool)

A small C++ tool that walks `output/target/` and prints an ASCII tree with
file sizes — useful in CI log output.

```cpp
// tools/rootfs_tree.cpp
// Build: g++ -std=c++17 -Wall -o rootfs_tree rootfs_tree.cpp
// Usage: ./rootfs_tree output/target/ [max_depth]

#include <filesystem>
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <sys/stat.h>

namespace fs = std::filesystem;

static const std::vector<std::string> SKIP_DIRS = {"proc", "sys", "dev"};

static std::string human_size(uintmax_t bytes)
{
    if (bytes < 1024)        return std::to_string(bytes) + "B";
    if (bytes < 1024*1024)   return std::to_string(bytes/1024) + "K";
    return std::to_string(bytes/(1024*1024)) + "M";
}

static void walk(const fs::path &dir,
                 const std::string &prefix,
                 int depth, int max_depth)
{
    if (depth > max_depth) return;

    std::vector<fs::directory_entry> entries;
    for (const auto &e : fs::directory_iterator(dir,
             fs::directory_options::skip_permission_denied))
        entries.push_back(e);

    std::sort(entries.begin(), entries.end(),
              [](const auto &a, const auto &b){ return a.path() < b.path(); });

    for (std::size_t i = 0; i < entries.size(); ++i) {
        const auto &e   = entries[i];
        bool last        = (i == entries.size() - 1);
        const char *tee  = last ? "└── " : "├── ";
        const char *cont = last ? "    " : "│   ";

        std::string name = e.path().filename().string();

        /* Skip virtual filesystems */
        if (e.is_directory()) {
            bool skip = false;
            for (const auto &s : SKIP_DIRS)
                if (name == s) { skip = true; break; }
            if (skip) {
                std::cout << prefix << tee << name << "/  [skipped]\n";
                continue;
            }
            std::cout << prefix << tee << name << "/\n";
            walk(e.path(), prefix + cont, depth + 1, max_depth);
        } else if (e.is_symlink()) {
            auto target = fs::read_symlink(e.path());
            std::cout << prefix << tee << name << " -> " << target.string() << "\n";
        } else if (e.is_regular_file()) {
            uintmax_t sz = e.file_size();
            std::cout << prefix << tee << name
                      << "  [" << human_size(sz) << "]\n";
        }
    }
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <target_dir> [max_depth=3]\n";
        return EXIT_FAILURE;
    }

    fs::path root = argv[1];
    int max_depth  = (argc >= 3) ? std::stoi(argv[2]) : 3;

    if (!fs::is_directory(root)) {
        std::cerr << "Not a directory: " << root << "\n";
        return EXIT_FAILURE;
    }

    std::cout << root.string() << "/\n";
    walk(root, "", 0, max_depth);
    return EXIT_SUCCESS;
}
```

**Sample output:**

```
output/target/
├── bin/
│   ├── busybox  [1M]
│   └── sh -> busybox
├── dev/  [skipped]
├── etc/
│   ├── build-info  [128B]
│   ├── group  [312B]
│   ├── hostname  [8B]
│   ├── inittab  [1K]
│   ├── passwd  [892B]
│   └── shadow  [640B]
├── lib/
│   ├── libc.so.6  [1M]
│   └── libm.so.6  [512K]
├── proc/  [skipped]
├── sys/   [skipped]
├── usr/
│   ├── bin/
│   │   └── myapp  [256K]
│   └── lib/
└── var/
    ├── lib/
    │   └── myapp/
    ├── log/
    │   └── myapp/
    └── run/
```

---

## 9. Code Examples — Rust

### 9.1 Cross-Compiling a Rust Binary for Buildroot Targets

First, configure Buildroot to call `cargo` via a custom package:

```makefile
# package/myapp-rs/myapp-rs.mk
MYAPP_RS_VERSION  = 1.0.0
MYAPP_RS_SITE     = $(BR2_EXTERNAL)/package/myapp-rs/src
MYAPP_RS_SITE_METHOD = local
MYAPP_RS_LICENSE  = MIT

MYAPP_RS_CARGO_ENV = \
    CARGO_HOME=$(HOST_DIR)/share/cargo \
    CC=$(TARGET_CC) \
    CXX=$(TARGET_CXX) \
    AR=$(TARGET_AR)

define MYAPP_RS_BUILD_CMDS
    $(MYAPP_RS_CARGO_ENV) \
    $(HOST_DIR)/bin/cargo build \
        --release \
        --target $(RUSTC_TARGET_NAME) \
        --manifest-path $(@D)/Cargo.toml
endef

define MYAPP_RS_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 \
        $(@D)/target/$(RUSTC_TARGET_NAME)/release/myapp-rs \
        $(TARGET_DIR)/usr/bin/myapp-rs
endef

$(eval $(generic-package))
```

### 9.2 Rust Daemon Reading Build Info and Dropping Privileges

```rust
// src/main.rs  (myapp-rs)
// Cargo.toml dependencies:
//   nix  = { version = "0.27", features = ["user", "process"] }
//   log  = "0.4"
//   env_logger = "0.10"

use std::collections::HashMap;
use std::fs;
use std::os::unix::net::UnixListener;
use std::path::Path;

use nix::unistd::{Gid, Uid, User, initgroups, setgid, setuid};
use nix::sys::stat::{fchmodat, Mode, FchmodatFlags};
use nix::fcntl::OFlag;

const BUILD_INFO_PATH: &str = "/etc/build-info";
const SOCKET_PATH:     &str = "/var/run/myapp/myapp-rs.sock";
const SERVICE_USER:    &str = "myapp";

/// Parse a KEY=VALUE file into a HashMap.
fn parse_key_value(path: &str) -> HashMap<String, String> {
    let content = fs::read_to_string(path).unwrap_or_default();
    content
        .lines()
        .filter(|l| !l.trim_start().starts_with('#') && l.contains('='))
        .filter_map(|l| {
            let mut parts = l.splitn(2, '=');
            Some((parts.next()?.to_string(), parts.next()?.to_string()))
        })
        .collect()
}

/// Drop privileges to the given username (must exist in /etc/passwd).
fn drop_privileges(username: &str) -> Result<(), Box<dyn std::error::Error>> {
    let user = User::from_name(username)?
        .ok_or_else(|| format!("User '{}' not found in /etc/passwd", username))?;

    initgroups(
        &std::ffi::CString::new(username)?,
        user.gid,
    )?;
    setgid(user.gid)?;
    setuid(user.uid)?;

    log::info!(
        "Dropped privileges → uid={} gid={}",
        Uid::current(),
        Gid::current()
    );
    Ok(())
}

fn create_unix_socket(path: &str) -> Result<UnixListener, Box<dyn std::error::Error>> {
    let socket_path = Path::new(path);
    if socket_path.exists() {
        fs::remove_file(socket_path)?;
    }

    let listener = UnixListener::bind(socket_path)?;

    // chmod 0660 — group-writable so IPC clients in the 'myapp' group can connect
    fchmodat(
        None,
        socket_path,
        Mode::S_IRUSR | Mode::S_IWUSR | Mode::S_IRGRP | Mode::S_IWGRP,
        FchmodatFlags::NoFollowSymlink,
    )?;

    Ok(listener)
}

fn main() {
    env_logger::Builder::new()
        .filter_level(log::LevelFilter::Info)
        .init();

    // ── 1. Print build metadata ──────────────────────────────────────
    let info = parse_key_value(BUILD_INFO_PATH);
    log::info!(
        "Starting myapp-rs  build={} git={}",
        info.get("BUILD_DATE").map_or("?", |s| s),
        info.get("GIT_HASH").map_or("?", |s| s)
    );

    // ── 2. Drop privileges ───────────────────────────────────────────
    if let Err(e) = drop_privileges(SERVICE_USER) {
        log::error!("Failed to drop privileges: {}", e);
        std::process::exit(1);
    }

    // ── 3. Bind Unix socket ──────────────────────────────────────────
    let listener = match create_unix_socket(SOCKET_PATH) {
        Ok(l)  => l,
        Err(e) => {
            log::error!("Cannot create socket {}: {}", SOCKET_PATH, e);
            std::process::exit(1);
        }
    };

    log::info!("Listening on {}", SOCKET_PATH);

    // ── 4. Accept loop ───────────────────────────────────────────────
    for stream in listener.incoming() {
        match stream {
            Ok(_conn) => {
                log::debug!("Accepted connection");
                // Handle connection in a thread / async runtime in production
            }
            Err(e) => {
                log::error!("Accept error: {}", e);
                break;
            }
        }
    }
}
```

### 9.3 Rust Host Tool — Device Table Generator

A Rust program that reads a simple TOML device specification and emits a
Buildroot-compatible device table. Run it from a post-build script or generate
the table at configure time.

```rust
// tools/gen_device_table/src/main.rs
// Cargo.toml: serde = { version = "1", features = ["derive"] }
//             toml  = "0.8"

use std::{env, fs};
use serde::Deserialize;

#[derive(Deserialize)]
struct DeviceSpec {
    devices: Vec<DevEntry>,
    files:   Vec<FileEntry>,
    dirs:    Vec<DirEntry>,
}

#[derive(Deserialize)]
struct DevEntry {
    path:  String,
    #[serde(rename = "type")]
    kind:  String,   // "c" or "b"
    mode:  u32,
    uid:   u32,
    gid:   u32,
    major: u32,
    minor: u32,
}

#[derive(Deserialize)]
struct FileEntry {
    path: String,
    mode: u32,
    uid:  u32,
    gid:  u32,
}

#[derive(Deserialize)]
struct DirEntry {
    path: String,
    mode: u32,
    uid:  u32,
    gid:  u32,
}

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Usage: {} <spec.toml>", args[0]);
        std::process::exit(1);
    }

    let content = fs::read_to_string(&args[1])
        .unwrap_or_else(|e| { eprintln!("Cannot read {}: {}", args[1], e); std::process::exit(1); });
    let spec: DeviceSpec = toml::from_str(&content)
        .unwrap_or_else(|e| { eprintln!("Parse error: {}", e); std::process::exit(1); });

    println!("# Auto-generated device table — do not edit by hand");
    println!("# Generated from: {}\n", args[1]);
    println!("# {:50}  type  mode  uid   gid   major  minor", "path");
    println!("{}", "#".repeat(80));

    for d in &spec.dirs {
        println!("{:52}  d     {:04o}  {:5}  {:5}  -      -",
                 d.path, d.mode, d.uid, d.gid);
    }

    println!();
    for f in &spec.files {
        println!("{:52}  f     {:04o}  {:5}  {:5}  -      -",
                 f.path, f.mode, f.uid, f.gid);
    }

    println!();
    for dev in &spec.devices {
        println!("{:52}  {}     {:04o}  {:5}  {:5}  {:5}  {:5}",
                 dev.path, dev.kind, dev.mode, dev.uid, dev.gid,
                 dev.major, dev.minor);
    }
}
```

**Example TOML input** (`board/myboard/devices.toml`):

```toml
[[dirs]]
path = "/var/run/myapp"
mode = 0o750
uid  = 1000
gid  = 1000

[[dirs]]
path = "/var/log/myapp"
mode = 0o750
uid  = 1000
gid  = 1000

[[files]]
path = "/etc/shadow"
mode = 0o640
uid  = 0
gid  = 42

[[devices]]
path  = "/dev/null"
type  = "c"
mode  = 0o666
uid   = 0
gid   = 0
major = 1
minor = 3

[[devices]]
path  = "/dev/ttyS0"
type  = "c"
mode  = 0o660
uid   = 0
gid   = 5
major = 4
minor = 64
```

---

## 10. Summary

```
  ╔══════════════════════════════════════════════════════════════════╗
  ║         Root Filesystem Configuration — Quick Reference          ║
  ╠══════════════════════════════════════════════════════════════════╣
  ║                                                                  ║
  ║  SKELETON (BR2_ROOTFS_SKELETON_CUSTOM)                           ║
  ║  ───────────────────────────────────                             ║
  ║  • Replaces the default BusyBox skeleton entirely.               ║
  ║  • Copied first — packages install on top of it.                 ║
  ║  • Use for: custom init, merged-usr, per-product /etc defaults.  ║
  ║                                                                  ║
  ║  DEVICE TABLE (BR2_ROOTFS_DEVICE_TABLE)                          ║
  ║  ──────────────────────────────────────                          ║
  ║  • Processed by makedevs during image generation.                ║
  ║  • Creates device nodes and corrects file ownership/mode.        ║
  ║  • No root required on the build host.                           ║
  ║  • Multiple space-separated table files supported.               ║
  ║                                                                  ║
  ║  USER TABLE (BR2_ROOTFS_USERS_TABLES)                            ║
  ║  ──────────────────────────────────────                          ║
  ║  • Processed by makeusers.                                       ║
  ║  • Writes passwd / shadow / group inside target rootfs.          ║
  ║  • UIDs in user table must match UIDs in device table.           ║
  ║  • Use uid=-1 for auto-assignment (non-deterministic — avoid).   ║
  ║                                                                  ║
  ║  POST-BUILD HOOKS (BR2_ROOTFS_POST_BUILD_SCRIPT)                 ║
  ║  ─────────────────────────────────────────────────               ║
  ║  • Execute after all packages install, before image creation.    ║
  ║  • Receive $1=TARGET_DIR  $2=IMAGES_DIR.                         ║
  ║  • Use for: metadata injection, file stripping, validation.      ║
  ║  • POST_IMAGE_SCRIPT runs after image files exist (signing, …).  ║
  ║                                                                  ║
  ║  EXECUTION ORDER                                                 ║
  ║  ───────────────                                                 ║
  ║                                                                  ║
  ║  [1] Skeleton copy                                               ║
  ║       │                                                          ║
  ║  [2] Package installs (may overwrite skeleton files)             ║
  ║       │                                                          ║
  ║  [3] User table applied  (makeusers)                             ║
  ║       │                                                          ║
  ║  [4] Post-build scripts  (BR2_ROOTFS_POST_BUILD_SCRIPT)          ║
  ║       │                                                          ║
  ║  [5] Device table applied (makedevs — sets modes/ownership)      ║
  ║       │                                                          ║
  ║  [6] Filesystem image created (squashfs / ext4 / cpio …)         ║
  ║       │                                                          ║
  ║  [7] Post-image scripts  (BR2_ROOTFS_POST_IMAGE_SCRIPT)          ║
  ║                                                                  ║
  ╠══════════════════════════════════════════════════════════════════╣
  ║  Key Rules                                                       ║
  ║  ──────────                                                      ║
  ║  +  Pin UIDs/GIDs numerically — never rely on auto-assign in     ║
  ║     production. Reproducible images require deterministic IDs.   ║
  ║  +  Device table is the only safe way to chown in a rootfs.      ║
  ║  +  Keep post-build scripts idempotent — they may run on an      ║
  ║     already-populated target dir after a partial rebuild.        ║
  ║  +  Validate mandatory files in post-build to catch missing      ║
  ║     package installs before the image is written.                ║
  ║  -  Do NOT use sudo inside post-build scripts.                   ║
  ║  -  Do NOT hardcode absolute host paths in scripts; use $1/$2.   ║
  ╚══════════════════════════════════════════════════════════════════╝
```

### Key Takeaways

**Skeleton** is about structure — it defines the bare-minimum directory hierarchy
and `/etc` defaults that every other layer builds upon. Using a custom skeleton
gives you full control over the shape of the root filesystem from the very first
file.

**Device and permissions tables** solve the fundamental problem of constructing
a correctly-owned Linux filesystem without root access on the build host. They
are declarative, auditable, and reproducible — prefer them over `chmod`/`chown`
calls in scripts.

**User tables** are the declarative equivalent of `useradd` for embedded
targets. Pairing them tightly with device tables (shared numeric UIDs) ensures
that files are owned by the right accounts regardless of build host state.

**Post-build hooks** are your escape valve — the place to inject runtime-specific
information, enforce policies, and validate the filesystem before any bytes are
committed to a flash image. Keep them lean, idempotent, and fail-fast.

Together, these four mechanisms let you build a root filesystem that is minimal,
secure, reproducible, and auditable — essential qualities for any production
embedded Linux product.

---

*Document generated for the Buildroot Series, Topic 15.*
*Language examples: C · C++ · Rust | Graphics: ASCII*