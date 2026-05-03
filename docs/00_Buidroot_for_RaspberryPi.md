# Buildroot Image for Raspberry Pi (1–5) with Qt, I2C & SPI Support

**Structure of the guide:**

- **§1 Prerequisites** — host packages, disk/RAM requirements
- **§2 Board mapping** — all RPi 1–5 variants mapped to their exact defconfig names and DTBs
- **§3–4 Getting & configuring Buildroot** — clone, stable checkout, loading defconfigs
- **§5 I2C & SPI** — kernel `menuconfig` paths for `CONFIG_I2C_BCM2835`, `CONFIG_SPI_BCM2835`, `CONFIG_SPI_SPIDEV`, plus `config.txt` overlays (`dtparam=i2c_arm=on`, `dtparam=spi=on`)
- **§6 Qt5 / Qt6** — LinuxFB vs EGLFS vs Wayland backends, Mesa/V3D/VC4 GPU setup, toolchain C++17 requirements
- **§7–8 Extra packages** — `i2c-tools`, `spi-tools`, fonts, SSH, init system choices
- **§9 Building** — `make -j$(nproc)`, incremental rebuilds, output artefacts
- **§10 Flashing** — `dd`, `bmaptool`, partition layout
- **§11 Verification** — runtime checks for I2C, SPI, and Qt platform plugins
- **§12 Board-by-board notes** — specific caveats for each RPi generation (ARMv6 limitations on RPi 1, RP1 south-bridge on RPi 5, etc.)
- **§13 Troubleshooting** — common failure modes with fixes
- **§14 Config summary** — a single cheat-sheet of all key `BR2_*` symbols


> **Target audience:** Embedded Linux developers who want a minimal, reproducible OS image for Raspberry Pi boards using [Buildroot](https://buildroot.org/).  
> **Buildroot version used as reference:** 2024.02 LTS  
> **Host OS assumed:** Ubuntu 22.04 / Debian 12 (64-bit)

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Board-Specific Target Names](#2-board-specific-target-names)
3. [Getting Buildroot](#3-getting-buildroot)
4. [Loading a Board Defconfig](#4-loading-a-board-defconfig)
5. [Configuring the Kernel — I2C & SPI](#5-configuring-the-kernel--i2c--spi)
6. [Configuring Qt5 / Qt6](#6-configuring-qt5--qt6)
7. [Additional Useful Packages](#7-additional-useful-packages)
8. [Filesystem & Init System](#8-filesystem--init-system)
9. [Building the Image](#9-building-the-image)
10. [Flashing to SD Card](#10-flashing-to-sd-card)
11. [Runtime Verification](#11-runtime-verification)
12. [Board-by-Board Notes](#12-board-by-board-notes)
13. [Troubleshooting](#13-troubleshooting)
14. [Reference Configuration Summary](#14-reference-configuration-summary)

---

## 1. Prerequisites

### 1.1 Host packages

Install all required build dependencies on the host machine:

```bash
sudo apt update
sudo apt install -y \
  build-essential gcc g++ make cmake \
  git wget curl unzip \
  bc cpio rsync \
  python3 python3-pip \
  libncurses-dev libssl-dev \
  bison flex \
  file libelf-dev \
  parted dosfstools mtools \
  qemu-user-static    # optional: for QEMU-based testing
```

### 1.2 Disk space & RAM

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| Free disk (build tree) | 20 GB | 40 GB |
| Free disk (with Qt) | 35 GB | 60 GB |
| RAM | 4 GB | 8 GB |
| CPU cores | 2 | 4+ |

### 1.3 User permissions

Do **not** build as root. A regular user with `sudo` rights is sufficient.

---

## 2. Board-Specific Target Names

Each Raspberry Pi model maps to a Buildroot defconfig and a Device Tree:

| Board | Buildroot defconfig | Default DTB |
|-------|---------------------|-------------|
| RPi 1 (Model A/B/B+/Zero) | `raspberrypi_defconfig` | `bcm2835-rpi-b-plus.dtb` |
| RPi 2 | `raspberrypi2_defconfig` | `bcm2836-rpi-2-b.dtb` |
| RPi 3 (32-bit) | `raspberrypi3_defconfig` | `bcm2837-rpi-3-b.dtb` |
| RPi 3 (64-bit) | `raspberrypi3_64_defconfig` | `bcm2837-rpi-3-b.dtb` |
| RPi 4 (32-bit) | `raspberrypi4_defconfig` | `bcm2711-rpi-4-b.dtb` |
| RPi 4 (64-bit) | `raspberrypi4_64_defconfig` | `bcm2711-rpi-4-b.dtb` |
| RPi 5 (64-bit) | `raspberrypi5_defconfig` | `bcm2712-rpi-5-b.dtb` |

> **RPi Zero / Zero W / Zero 2 W:**  
> Zero and Zero W → use `raspberrypi_defconfig` (ARMv6).  
> Zero 2 W → use `raspberrypi3_defconfig` (ARMv8 32-bit) or the 64-bit variant.

---

## 3. Getting Buildroot

### 3.1 Clone the repository

```bash
git clone https://git.buildroot.net/buildroot
cd buildroot
```

### 3.2 Checkout a stable release (recommended)

```bash
# List available tags
git tag | grep -E '^[0-9]{4}\.' | tail -10

# Checkout the latest LTS
git checkout 2024.02
```

### 3.3 Directory structure overview

```
buildroot/
├── board/          # Board-specific scripts and overlays
├── configs/        # All defconfigs (raspberrypi*, etc.)
├── dl/             # Downloaded source tarballs (auto-populated)
├── output/         # Build artefacts (created during build)
├── package/        # Package definitions
└── .config         # Active configuration (created by make)
```

---

## 4. Loading a Board Defconfig

Replace `<DEFCONFIG>` with the value from the table in §2.

```bash
# Example: Raspberry Pi 4 64-bit
make raspberrypi4_64_defconfig
```

This populates `.config` with sensible defaults for the chosen board.

### 4.1 Open the interactive menu

```bash
make menuconfig
```

The TUI (`ncurses`) menu is the primary way to customise the build. All subsequent sections describe paths inside this menu.

---

## 5. Configuring the Kernel — I2C & SPI

### 5.1 Open the kernel configuration sub-menu

Inside `make menuconfig`:

```
Kernel  →
  Linux Kernel  →
    [*] Linux Kernel
    Kernel configuration: Using a custom (def)config file
    Defconfig name: bcmrpi_defconfig      # RPi 1/Zero
                    bcm2709_defconfig      # RPi 2
                    bcm2710_defconfig      # RPi 3
                    bcm2711_defconfig      # RPi 4
                    bcm2712_defconfig      # RPi 5
```

Then open the *kernel menuconfig* directly:

```bash
make linux-menuconfig
```

### 5.2 Enable I2C

Navigate to:

```
Device Drivers  →
  I2C support  →
    <*> I2C support
    [*]   Enable compatibility bits for old user-space
    I2C Hardware Bus support  →
      <*> Broadcom BCM2835 I2C controller
```

Also enable the user-space device node:

```
Device Drivers  →
  I2C support  →
    [*] I2C device interface        # creates /dev/i2c-*
```

### 5.3 Enable SPI

Navigate to:

```
Device Drivers  →
  SPI support  →
    <*> SPI support
    [*]   SPI master
    <*>   BCM2835 SPI controller
    [*] User mode SPI device driver support   # creates /dev/spidev*
```

### 5.4 Enable Device Tree overlays for I2C / SPI

Buildroot uses the Raspberry Pi firmware bootloader. Enable overlays in the board config overlay file:

```bash
# Create (or edit) the board overlay config.txt
mkdir -p board/raspberrypi/overlay
cat >> board/raspberrypi/overlay/config.txt << 'EOF'

# Enable I2C
dtparam=i2c_arm=on
dtparam=i2c_vc=on

# Enable SPI
dtparam=spi=on

# Optional: enable SPI1 (RPi 4 / 5)
# dtoverlay=spi1-3cs
EOF
```

Point Buildroot to this overlay in `menuconfig`:

```
System configuration  →
  Root filesystem overlay directories: board/raspberrypi/overlay
```

### 5.5 Save and exit kernel menuconfig

Press `Esc` → `Esc` → select **Yes** to save. The kernel config is saved to `output/build/linux-<ver>/.config` (and persisted via the Buildroot build system).

---

## 6. Configuring Qt5 / Qt6

### 6.1 Choose a graphics backend

Before enabling Qt, decide on the graphics stack:

| Backend | When to use | Buildroot option |
|---------|-------------|-----------------|
| **LinuxFB** | Simple framebuffer, no GPU, RPi 1–3 | `BR2_PACKAGE_QT5BASE_LINUXFB` |
| **EGLFS + VC4/V3D** | Full OpenGL/ES on RPi 3–5 | `BR2_PACKAGE_QT5BASE_EGLFS` |
| **Wayland** | Compositor-based desktop, RPi 4–5 | `BR2_PACKAGE_QT5WAYLAND` |
| **xcb / X11** | Full X11 desktop | `BR2_PACKAGE_QT5BASE_XCB` |

For headless or simple HMI use, **LinuxFB** or **EGLFS** is recommended.

### 6.2 Enable Qt5 in menuconfig

```
Target packages  →
  Graphic libraries and applications (graphic/video/...)  →
    Qt5  →
      [*] qt5base
            Gui module (*)
            Widgets module (*)
            [*] LinuxFB support           ← framebuffer backend
            [*] EGL support               ← if using GPU
            [*] OpenGL ES 2.0 support     ← if using GPU
      [*] qt5declarative                  ← QML / Qt Quick
      [*] qt5serialbus                    ← optional: CAN bus
      [*] qt5serialport                   ← optional: serial ports
```

### 6.3 Enable Qt6 (alternative — RPi 4/5 recommended)

Qt6 support in Buildroot is available from 2023.11+:

```
Target packages  →
  Graphic libraries and applications  →
    Qt6  →
      [*] qt6base
            [*] Gui module
            [*] Widgets module
            [*] LinuxFB support
            [*] OpenGL ES 2.0 support
      [*] qt6declarative
      [*] qt6shadertools    ← required by qt6declarative
```

> ⚠️ Qt6 requires a C++17-capable toolchain. Make sure your Buildroot toolchain is GCC 10+ (`Toolchain → Toolchain type → Buildroot toolchain → GCC compiler version → gcc 12.x`).

### 6.4 Toolchain requirements for Qt

```
Toolchain  →
  [*] Enable C++ support            ← mandatory
  [*] Enable thread library debugging support
  C library: glibc                  ← recommended (musl has Qt limitations)
  GCC compiler version: gcc 12.x
```

### 6.5 GPU / OpenGL support (RPi 3, 4, 5)

For hardware-accelerated Qt rendering:

```
Target packages  →
  Graphic libraries and applications  →
    [*] mesa3d
          [*] OpenGL ES
          [*] GBM
          Gallium drivers: vc4 (RPi 1–3), v3d (RPi 4–5)
    [*] libdrm
    [*] libegl-mesa
```

Also in `Kernel → linux-menuconfig`:

```
Device Drivers  →
  Graphics support  →
    <*> Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)
    <*>   Broadcom VC4 Graphics     ← RPi 1–3
    <*>   Broadcom V3D              ← RPi 4–5
```

---

## 7. Additional Useful Packages

### 7.1 I2C and SPI userspace tools

```
Target packages  →
  Hardware handling  →
    [*] i2c-tools      # i2cdetect, i2cdump, i2cget, i2cset
    [*] spi-tools      # spidev_test
```

### 7.2 Runtime libraries commonly needed with Qt + hardware

```
Target packages  →
  Libraries  →
    Other  →
      [*] tslib          # Touchscreen calibration
      [*] libgpiod       # GPIO access from Qt/C++
      [*] libinput       # Input device handling for EGLFS
```

### 7.3 Fonts (required for Qt to render text)

```
Target packages  →
  Fonts, cursors, icons, sounds and themes  →
    [*] dejavu           # DejaVu Sans/Serif/Mono
    [*] liberation       # Metric-compatible with Arial/Times
```

### 7.4 SSH & development helpers

```
Target packages  →
  Networking applications  →
    [*] openssh
    [*] dropbear         # lighter alternative to openssh

  Development tools  →
    [*] gdb
    [*] gdbserver
    [*] strace
```

---

## 8. Filesystem & Init System

### 8.1 Init system

```
System configuration  →
  Init system: BusyBox         # minimal (default)
             | systemd         # feature-rich (RPi 3–5 recommended)
```

For Qt applications that need D-Bus (required by some Qt modules), `systemd` is recommended.

### 8.2 Root filesystem image type

```
Filesystem images  →
  [*] ext2/3/4 root filesystem
        ext2/3/4 variant: ext4
        exact size: 512M           # adjust to your needs
  [*] tar the root filesystem      # optional: for NFS or inspection
```

### 8.3 Root password & hostname

```
System configuration  →
  System hostname: rpi-buildroot
  System banner: Welcome to Buildroot RPi
  Root password: (set your password here)
```

### 8.4 Enable package-managers (optional)

Buildroot is not designed for a runtime package manager, but `opkg` can be added for field updates:

```
Target packages  →
  System tools  →
    [*] opkg
```

---

## 9. Building the Image

### 9.1 Save your configuration

Before building, save your customised config to a named defconfig:

```bash
make savedefconfig BR2_DEFCONFIG=configs/my_rpi4_qt_defconfig
```

### 9.2 Start the build

```bash
# Use all available CPU cores
make -j$(nproc)
```

> ☕ First build: 1–3 hours depending on hardware and internet speed.  
> Subsequent builds (after changes): typically 5–20 minutes.

### 9.3 Build artefacts

After a successful build, the following files are in `output/images/`:

| File | Description |
|------|-------------|
| `sdcard.img` | Complete SD card image (bootloader + kernel + rootfs) |
| `zImage` / `Image` | Compressed kernel |
| `rootfs.ext4` | Root filesystem image |
| `*.dtb` | Device Tree Blobs |
| `rpi-firmware/` | Bootloader files (bootcode.bin, start.elf, etc.) |

### 9.4 Incremental builds

After changing a package or config:

```bash
# Rebuild only a single package
make qt5base-rebuild

# Rebuild Linux kernel only
make linux-rebuild

# Rebuild rootfs image without full recompile
make

# Clean everything (nuclear option)
make clean
```

---

## 10. Flashing to SD Card

### 10.1 Using `dd`

```bash
# Identify your SD card device (DANGEROUS — double-check!)
lsblk

# Flash — replace /dev/sdX with your actual device
sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M conv=fsync status=progress
sudo sync
```

### 10.2 Using `bmaptool` (faster, safer)

```bash
sudo apt install bmap-tools
sudo bmaptool copy output/images/sdcard.img /dev/sdX
```

### 10.3 Using Raspberry Pi Imager (GUI alternative)

Open Raspberry Pi Imager → **Use custom** → select `sdcard.img`.

### 10.4 Verify partitions after flashing

```bash
sudo fdisk -l /dev/sdX
# Expected:
# /dev/sdX1  FAT32  ~64 MB   (boot partition: kernel, DTBs, firmware)
# /dev/sdX2  ext4   ~512 MB  (root filesystem)
```

---

## 11. Runtime Verification

### 11.1 Check I2C

```bash
# List I2C buses
ls /dev/i2c-*

# Scan bus 1 (GPIO pins 2 & 3 on all RPi models)
i2cdetect -y 1

# Read a register from a device at address 0x48
i2cget -y 1 0x48 0x00
```

### 11.2 Check SPI

```bash
# List SPI devices
ls /dev/spidev*

# Loopback test (connect MOSI to MISO)
spidev_test -D /dev/spidev0.0 -v
```

### 11.3 Launch a Qt application

```bash
# Framebuffer backend (no display server needed)
export QT_QPA_PLATFORM=linuxfb
/usr/bin/my_qt_app

# EGLFS backend
export QT_QPA_PLATFORM=eglfs
/usr/bin/my_qt_app

# Debug: list available platforms
/usr/bin/my_qt_app -platform help
```

### 11.4 Check Qt platform plugins

```bash
ls /usr/lib/qt/plugins/platforms/
# Expected: libqlinuxfb.so, libqeglfs.so, libqminimal.so, etc.
```

---

## 12. Board-by-Board Notes

### RPi 1 / Zero / Zero W (ARMv6, BCM2835)

- Use `raspberrypi_defconfig`.
- Architecture: `arm (ARMv6)` — set in `Toolchain → Architecture → arm` with `Architecture variant: arm1176jzf-s`.
- Qt5 with LinuxFB is the only practical GUI backend (no open-source OpenGL driver for VC4 on ARMv6).
- GPU memory: set `gpu_mem=16` in `config.txt` for a headless build, or `gpu_mem=128` for display.
- RAM is 256–512 MB — keep the rootfs lean. Avoid full Qt WebEngine.

### RPi 2 (ARMv7, BCM2836)

- Use `raspberrypi2_defconfig`.
- Architecture: `cortex-A7`, enable NEON: `BR2_ARM_FPU_NEON_VFPV4=y`.
- VC4 Mesa driver available but limited. LinuxFB or EGLFS both viable.

### RPi 3 (ARMv8, BCM2837)

- Use `raspberrypi3_64_defconfig` for a 64-bit image (recommended).
- VC4 Mesa (`gallium-vc4`) gives hardware-accelerated OpenGL ES 2.0.
- Enable 64-bit: `Toolchain → Architecture: aarch64`.
- Qt5 EGLFS + VC4 Mesa works well for HMI applications.
- Bluetooth / WiFi require firmware blobs: add `BR2_PACKAGE_RPI_WIFI_FIRMWARE=y`.

### RPi 4 (BCM2711)

- Use `raspberrypi4_64_defconfig`.
- V3D Mesa (`gallium-v3d`) provides OpenGL ES 3.1 and Vulkan (via `turnip` — experimental).
- 64-bit only recommended. Supports up to 8 GB RAM.
- Dual display via two HDMI ports; requires additional DT configuration.
- USB 3.0 enabled by default in the kernel defconfig.
- Use `config.txt` entry `dtoverlay=vc4-kms-v3d` for KMS/DRM display.

### RPi 5 (BCM2712)

- Use `raspberrypi5_defconfig` (64-bit only).
- **Requires Buildroot ≥ 2024.02** for proper BCM2712 support.
- PCIe 2.0 interface exposed for NVMe HATs.
- New RP1 south-bridge chip handles USB, Ethernet, GPIO, I2C, SPI — DTB overlays differ from RPi 4.
- EGLFS + V3D Mesa recommended; KMS/DRM support mature.
- Add `dtparam=i2c_arm=on` and `dtparam=spi=on` in `config.txt` as on previous models.
- The RPi 5 also supports I2C on the dedicated `i2c0` (ID EEPROM bus) — don't confuse with `i2c1` (user GPIO bus 2/3).

---

## 13. Troubleshooting

### Build fails: "host-python3 not found"

```bash
sudo apt install python3 python3-distutils
```

### Qt platform plugin not found at runtime

```
This application failed to start because no Qt platform plugin could be initialized.
```

Fix: ensure `BR2_PACKAGE_QT5BASE_LINUXFB=y` (or the relevant plugin) is enabled, then rebuild:

```bash
make qt5base-rebuild && make
```

### I2C devices not appearing (`/dev/i2c-*` missing)

1. Verify `dtparam=i2c_arm=on` is in `/boot/config.txt` on the SD card.
2. Confirm `CONFIG_I2C_BCM2835=y` and `CONFIG_I2C_CHARDEV=y` in the kernel config.
3. Check dmesg: `dmesg | grep i2c`.

### SPI device missing (`/dev/spidev*` missing)

1. Verify `dtparam=spi=on` in `config.txt`.
2. Confirm `CONFIG_SPI_BCM2835=y` and `CONFIG_SPI_SPIDEV=y`.
3. Check dmesg: `dmesg | grep spi`.

### Qt rendering is blank / black screen

- Check the framebuffer: `cat /dev/urandom > /dev/fb0` — should show noise.
- Try `QT_QPA_PLATFORM=linuxfb:fb=/dev/fb0`.
- For EGLFS: verify Mesa and DRM kernel driver are loaded (`dmesg | grep drm`).
- Set `export QT_LOGGING_RULES="qt.qpa.*=true"` for verbose Qt platform logging.

### Out of disk space during build

Buildroot's `dl/` directory caches all source tarballs. You can symlink it to a larger disk:

```bash
mkdir -p /data/buildroot-dl
ln -s /data/buildroot-dl buildroot/dl
```

### Slow rebuild after minor changes

Use `ccache` to speed up recompilation:

```bash
# In menuconfig:
Build options  →
  [*] Enable compiler cache
      Compiler cache location: $(HOME)/.buildroot-ccache
```

---

## 14. Reference Configuration Summary

The following is a condensed checklist of all key `menuconfig` options covered in this guide:

```
### Toolchain
BR2_TOOLCHAIN_BUILDROOT_CXX=y
BR2_GCC_VERSION_12_X=y
BR2_TOOLCHAIN_BUILDROOT_GLIBC=y

### Kernel
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_DEFCONFIG="bcm2711"     # adjust per board
# Via linux-menuconfig:
CONFIG_I2C=y
CONFIG_I2C_BCM2835=y
CONFIG_I2C_CHARDEV=y
CONFIG_SPI=y
CONFIG_SPI_BCM2835=y
CONFIG_SPI_SPIDEV=y
CONFIG_DRM=y
CONFIG_DRM_V3D=y                          # RPi 4/5
CONFIG_DRM_VC4=y                          # RPi 1–3

### System
BR2_SYSTEM_DHCP="eth0"
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y

### Qt5
BR2_PACKAGE_QT5=y
BR2_PACKAGE_QT5BASE=y
BR2_PACKAGE_QT5BASE_GUI=y
BR2_PACKAGE_QT5BASE_WIDGETS=y
BR2_PACKAGE_QT5BASE_LINUXFB=y
BR2_PACKAGE_QT5BASE_EGLFS=y
BR2_PACKAGE_QT5DECLARATIVE=y

### Mesa / GPU (RPi 3+)
BR2_PACKAGE_MESA3D=y
BR2_PACKAGE_MESA3D_OPENGL_ES=y
BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_VC4=y  # RPi 1–3
BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_V3D=y  # RPi 4–5
BR2_PACKAGE_LIBDRM=y

### Hardware tools
BR2_PACKAGE_I2C_TOOLS=y
BR2_PACKAGE_SPI_TOOLS=y
BR2_PACKAGE_LIBGPIOD=y

### Fonts
BR2_PACKAGE_DEJAVU=y
```

---

## Further Reading

- [Buildroot User Manual](https://buildroot.org/downloads/manual/manual.html)
- [Raspberry Pi Linux Kernel Source](https://github.com/raspberrypi/linux)
- [Qt for Embedded Linux](https://doc.qt.io/qt-6/embedded-linux.html)
- [Buildroot RPi board support files](https://github.com/buildroot/buildroot/tree/master/board/raspberrypi)
- [RPi `config.txt` reference](https://www.raspberrypi.com/documentation/computers/config_txt.html)

---

*Generated for Buildroot 2024.02 — verify package names against your Buildroot version with `make searchall BR2_PACKAGE_NAME=<name>`.*