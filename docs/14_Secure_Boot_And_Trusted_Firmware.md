# 14. Secure Boot & Trusted Firmware (TF-A / OP-TEE)

**Architecture & Concepts**
- Full trust chain from ROM/eFUSE → TF-A boot stages (BL1→BL2→BL31→BL32→BL33→Linux)
- TrustZone memory map with NS-bit / TZASC / TZPC explained
- OP-TEE software stack from `libteec.so` → `tee.ko` → SMC → BL31 → OP-TEE → TA
- FIP (Firmware Image Package) and FIT image internal layouts

**Buildroot Integration**
- Full `BR2_*` config snippet for TF-A, OP-TEE OS, OP-TEE client, U-Boot FIT signing
- `post-image.sh` assembling the FIP bundle with `fiptool`
- FIT `.its` template and the `mkimage` signing workflow

**C Examples**
- Normal World Client App (CA) calling a TEE TA for AES-256-CBC
- Trusted Application (TA) using the GP `TEE_*` Internal Crypto API
- Secure persistent key storage using `TEE_STORAGE_PRIVATE`
- BL31 custom SiP SMC handler with `DECLARE_RT_SVC`

**C++ Examples**
- RAII `Context` / `Session` wrappers preventing TEE resource leaks
- Full `KeyProvisioner` class for ECC key generation, public key export, and signing

**Rust Examples**
- `AesSession` struct with `Zeroizing<Vec<u8>>` for automatic key scrubbing on drop
- FIT image RSA-SHA256 verification tool using OpenSSL bindings
- Secure Boot status inspector reading TEE sysfs nodes


> **Buildroot Series — Chapter 14**  
> Integrating ARM Trusted Firmware-A, OP-TEE OS, signing images with `fit-image`,  
> and Buildroot packages `arm-trusted-firmware`, `optee-os`, `optee-client`.

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [ARM Trust Zone Fundamentals](#2-arm-trustzone-fundamentals)
3. [Boot Chain Anatomy](#3-boot-chain-anatomy)
4. [ARM Trusted Firmware-A (TF-A)](#4-arm-trusted-firmware-a-tf-a)
5. [OP-TEE OS & Client](#5-op-tee-os--client)
6. [FIT Image Signing](#6-fit-image-signing)
7. [Buildroot Integration](#7-buildroot-integration)
8. [C Programming Examples](#8-c-programming-examples)
9. [C++ Programming Examples](#9-c-programming-examples-1)
10. [Rust Programming Examples](#10-rust-programming-examples)
11. [Key Management & PKI](#11-key-management--pki)
12. [Debugging & Attestation](#12-debugging--attestation)
13. [Security Hardening Checklist](#13-security-hardening-checklist)
14. [Summary](#14-summary)

---

## 1. Overview & Architecture

Secure Boot is the mechanism that guarantees only cryptographically verified, trusted
software executes on an embedded device — from the very first instruction out of ROM
to the full Linux userspace. Combined with ARM TrustZone hardware isolation,
Trusted Firmware-A (TF-A) and OP-TEE provide a complete **Trusted Execution
Environment (TEE)** for ARM-based SoCs.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SECURE BOOT & TEE OVERVIEW                          │
│                                                                         │
│  ┌──────────────────────┐    ┌───────────────────────────────────────┐  │
│  │   NORMAL WORLD (REE) │    │       SECURE WORLD (TEE)              │  │
│  │                      │    │                                       │  │
│  │  ┌────────────────┐  │    │  ┌─────────────────────────────────┐  │  │
│  │  │  Linux Kernel  │  │    │  │         OP-TEE OS               │  │  │
│  │  └───────┬────────┘  │    │  │  ┌───────────┐ ┌─────────────┐  │  │  │
│  │          │           │    │  │  │ Trusted   │ │  Crypto     │  │  │  │
│  │  ┌───────┴────────┐  │    │  │  │   Apps    │ │  Services   │  │  │  │
│  │  │ TEE Client API │◄─┼────┼──┼──┤  (TAs)    │ │  (AES/RSA)  │  │  │  │
│  │  │  (optee-client)│  │    │  │  └───────────┘ └─────────────┘  │  │  │
│  │  └────────────────┘  │    │  └─────────────────────────────────┘  │  │
│  │                      │    │                                       │  │
│  └──────────────────────┘    └───────────────────────────────────────┘  │
│                                                                         │
│  ═══════════════════════ ARM TrustZone Hardware ═══════════════════════ │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                  ARM Trusted Firmware-A (EL3)                   │    │
│  │   BL1 (ROM)  ──►  BL2 (Trusted Boot)  ──►  BL31 (Secure Mon.)   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component        | Role                                      | Exception Level |
|------------------|-------------------------------------------|-----------------|
| BL1 (ROM/BL1)    | First-stage bootloader, ROM code          | EL3             |
| BL2              | Trusted Boot firmware, loads others       | S-EL1           |
| BL31 (TF-A)      | Secure Monitor, SMC handler               | EL3             |
| BL32 (OP-TEE)    | Trusted OS (Secure OS kernel)             | S-EL1           |
| BL33 (U-Boot)    | Normal-world bootloader                   | EL2 / EL1       |
| Linux Kernel     | Rich OS / REE                             | EL1             |
| Trusted Apps     | Security-sensitive logic in TEE           | S-EL0           |
| optee-client     | Linux-side API + TEE supplicant daemon    | EL0             |

---

## 2. ARM TrustZone Fundamentals

TrustZone divides every CPU core, memory region, and peripheral into two security
domains enforced in hardware.

```
┌───────────────────────────────────────────────────────────────────────┐
│                   ARM TRUSTZONE MEMORY MAP                            │
│                                                                       │
│  Physical Address Space                                               │
│  0xFFFFFFFF ┌─────────────────────────────────────────────────────┐   │
│             │  Secure ROM / BL1 entry (platform-specific)         │   │
│             ├─────────────────────────────────────────────────────┤   │
│             │  Trusted SRAM  [S-bit=1, NS=0]                      │   │
│             │  ┌──────────┐ ┌──────────┐ ┌───────────────────┐    │   │
│             │  │   BL2    │ │  BL31    │ │   OP-TEE (BL32)   │    │   │
│             │  │  (Boot)  │ │  (Mon.)  │ │   Heap/Stack      │    │   │
│             │  └──────────┘ └──────────┘ └───────────────────┘    │   │
│             ├─────────────────────────────────────────────────────┤   │
│             │  Shared Memory (NS-bit set, TEE-configured)         │   │
│             │  ← used for SMC parameter passing →                 │   │
│             ├─────────────────────────────────────────────────────┤   │
│             │  Non-Secure DRAM  [NS=1]                            │   │
│             │  ┌──────────────┐ ┌─────────────────────────────┐   │   │
│             │  │  U-Boot/BL33 │ │   Linux Kernel + Userspace  │   │   │
│             │  └──────────────┘ └─────────────────────────────┘   │   │
│  0x00000000 └─────────────────────────────────────────────────────┘   │
│                                                                       │
│  NS-bit: Non-Secure Memory Attribute (AXI bus signal)                 │
│  TZASC:  TrustZone Address Space Controller (enforces NS-bit)         │
│  TZPC:   TrustZone Protection Controller (peripheral NS/S)            │
└───────────────────────────────────────────────────────────────────────┘
```

### SMC (Secure Monitor Call) Flow

```
  Normal World (EL1/EL2)              Secure Monitor (EL3)       Secure World (S-EL1)
  ──────────────────────              ────────────────────       ────────────────────
  optee_open_session()
         │
         ▼
  ioctl(TEE_IOC_OPEN_SESSION)
         │
         ▼
  tee_supplicant daemon ─────────────► SMC #0 (OPTEE_SMC_CALL)
                                              │
                                              ▼
                                       BL31 SMC handler
                                       switches to S-EL1
                                              │
                                              ▼
                                                          OP-TEE dispatcher
                                                          routes to TA session
                                                          returns result
                                              │
                                       BL31 returns
                                       to NS-EL1
         ◄─────────────────────────────────────
  Session handle returned
```

---

## 3. Boot Chain Anatomy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECURE BOOT CHAIN  (ARMv8-A)                             │
│                                                                             │
│  POWER ON                                                                   │
│     │                                                                       │
│     ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  BL1  — On-chip ROM / eFUSE-anchored trust root                      │   │
│  │  • Loads BL2 from storage (eMMC/NOR/NAND)                            │   │
│  │  • Verifies BL2 signature with OEM public key burned into OTP/eFUSE  │   │
│  │  • If invalid → halt / lockout                                       │   │
│  └────────────────────────────┬─────────────────────────────────────────┘   │
│                               │  Verified ✓                                 │
│                               ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  BL2  — Trusted Boot firmware                                        │   │
│  │  • Loads and verifies BL31, BL32 (OP-TEE), BL33 (U-Boot)             │   │
│  │  • Configures TZASC / TZPC memory protection                         │   │
│  │  • Populates firmware handoff (FW_CONFIG, HW_CONFIG DTBs)            │   │
│  └────────────────────────────┬─────────────────────────────────────────┘   │
│                               │  Verified ✓                                 │
│                               ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  BL31 — Secure Monitor (runs at EL3 forever)                         │   │
│  │  • Installs EL3 exception vectors                                    │   │
│  │  • Manages power states (PSCI)                                       │   │
│  │  • Dispatches SMCs to OP-TEE or SPMD                                 │   │
│  └───────┬───────────────────────────────────────┬──────────────────────┘   │
│          │  Hands off to BL32                    │  Hands off to BL33       │
│          ▼                                       ▼                          │
│  ┌──────────────────────┐              ┌─────────────────────────────┐      │
│  │  BL32  — OP-TEE OS   │              │  BL33  — U-Boot             │      │
│  │  • TEE kernel init   │              │  • Verifies FIT image       │      │
│  │  • TA loader / mgr   │              │  • Kernel + DTB signature   │      │
│  │  • Waits for SMC     │              │  • Boots Linux              │      │
│  └──────────────────────┘              └──────────────┬──────────────┘      │
│                                                       │  Verified ✓         │
│                                                       ▼                     │
│                                           ┌─────────────────────────┐       │
│                                           │  Linux Kernel  (EL1)    │       │
│                                           │  + optee-client driver  │       │
│                                           └─────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘

  Legend:
  eFUSE/OTP  ─► Hardware trust anchor, written once, read-only thereafter
  TZASC      ─► Enforces NS/S memory split in DRAM controller
  FIT image  ─► Flattened uImage Tree: kernel + DTB + ramdisk, all signed
```

---

## 4. ARM Trusted Firmware-A (TF-A)

TF-A (formerly ARM Trusted Firmware) is the reference implementation of secure
world firmware for ARMv8-A. It provides BL1, BL2, and BL31 along with platform
abstraction layers.

### 4.1 Buildroot Package: `arm-trusted-firmware`

```
BR2_TARGET_ARM_TRUSTED_FIRMWARE=y
BR2_TARGET_ARM_TRUSTED_FIRMWARE_PLATFORM="rpi4"        # or "fvp", "qemu", "sun50i_a64", etc.
BR2_TARGET_ARM_TRUSTED_FIRMWARE_UBOOT_AS_BL33=y        # U-Boot as BL33
BR2_TARGET_ARM_TRUSTED_FIRMWARE_BL31=y
BR2_TARGET_ARM_TRUSTED_FIRMWARE_ADDITIONAL_VARIABLES="DEBUG=0 LOG_LEVEL=20"
```

### 4.2 TF-A Platform Port Directory Structure

```
arm-trusted-firmware/
├── plat/
│   ├── arm/                    # ARM reference platforms (FVP, Juno)
│   │   └── fvp/
│   │       ├── fvp_bl1_setup.c
│   │       ├── fvp_bl2_setup.c
│   │       └── fvp_bl31_setup.c
│   ├── rpi/rpi4/              # Raspberry Pi 4
│   └── allwinner/             # Allwinner SoCs (sun50i, sun8i)
├── bl1/                        # BL1 generic code
├── bl2/                        # BL2 generic code
├── bl31/                       # BL31 Secure Monitor
│   └── aarch64/
│       └── bl31_entrypoint.S
├── lib/
│   ├── psci/                   # Power State Coordination Interface
│   └── el3_runtime/            # EL3 exception handling
└── include/
    ├── common/
    └── plat/
```

### 4.3 PSCI (Power State Coordination Interface)

TF-A implements PSCI so Linux can request CPU hot-plug and system suspend via
standardized SMC calls.

```
PSCI SMC Call Numbers (ARM SMC Calling Convention):
  CPU_SUSPEND    0x84000001  (32-bit) / 0xC4000001  (64-bit)
  CPU_OFF        0x84000002
  CPU_ON         0x84000003
  AFFINITY_INFO  0x84000004
  SYSTEM_RESET   0x84000009
  SYSTEM_OFF     0x84000008
```

---

## 5. OP-TEE OS & Client

OP-TEE (Open Portable Trusted Execution Environment) is a TEE OS for ARM TrustZone,
following the GlobalPlatform TEE specification. It runs as BL32 in the secure world
and exposes a TEE Client API to the normal-world Linux side.

### 5.1 Buildroot Packages

```
BR2_PACKAGE_OPTEE_OS=y
BR2_PACKAGE_OPTEE_OS_PLATFORM="vexpress-qemu_virt"   # or "rpi4", "stm32mp1", etc.
BR2_PACKAGE_OPTEE_OS_PLATFORM_FLAVOR=""
BR2_PACKAGE_OPTEE_CLIENT=y
BR2_PACKAGE_OPTEE_CLIENT_EXAMPLES=y
```

### 5.2 OP-TEE Software Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      OP-TEE SOFTWARE STACK                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  NORMAL WORLD (Linux)                        │   │
│  │                                                              │   │
│  │  User App         CA (Client App)                            │   │
│  │     │                    │                                   │   │
│  │     └────────────────────┤                                   │   │
│  │                          ▼                                   │   │
│  │              ┌──────────────────────┐                        │   │
│  │              │  libteec.so          │  TEE Client API        │   │
│  │              │  (optee-client)      │  GP TEE spec           │   │
│  │              └──────────┬───────────┘                        │   │
│  │                         │  /dev/tee0  (ioctl)                │   │
│  │              ┌──────────▼───────────┐                        │   │
│  │              │  tee.ko (kernel drv) │  Linux TEE subsystem   │   │
│  │              └──────────┬───────────┘                        │   │
│  │                         │  SMC instruction                   │   │
│  └─────────────────────────┼────────────────────────────────────┘   │
│  ═══════════════════ TrustZone boundary ═══════════════════════     │
│  ┌─────────────────────────┼────────────────────────────────────┐   │
│  │                         ▼          SECURE WORLD              │   │
│  │              ┌──────────────────────┐                        │   │
│  │              │  BL31 SMC handler    │  Secure Monitor (EL3)  │   │
│  │              └──────────┬───────────┘                        │   │
│  │                         │  World switch to S-EL1             │   │
│  │              ┌──────────▼───────────┐                        │   │
│  │              │  OP-TEE Core (BL32)  │  Secure Kernel (S-EL1) │   │
│  │              │  ┌────────────────┐  │                        │   │
│  │              │  │  TA Manager    │  │                        │   │
│  │              │  │  Session Mgr   │  │                        │   │
│  │              │  │  Crypto Engine │  │                        │   │
│  │              │  │  Secure Store  │  │                        │   │
│  │              │  └────────────────┘  │                        │   │
│  │              └──────────┬───────────┘                        │   │
│  │                         │  TA context switch                 │   │
│  │              ┌──────────▼───────────┐                        │   │
│  │              │  Trusted Application │  (S-EL0, per session)  │   │
│  │              │  (TA UUID: binary)   │                        │   │
│  │              └──────────────────────┘                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 TEE Supplicant

`tee-supplicant` is a userspace daemon started by the system that handles TA loading
from the normal-world filesystem into the secure world:

```
/usr/lib/tee-supplicant &          # Started by init/systemd
/lib/optee_armtz/                  # Default TA storage directory
    <UUID>.ta                      # Signed & encrypted Trusted Application binary
```

---

## 6. FIT Image Signing

A FIT (Flattened Image Tree) image is a U-Boot construct that bundles kernel,
device tree, and ramdisk with cryptographic signatures, verified by U-Boot before
execution.

### 6.1 FIT Image Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FIT IMAGE (.itb) LAYOUT                          │
│                                                                     │
│  FDT Header (magic: 0xd00dfeed)                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  / (root)                                                    │   │
│  │  ├── description = "Signed Linux FIT"                        │   │
│  │  ├── timestamp   = <unix epoch>                              │   │
│  │  │                                                           │   │
│  │  ├── images/                                                 │   │
│  │  │   ├── kernel@1                                            │   │
│  │  │   │   ├── type        = "kernel"                          │   │
│  │  │   │   ├── arch        = "arm64"                           │   │
│  │  │   │   ├── compression = "gzip"                            │   │
│  │  │   │   ├── data        = <kernel binary blob>              │   │
│  │  │   │   └── hash@1 / signature@1                            │   │
│  │  │   │       ├── algo    = "sha256,rsa4096"                  │   │
│  │  │   │       └── value   = <RSA signature bytes>             │   │
│  │  │   │                                                       │   │
│  │  │   ├── fdt@1          (Device Tree Blob)                   │   │
│  │  │   │   └── signature@1  (signed with same key)             │   │
│  │  │   │                                                       │   │
│  │  │   └── ramdisk@1      (initramfs, optional)                │   │
│  │  │                                                           │   │
│  │  └── configurations/                                         │   │
│  │      └── conf@1                                              │   │
│  │          ├── kernel    = "kernel@1"                          │   │
│  │          ├── fdt      = "fdt@1"                              │   │
│  │          ├── ramdisk  = "ramdisk@1"                          │   │
│  │          └── signature@1                                     │   │
│  │              ├── algo  = "sha256,rsa4096"                    │   │
│  │              ├── key-name-hint = "dev-key"                   │   │
│  │              └── signed-images = "kernel", "fdt"             │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 FIT Image Source (`.its`) Template

```dts
/dts-v1/;
/ {
    description = "Signed Linux Kernel FIT Image";
    #address-cells = <1>;

    images {
        kernel@1 {
            description = "Linux Kernel 6.6";
            data = /incbin/("Image.gz");
            type = "kernel";
            arch = "arm64";
            os = "linux";
            compression = "gzip";
            load = <0x40200000>;
            entry = <0x40200000>;
            hash@1 {
                algo = "sha256";
            };
        };

        fdt@1 {
            description = "RPI4 Device Tree";
            data = /incbin/("bcm2711-rpi-4-b.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash@1 {
                algo = "sha256";
            };
        };
    };

    configurations {
        default = "conf@1";
        conf@1 {
            description = "Boot Linux";
            kernel = "kernel@1";
            fdt = "fdt@1";
            signature@1 {
                algo = "sha256,rsa4096";
                key-name-hint = "dev-key";
                sign-images = "fdt", "kernel";
            };
        };
    };
};
```

### 6.3 Signing Workflow

```bash
# 1. Generate RSA-4096 key pair (done once, offline)
openssl genrsa -out dev-key.pem 4096
openssl req -batch -new -x509 -key dev-key.pem -out dev-key.crt

# 2. Assemble unsigned FIT image
mkimage -f kernel.its kernel.itb

# 3. Sign the FIT image, embed public key into U-Boot DTB
mkimage -F -k ./keys -K u-boot.dtb -r kernel.itb

# 4. Rebuild U-Boot with the updated DTB (contains embedded public key)
# U-Boot reads the embedded key at boot to verify the FIT image

# 5. Buildroot automates steps 2-4 via BR2_UBOOT_SIGN_KEY_PATH
```

### 6.4 Key Embedding Flow

```
 Offline Signing Station                    Device Boot
 ─────────────────────────                  ────────────
 dev-key.pem (PRIVATE)  ──────────────X     (never leaves HSM)
       │
       ▼
 dev-key.crt (PUBLIC) ────────────────►  u-boot.dtb
                                            │
                                            ▼
                                       mkimage embeds
                                       public key node
                                            │
                                            ▼
                                       U-Boot compiled/linked
                                       with signed DTB
                                            │
                                       Flash to device
                                            │
                                            ▼
                                       U-Boot verifies
                                       FIT signature at boot
                                       using embedded pubkey
```

---

## 7. Buildroot Integration

### 7.1 Full Buildroot Config for Secure Boot

```makefile
# ── Target & Toolchain ────────────────────────────────────────────────
BR2_aarch64=y
BR2_TOOLCHAIN_EXTERNAL=y
BR2_TOOLCHAIN_EXTERNAL_ARM_AARCH64=y

# ── ARM Trusted Firmware-A ────────────────────────────────────────────
BR2_TARGET_ARM_TRUSTED_FIRMWARE=y
BR2_TARGET_ARM_TRUSTED_FIRMWARE_CUSTOM_GIT=y
BR2_TARGET_ARM_TRUSTED_FIRMWARE_CUSTOM_REPO_URL="https://git.trustedfirmware.org/TF-A/trusted-firmware-a.git"
BR2_TARGET_ARM_TRUSTED_FIRMWARE_CUSTOM_REPO_VERSION="v2.10"
BR2_TARGET_ARM_TRUSTED_FIRMWARE_PLATFORM="qemu"
BR2_TARGET_ARM_TRUSTED_FIRMWARE_BL31=y
BR2_TARGET_ARM_TRUSTED_FIRMWARE_UBOOT_AS_BL33=y
BR2_TARGET_ARM_TRUSTED_FIRMWARE_ADDITIONAL_VARIABLES="DEBUG=0 PLAT=qemu"

# ── OP-TEE OS (BL32) ─────────────────────────────────────────────────
BR2_PACKAGE_OPTEE_OS=y
BR2_PACKAGE_OPTEE_OS_PLATFORM="vexpress-qemu_virt"

# ── OP-TEE Client (Normal World) ──────────────────────────────────────
BR2_PACKAGE_OPTEE_CLIENT=y
BR2_PACKAGE_OPTEE_EXAMPLES=y

# ── U-Boot with FIT verification ──────────────────────────────────────
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="qemu_arm64"
BR2_TARGET_UBOOT_CONFIG_FRAGMENT_FILES="board/myboard/uboot-fit.config"
BR2_TARGET_UBOOT_SIGN=y
BR2_TARGET_UBOOT_SIGN_KEY_PATH="board/myboard/keys"

# ── Linux Kernel ──────────────────────────────────────────────────────
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_USE_DEFCONFIG=y
BR2_LINUX_KERNEL_DEFCONFIG="defconfig"

# ── FIT image ─────────────────────────────────────────────────────────
BR2_LINUX_KERNEL_UIMAGE=y
BR2_LINUX_KERNEL_UIMAGE_LOADADDR="0x40200000"
BR2_LINUX_KERNEL_FIT_IMAGE=y
BR2_LINUX_KERNEL_FIT_IMAGE_ITS="board/myboard/kernel.its"
```

### 7.2 Post-Build Image Assembly Script

```bash
#!/bin/bash
# board/myboard/post-image.sh

BINARIES_DIR="$1"
BOARD_DIR="$(dirname $0)"

# Assemble and sign FIT image
mkimage -f "${BOARD_DIR}/kernel.its" "${BINARIES_DIR}/kernel.itb"
mkimage -F \
    -k "${BOARD_DIR}/keys" \
    -K "${BINARIES_DIR}/u-boot.dtb" \
    -r "${BINARIES_DIR}/kernel.itb"

# Create final flash image: BL1 + BL2 + BL31 + OP-TEE + U-Boot + kernel.itb
${HOST_DIR}/bin/fiptool create \
    --tb-fw    "${BINARIES_DIR}/bl2.bin" \
    --soc-fw   "${BINARIES_DIR}/bl31.bin" \
    --tos-fw   "${BINARIES_DIR}/tee-pager_v2.bin" \
    --nt-fw    "${BINARIES_DIR}/u-boot.bin" \
    "${BINARIES_DIR}/fip.bin"

echo "FIP image assembled: ${BINARIES_DIR}/fip.bin"
```

### 7.3 FIP (Firmware Image Package) Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FIP.BIN LAYOUT                                   │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │  FIP Header     │  Magic + version + ToC entry count             │
│  ├─────────────────┤                                                │
│  │  ToC Entry[0]   │  UUID: BL2  → offset, size                     │
│  │  ToC Entry[1]   │  UUID: BL31 → offset, size                     │
│  │  ToC Entry[2]   │  UUID: BL32 (OP-TEE) → offset, size            │
│  │  ToC Entry[3]   │  UUID: BL33 (U-Boot) → offset, size            │
│  ├─────────────────┤                                                │
│  │  BL2 binary     │  (verified by BL1 ROTPK)                       │
│  ├─────────────────┤                                                │
│  │  BL31 binary    │  Secure Monitor                                │
│  ├─────────────────┤                                                │
│  │  OP-TEE binary  │  tee-pager_v2.bin + tee-pageable_v2.bin        │
│  ├─────────────────┤                                                │
│  │  U-Boot binary  │  BL33, normal-world loader                     │
│  └─────────────────┘                                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. C Programming Examples

### 8.1 OP-TEE Client Application (CA) — Session Open & Invoke

```c
/* ca_aes_encrypt.c - Normal World Client Application
 * Calls a Trusted Application inside OP-TEE for AES-256-CBC encryption.
 *
 * Build:
 *   aarch64-linux-gnu-gcc ca_aes_encrypt.c -o ca_aes_encrypt \
 *       -I/usr/include/optee/client \
 *       -lteec
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <tee_client_api.h>

/* TA UUID — must match the TA's ta_entry.h */
#define TA_AES_UUID \
    { 0xaabbccdd, 0x1234, 0x5678, \
      { 0xaa, 0xbb, 0xcc, 0xdd, 0x11, 0x22, 0x33, 0x44 } }

/* TA Command IDs */
#define TA_AES_CMD_ENCRYPT   0x00
#define TA_AES_CMD_DECRYPT   0x01

#define AES_KEY_SIZE    32  /* 256-bit */
#define AES_IV_SIZE     16  /* 128-bit */
#define BUFFER_SIZE     64

static void dump_hex(const char *label, const uint8_t *buf, size_t len)
{
    printf("%s: ", label);
    for (size_t i = 0; i < len; i++)
        printf("%02x", buf[i]);
    printf("\n");
}

int main(void)
{
    TEEC_Result        res;
    TEEC_Context       ctx;
    TEEC_Session       sess;
    TEEC_Operation     op;
    TEEC_UUID          uuid = TA_AES_UUID;
    uint32_t           err_origin;

    /* Plaintext, key, IV */
    uint8_t plaintext[BUFFER_SIZE] = "Hello from the Normal World!    ";
    uint8_t key[AES_KEY_SIZE]      = {0};  /* In production: derive via HKDF */
    uint8_t iv[AES_IV_SIZE]        = {0};  /* Random IV in production */
    uint8_t ciphertext[BUFFER_SIZE] = {0};

    /* 1. Initialize TEE context */
    res = TEEC_InitializeContext(NULL, &ctx);
    if (res != TEEC_SUCCESS) {
        fprintf(stderr, "TEEC_InitializeContext: 0x%x\n", res);
        return EXIT_FAILURE;
    }

    /* 2. Open session with the Trusted Application */
    memset(&op, 0, sizeof(op));
    op.paramTypes = TEEC_PARAM_TYPES(
        TEEC_NONE, TEEC_NONE, TEEC_NONE, TEEC_NONE);

    res = TEEC_OpenSession(&ctx, &sess, &uuid,
                           TEEC_LOGIN_PUBLIC, NULL, &op, &err_origin);
    if (res != TEEC_SUCCESS) {
        fprintf(stderr, "TEEC_OpenSession: 0x%x (origin 0x%x)\n",
                res, err_origin);
        TEEC_FinalizeContext(&ctx);
        return EXIT_FAILURE;
    }

    /* 3. Invoke TA: pass key, IV, plaintext; receive ciphertext */
    memset(&op, 0, sizeof(op));
    op.paramTypes = TEEC_PARAM_TYPES(
        TEEC_MEMREF_TEMP_INPUT,   /* param[0]: AES key       */
        TEEC_MEMREF_TEMP_INPUT,   /* param[1]: IV            */
        TEEC_MEMREF_TEMP_INPUT,   /* param[2]: plaintext     */
        TEEC_MEMREF_TEMP_OUTPUT   /* param[3]: ciphertext    */
    );

    op.params[0].tmpref.buffer = key;
    op.params[0].tmpref.size   = AES_KEY_SIZE;

    op.params[1].tmpref.buffer = iv;
    op.params[1].tmpref.size   = AES_IV_SIZE;

    op.params[2].tmpref.buffer = plaintext;
    op.params[2].tmpref.size   = BUFFER_SIZE;

    op.params[3].tmpref.buffer = ciphertext;
    op.params[3].tmpref.size   = BUFFER_SIZE;

    res = TEEC_InvokeCommand(&sess, TA_AES_CMD_ENCRYPT, &op, &err_origin);
    if (res != TEEC_SUCCESS) {
        fprintf(stderr, "TEEC_InvokeCommand: 0x%x (origin 0x%x)\n",
                res, err_origin);
    } else {
        dump_hex("Plaintext ", plaintext,  BUFFER_SIZE);
        dump_hex("Key       ", key,        AES_KEY_SIZE);
        dump_hex("IV        ", iv,         AES_IV_SIZE);
        dump_hex("Ciphertext", ciphertext, BUFFER_SIZE);
    }

    /* 4. Cleanup */
    TEEC_CloseSession(&sess);
    TEEC_FinalizeContext(&ctx);

    return (res == TEEC_SUCCESS) ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

### 8.2 Trusted Application (TA) — Secure World AES Implementation

```c
/* ta_aes_encrypt.c - Trusted Application (runs in OP-TEE / Secure World)
 *
 * Compiled with the OP-TEE TA devkit.
 * Place in: ta/aes_ta/ta_aes_encrypt.c
 */

#include <tee_internal_api.h>
#include <tee_internal_api_extensions.h>
#include <string.h>

#define TA_AES_CMD_ENCRYPT   0x00
#define TA_AES_CMD_DECRYPT   0x01

/* TA lifecycle: called when the TA is loaded */
TEE_Result TA_CreateEntryPoint(void)
{
    DMSG("AES TA created");
    return TEE_SUCCESS;
}

void TA_DestroyEntryPoint(void)
{
    DMSG("AES TA destroyed");
}

/* Called when a new session is opened */
TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types,
                                    TEE_Param params[4],
                                    void **sess_ctx)
{
    (void)param_types;
    (void)params;
    (void)sess_ctx;
    DMSG("AES TA session opened");
    return TEE_SUCCESS;
}

void TA_CloseSessionEntryPoint(void *sess_ctx)
{
    (void)sess_ctx;
    DMSG("AES TA session closed");
}

/* AES-256-CBC encryption using GP TEE Crypto API */
static TEE_Result do_aes_cbc(uint32_t param_types, TEE_Param params[4],
                              TEE_OperationMode mode)
{
    TEE_Result       res;
    TEE_ObjectHandle key_obj  = TEE_HANDLE_NULL;
    TEE_OperationHandle op    = TEE_HANDLE_NULL;
    TEE_Attribute    key_attr;

    uint32_t exp_types = TEE_PARAM_TYPES(
        TEE_PARAM_TYPE_MEMREF_INPUT,    /* key       */
        TEE_PARAM_TYPE_MEMREF_INPUT,    /* IV        */
        TEE_PARAM_TYPE_MEMREF_INPUT,    /* plaintext */
        TEE_PARAM_TYPE_MEMREF_OUTPUT    /* ciphertext */
    );

    if (param_types != exp_types)
        return TEE_ERROR_BAD_PARAMETERS;

    void   *key_buf = params[0].memref.buffer;
    size_t  key_sz  = params[0].memref.size;
    void   *iv_buf  = params[1].memref.buffer;
    size_t  iv_sz   = params[1].memref.size;
    void   *in_buf  = params[2].memref.buffer;
    size_t  in_sz   = params[2].memref.size;
    void   *out_buf = params[3].memref.buffer;
    size_t  out_sz  = params[3].memref.size;

    /* Allocate transient key object */
    res = TEE_AllocateTransientObject(TEE_TYPE_AES, key_sz * 8, &key_obj);
    if (res != TEE_SUCCESS) goto out;

    /* Populate key */
    TEE_InitRefAttribute(&key_attr, TEE_ATTR_SECRET_VALUE, key_buf, key_sz);
    res = TEE_PopulateTransientObject(key_obj, &key_attr, 1);
    if (res != TEE_SUCCESS) goto out;

    /* Allocate cipher operation */
    res = TEE_AllocateOperation(&op, TEE_ALG_AES_CBC_NOPAD, mode,
                                key_sz * 8);
    if (res != TEE_SUCCESS) goto out;

    /* Set key and IV */
    res = TEE_SetOperationKey(op, key_obj);
    if (res != TEE_SUCCESS) goto out;

    TEE_CipherInit(op, iv_buf, iv_sz);

    /* Perform cipher */
    res = TEE_CipherDoFinal(op, in_buf, in_sz, out_buf, &out_sz);

out:
    if (op != TEE_HANDLE_NULL)
        TEE_FreeOperation(op);
    if (key_obj != TEE_HANDLE_NULL)
        TEE_FreeTransientObject(key_obj);

    return res;
}

/* Main TA command dispatcher */
TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx,
                                      uint32_t cmd_id,
                                      uint32_t param_types,
                                      TEE_Param params[4])
{
    (void)sess_ctx;

    switch (cmd_id) {
    case TA_AES_CMD_ENCRYPT:
        return do_aes_cbc(param_types, params, TEE_MODE_ENCRYPT);
    case TA_AES_CMD_DECRYPT:
        return do_aes_cbc(param_types, params, TEE_MODE_DECRYPT);
    default:
        EMSG("Unknown command: 0x%x", cmd_id);
        return TEE_ERROR_NOT_SUPPORTED;
    }
}
```

### 8.3 Reading OP-TEE Secure Storage (Persistent Keys)

```c
/* ta_secure_storage.c — Snippet: store/retrieve a secret in TEE secure storage */

#include <tee_internal_api.h>

#define OBJ_ID      "master-key-v1"
#define OBJ_ID_LEN  (sizeof(OBJ_ID) - 1)
#define KEY_SIZE    32

/* Write a secret key to secure (encrypted, hardware-bound) storage */
TEE_Result ta_write_secret_key(const uint8_t *key, size_t key_len)
{
    TEE_Result          res;
    TEE_ObjectHandle    obj = TEE_HANDLE_NULL;
    uint32_t            flags;

    flags = TEE_DATA_FLAG_ACCESS_READ  |
            TEE_DATA_FLAG_ACCESS_WRITE |
            TEE_DATA_FLAG_OVERWRITE;

    res = TEE_CreatePersistentObject(
        TEE_STORAGE_PRIVATE,   /* TEE-private encrypted storage */
        OBJ_ID, OBJ_ID_LEN,
        flags,
        TEE_HANDLE_NULL,
        key, key_len,
        &obj
    );

    if (obj != TEE_HANDLE_NULL)
        TEE_CloseObject(obj);

    return res;
}

/* Read the secret key back from secure storage */
TEE_Result ta_read_secret_key(uint8_t *out, size_t out_len)
{
    TEE_Result          res;
    TEE_ObjectHandle    obj = TEE_HANDLE_NULL;
    uint32_t            bytes_read;
    uint32_t            flags;

    flags = TEE_DATA_FLAG_ACCESS_READ |
            TEE_DATA_FLAG_SHARE_READ;

    res = TEE_OpenPersistentObject(
        TEE_STORAGE_PRIVATE,
        OBJ_ID, OBJ_ID_LEN,
        flags,
        &obj
    );
    if (res != TEE_SUCCESS) return res;

    res = TEE_ReadObjectData(obj, out, out_len, &bytes_read);
    TEE_CloseObject(obj);

    return res;
}
```

### 8.4 TF-A Platform SMC Handler Stub (C, BL31)

```c
/* plat/myplat/myplat_smc.c
 * Custom SMC handler registered in BL31 for platform-specific services.
 */

#include <runtime_svc.h>
#include <smccc.h>
#include <stdint.h>

/* SiP (Silicon Provider) SMC range: 0xC2000000 - 0xC200FFFF */
#define MYPLAT_SIP_FUNC_GET_VERSION     0xC2000001
#define MYPLAT_SIP_FUNC_SET_GPIO_SECURE 0xC2000002

static uintptr_t myplat_sip_handler(uint32_t smc_fid,
                                    u_register_t x1,
                                    u_register_t x2,
                                    u_register_t x3,
                                    u_register_t x4,
                                    void *cookie,
                                    void *handle,
                                    u_register_t flags)
{
    switch (smc_fid) {
    case MYPLAT_SIP_FUNC_GET_VERSION:
        /* Return platform firmware version in x0 */
        SMC_RET1(handle, 0x00010000U);  /* v1.0 */

    case MYPLAT_SIP_FUNC_SET_GPIO_SECURE:
        /*
         * x1 = GPIO bank
         * x2 = GPIO pin mask
         * Mark GPIO pins as secure-world-only via TZPC
         */
        /* myplat_tzpc_set_secure(x1, x2); */
        SMC_RET1(handle, SMC_OK);

    default:
        WARN("Unimplemented SiP SMC: 0x%x\n", smc_fid);
        SMC_RET1(handle, SMC_UNK);
    }
}

/* Register the SiP service */
DECLARE_RT_SVC(
    myplat_sip_svc,
    OEN_SIP_START,          /* Owner: Silicon Provider */
    OEN_SIP_END,
    SMC_TYPE_FAST,
    NULL,                   /* setup function */
    myplat_sip_handler
);
```

---

## 9. C++ Programming Examples

### 9.1 RAII TEE Session Wrapper

```cpp
// tee_session.hpp — Modern C++ RAII wrapper around the TEE Client API

#pragma once
#include <tee_client_api.h>
#include <stdexcept>
#include <cstdint>
#include <string>
#include <span>
#include <vector>

namespace tee {

class Error : public std::runtime_error {
public:
    explicit Error(const std::string &msg, TEEC_Result code)
        : std::runtime_error(msg + " (0x" + std::to_string(code) + ")")
        , code_(code) {}
    TEEC_Result code() const { return code_; }
private:
    TEEC_Result code_;
};

/* ──── Context ──────────────────────────────────────────────────────── */
class Context {
public:
    Context() {
        TEEC_Result res = TEEC_InitializeContext(nullptr, &ctx_);
        if (res != TEEC_SUCCESS)
            throw Error("TEEC_InitializeContext failed", res);
    }
    ~Context() { TEEC_FinalizeContext(&ctx_); }

    Context(const Context &) = delete;
    Context &operator=(const Context &) = delete;

    TEEC_Context *native() { return &ctx_; }

private:
    TEEC_Context ctx_{};
};

/* ──── Session ──────────────────────────────────────────────────────── */
class Session {
public:
    Session(Context &ctx, const TEEC_UUID &uuid) {
        TEEC_Operation op{};
        op.paramTypes = TEEC_PARAM_TYPES(
            TEEC_NONE, TEEC_NONE, TEEC_NONE, TEEC_NONE);

        uint32_t origin = 0;
        TEEC_Result res = TEEC_OpenSession(
            ctx.native(), &sess_, &uuid,
            TEEC_LOGIN_PUBLIC, nullptr, &op, &origin);

        if (res != TEEC_SUCCESS)
            throw Error("TEEC_OpenSession failed", res);
    }

    ~Session() { TEEC_CloseSession(&sess_); }

    Session(const Session &) = delete;
    Session &operator=(const Session &) = delete;

    /* Invoke a TA command with typed parameters */
    TEEC_Result invoke(uint32_t cmd_id, TEEC_Operation &op) {
        uint32_t origin = 0;
        TEEC_Result res = TEEC_InvokeCommand(&sess_, cmd_id, &op, &origin);
        return res;
    }

    /* Convenience: encrypt a buffer using the AES TA */
    std::vector<uint8_t> aes_encrypt(
        std::span<const uint8_t> key,
        std::span<const uint8_t> iv,
        std::span<const uint8_t> plaintext)
    {
        std::vector<uint8_t> ciphertext(plaintext.size());

        TEEC_Operation op{};
        op.paramTypes = TEEC_PARAM_TYPES(
            TEEC_MEMREF_TEMP_INPUT,
            TEEC_MEMREF_TEMP_INPUT,
            TEEC_MEMREF_TEMP_INPUT,
            TEEC_MEMREF_TEMP_OUTPUT);

        op.params[0].tmpref = {const_cast<uint8_t*>(key.data()),  key.size()};
        op.params[1].tmpref = {const_cast<uint8_t*>(iv.data()),   iv.size()};
        op.params[2].tmpref = {const_cast<uint8_t*>(plaintext.data()), plaintext.size()};
        op.params[3].tmpref = {ciphertext.data(), ciphertext.size()};

        TEEC_Result res = invoke(0x00 /* TA_AES_CMD_ENCRYPT */, op);
        if (res != TEEC_SUCCESS)
            throw Error("AES encrypt invoke failed", res);

        ciphertext.resize(op.params[3].tmpref.size);
        return ciphertext;
    }

private:
    TEEC_Session sess_{};
};

} // namespace tee
```

### 9.2 Secure Key Provisioning Manager (C++)

```cpp
// key_provisioner.cpp — Manages device key provisioning via TEE
// Uses C++20 features: std::span, std::format

#include "tee_session.hpp"
#include <iostream>
#include <array>
#include <algorithm>
#include <iomanip>
#include <format>

/* TA UUID for the key provisioning TA */
static constexpr TEEC_UUID PROV_TA_UUID = {
    0xdeadbeef, 0xcafe, 0xf00d,
    {0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08}
};

enum class ProvCmd : uint32_t {
    GenerateKey    = 0x10,
    ImportKey      = 0x11,
    DeleteKey      = 0x12,
    GetPublicKey   = 0x13,
    SignData       = 0x14,
};

class KeyProvisioner {
public:
    KeyProvisioner()
        : ctx_()
        , sess_(ctx_, PROV_TA_UUID)
    {}

    /* Generate a new ECC P-256 device identity key inside the TEE */
    bool generate_identity_key(uint32_t key_slot)
    {
        TEEC_Operation op{};
        op.paramTypes = TEEC_PARAM_TYPES(
            TEEC_VALUE_INPUT, TEEC_NONE, TEEC_NONE, TEEC_NONE);
        op.params[0].value.a = key_slot;

        try {
            auto res = sess_.invoke(
                static_cast<uint32_t>(ProvCmd::GenerateKey), op);
            if (res != TEEC_SUCCESS) {
                std::cerr << std::format("GenerateKey failed: 0x{:08x}\n", res);
                return false;
            }
        } catch (const tee::Error &e) {
            std::cerr << "TEE error: " << e.what() << "\n";
            return false;
        }

        std::cout << std::format(
            "[KeyProvisioner] ECC P-256 key generated in slot {}\n", key_slot);
        return true;
    }

    /* Retrieve the public key from a key slot */
    std::vector<uint8_t> get_public_key(uint32_t key_slot)
    {
        std::vector<uint8_t> pubkey(65);  /* Uncompressed P-256: 0x04 + 32 + 32 */

        TEEC_Operation op{};
        op.paramTypes = TEEC_PARAM_TYPES(
            TEEC_VALUE_INPUT,
            TEEC_MEMREF_TEMP_OUTPUT,
            TEEC_NONE, TEEC_NONE);
        op.params[0].value.a      = key_slot;
        op.params[1].tmpref.buffer = pubkey.data();
        op.params[1].tmpref.size   = pubkey.size();

        auto res = sess_.invoke(
            static_cast<uint32_t>(ProvCmd::GetPublicKey), op);
        if (res != TEEC_SUCCESS)
            throw tee::Error("GetPublicKey failed", res);

        pubkey.resize(op.params[1].tmpref.size);
        return pubkey;
    }

    /* Sign arbitrary data using the device identity key */
    std::vector<uint8_t> sign(uint32_t key_slot,
                               std::span<const uint8_t> data)
    {
        std::vector<uint8_t> signature(72);  /* DER-encoded ECDSA max */

        TEEC_Operation op{};
        op.paramTypes = TEEC_PARAM_TYPES(
            TEEC_VALUE_INPUT,
            TEEC_MEMREF_TEMP_INPUT,
            TEEC_MEMREF_TEMP_OUTPUT,
            TEEC_NONE);
        op.params[0].value.a       = key_slot;
        op.params[1].tmpref.buffer = const_cast<uint8_t*>(data.data());
        op.params[1].tmpref.size   = data.size();
        op.params[2].tmpref.buffer = signature.data();
        op.params[2].tmpref.size   = signature.size();

        auto res = sess_.invoke(
            static_cast<uint32_t>(ProvCmd::SignData), op);
        if (res != TEEC_SUCCESS)
            throw tee::Error("SignData failed", res);

        signature.resize(op.params[2].tmpref.size);
        return signature;
    }

private:
    tee::Context ctx_;
    tee::Session sess_;
};

int main()
{
    try {
        KeyProvisioner kp;

        constexpr uint32_t SLOT_ID = 0;

        /* Step 1: Generate a new identity key */
        if (!kp.generate_identity_key(SLOT_ID))
            return EXIT_FAILURE;

        /* Step 2: Retrieve public key */
        auto pubkey = kp.get_public_key(SLOT_ID);
        std::cout << "Public key (" << pubkey.size() << " bytes): ";
        for (auto b : pubkey)
            std::cout << std::hex << std::setw(2) << std::setfill('0')
                      << static_cast<int>(b);
        std::cout << "\n";

        /* Step 3: Sign a challenge */
        std::array<uint8_t, 32> challenge = {0xDE, 0xAD};
        auto sig = kp.sign(SLOT_ID, std::span{challenge});
        std::cout << "Signature (" << sig.size() << " bytes) obtained.\n";

    } catch (const tee::Error &e) {
        std::cerr << "Fatal TEE error: " << e.what() << "\n";
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

---

## 10. Rust Programming Examples

### 10.1 Cargo Project Setup for OP-TEE Client

```toml
# Cargo.toml
[package]
name    = "optee-client-rs"
version = "0.1.0"
edition = "2021"

[dependencies]
optee-teec  = "0.3"        # Rust bindings for libteec (GlobalPlatform TEE Client API)
uuid        = "1"
thiserror   = "1"
anyhow      = "1"
log         = "0.4"
env_logger  = "0.11"
zeroize     = "1"          # Securely zero key material on drop

[build-dependencies]
pkg-config  = "0.3"
```

### 10.2 Rust TEE Client with Zeroizing Key Material

```rust
// src/main.rs — Rust OP-TEE Client Application
//
// Demonstrates:
//   - Connecting to an OP-TEE Trusted Application
//   - RAII session management with automatic cleanup
//   - Zero-on-drop key buffers using `zeroize`

use anyhow::{Context, Result};
use optee_teec::{Context as TeecContext, Operation, ParamType, Session, Uuid};
use zeroize::Zeroizing;

/// TA UUID — must match the TA's header exactly.
const TA_AES_UUID: &str = "aabbccdd-1234-5678-aabb-ccdd11223344";

/// TA command IDs
const CMD_ENCRYPT: u32 = 0x00;
const CMD_DECRYPT: u32 = 0x01;

const AES_KEY_SIZE: usize = 32;
const AES_IV_SIZE:  usize = 16;
const BUF_SIZE:     usize = 64;

/// Wrapper that ensures the TEE session is closed when dropped.
struct AesSession {
    ctx:  TeecContext,
    sess: Session,
}

impl AesSession {
    fn new() -> Result<Self> {
        let mut ctx  = TeecContext::new()?;
        let uuid     = Uuid::parse_str(TA_AES_UUID)
            .context("Invalid TA UUID")?;
        let sess     = ctx.open_session(uuid)
            .context("Failed to open TEE session")?;

        Ok(Self { ctx, sess })
    }

    /// Encrypt plaintext with AES-256-CBC.
    /// Key is held in a `Zeroizing<Vec<u8>>` — memory is wiped on drop.
    fn encrypt(
        &mut self,
        key:       &Zeroizing<Vec<u8>>,
        iv:        &[u8; AES_IV_SIZE],
        plaintext: &[u8],
    ) -> Result<Vec<u8>> {
        assert_eq!(key.len(), AES_KEY_SIZE, "Key must be 32 bytes");

        let mut ciphertext = vec![0u8; plaintext.len()];

        let mut op = Operation::new(
            0,
            ParamType::MemrefTempInput,   // p[0]: key
            ParamType::MemrefTempInput,   // p[1]: iv
            ParamType::MemrefTempInput,   // p[2]: plaintext
            ParamType::MemrefTempOutput,  // p[3]: ciphertext
        );

        op.params.0.set_tmpref(key.as_slice());
        op.params.1.set_tmpref(iv.as_slice());
        op.params.2.set_tmpref(plaintext);
        op.params.3.set_tmpref(&mut ciphertext);

        self.sess
            .invoke_command(CMD_ENCRYPT, &mut op)
            .context("AES encrypt command failed")?;

        Ok(ciphertext)
    }

    fn decrypt(
        &mut self,
        key:        &Zeroizing<Vec<u8>>,
        iv:         &[u8; AES_IV_SIZE],
        ciphertext: &[u8],
    ) -> Result<Vec<u8>> {
        let mut plaintext = vec![0u8; ciphertext.len()];

        let mut op = Operation::new(
            0,
            ParamType::MemrefTempInput,
            ParamType::MemrefTempInput,
            ParamType::MemrefTempInput,
            ParamType::MemrefTempOutput,
        );

        op.params.0.set_tmpref(key.as_slice());
        op.params.1.set_tmpref(iv.as_slice());
        op.params.2.set_tmpref(ciphertext);
        op.params.3.set_tmpref(&mut plaintext);

        self.sess
            .invoke_command(CMD_DECRYPT, &mut op)
            .context("AES decrypt command failed")?;

        Ok(plaintext)
    }
}

fn main() -> Result<()> {
    env_logger::init();

    // Key is wrapped in Zeroizing: bytes are wiped when `key` drops.
    let key: Zeroizing<Vec<u8>> = Zeroizing::new(vec![0x42u8; AES_KEY_SIZE]);
    let iv:  [u8; AES_IV_SIZE]  = [0xABu8; AES_IV_SIZE];
    let msg  = b"Hello Secure World! (Rust)      ";

    let mut session = AesSession::new()
        .context("Cannot connect to TEE")?;

    let ciphertext = session.encrypt(&key, &iv, msg)?;
    println!("Encrypted: {:02x?}", &ciphertext);

    let recovered = session.decrypt(&key, &iv, &ciphertext)?;
    println!("Decrypted: {}", String::from_utf8_lossy(&recovered));

    assert_eq!(&recovered[..msg.len()], msg, "Round-trip mismatch!");
    println!("Round-trip OK ✓");

    // `key` is zeroed here automatically (Zeroizing Drop impl).
    Ok(())
}
// When `session` goes out of scope, Session::drop() calls TEEC_CloseSession
// and Context::drop() calls TEEC_FinalizeContext — no leaks possible.
```

### 10.3 Rust: FIT Image Verification Tool (Host-Side)

```rust
// src/fit_verify.rs — Verify a FIT image signature using OpenSSL bindings
//
// [dependencies]
// openssl   = { version = "0.10", features = ["v102", "v110"] }
// fdt       = "0.3"
// anyhow    = "1"
// hex       = "0.4"

use anyhow::{bail, Context, Result};
use openssl::{
    hash::MessageDigest,
    pkey::{PKey, Public},
    rsa::Rsa,
    sign::Verifier,
    x509::X509,
};
use std::{fs, path::Path};

/// Minimal FIT node representation
struct FitImage {
    raw: Vec<u8>,
}

impl FitImage {
    fn load(path: &Path) -> Result<Self> {
        let raw = fs::read(path)
            .with_context(|| format!("Cannot read FIT image: {}", path.display()))?;
        // Verify FDT magic: 0xd00dfeed
        if raw.len() < 4 || u32::from_be_bytes(raw[0..4].try_into()?) != 0xd00dfeed {
            bail!("Not a valid FDT/FIT image (bad magic)");
        }
        Ok(Self { raw })
    }

    /// Extract a named property bytes from a path (simplified, no full FDT parse)
    fn get_property(&self, _node_path: &str, _prop_name: &str) -> Option<&[u8]> {
        // In production: use `fdt` crate to properly traverse FDT structure.
        // Simplified placeholder:
        Some(&self.raw[128..256])
    }
}

/// Load an RSA public key from a PEM certificate
fn load_public_key(cert_path: &Path) -> Result<PKey<Public>> {
    let pem  = fs::read(cert_path)
        .with_context(|| format!("Cannot read cert: {}", cert_path.display()))?;
    let cert = X509::from_pem(&pem)
        .context("Failed to parse X509 certificate")?;
    let pkey = cert.public_key()
        .context("Failed to extract public key from cert")?;
    Ok(pkey)
}

/// Verify RSA-SHA256 signature over data
fn verify_rsa_sha256(
    pkey:      &PKey<Public>,
    data:      &[u8],
    signature: &[u8],
) -> Result<bool> {
    let mut verifier = Verifier::new(MessageDigest::sha256(), pkey)
        .context("Failed to create verifier")?;
    verifier.update(data)
        .context("Verifier update failed")?;
    verifier.verify(signature)
        .context("Verifier verify call failed")
}

fn main() -> Result<()> {
    let fit_path  = Path::new("kernel.itb");
    let cert_path = Path::new("dev-key.crt");

    println!("Loading FIT image: {}", fit_path.display());
    let fit = FitImage::load(fit_path)?;

    println!("Loading public key: {}", cert_path.display());
    let pkey = load_public_key(cert_path)?;

    // Extract kernel data and its signature from FIT
    let kernel_data = fit
        .get_property("/images/kernel@1", "data")
        .context("No kernel data in FIT")?;
    let signature = fit
        .get_property("/images/kernel@1/signature@1", "value")
        .context("No kernel signature in FIT")?;

    match verify_rsa_sha256(&pkey, kernel_data, signature)? {
        true  => println!("✓  FIT kernel signature VALID"),
        false => {
            eprintln!("✗  FIT kernel signature INVALID — refusing to boot");
            std::process::exit(1);
        }
    }

    Ok(())
}
```

### 10.4 Rust: Secure Boot State Inspector

```rust
// src/sb_inspector.rs — Read Secure Boot state from sysfs / OP-TEE
//
// Reads kernel's TEE subsystem sysfs nodes and reports device security posture.

use std::{fs, path::Path};

#[derive(Debug)]
struct SecureBootStatus {
    tee_present:       bool,
    tee_type:          String,
    tee_impl:          String,
    optee_version:     Option<String>,
    secure_boot_fused: bool,
    jtag_disabled:     bool,
}

fn read_sysfs(path: &str) -> Option<String> {
    fs::read_to_string(path)
        .ok()
        .map(|s| s.trim().to_string())
}

fn inspect_secure_boot() -> SecureBootStatus {
    let tee_present = Path::new("/sys/class/tee").exists();

    let tee_type = read_sysfs("/sys/class/tee/tee0/tee_version")
        .unwrap_or_else(|| "unknown".to_string());

    let tee_impl = read_sysfs("/sys/class/tee/tee0/implementation_id")
        .unwrap_or_else(|| "unknown".to_string());

    // OP-TEE version reported via ioctl TEE_IOC_VERSION (simplified)
    let optee_version = read_sysfs("/sys/firmware/optee/tee_api_revision");

    // Platform-specific: check OTP/eFUSE via sysfs or device driver
    let secure_boot_fused = Path::new("/sys/firmware/devicetree/base/secure-boot")
        .exists();

    let jtag_disabled = Path::new("/sys/firmware/devicetree/base/jtag-disabled")
        .exists();

    SecureBootStatus {
        tee_present,
        tee_type,
        tee_impl,
        optee_version,
        secure_boot_fused,
        jtag_disabled,
    }
}

fn print_status(s: &SecureBootStatus) {
    println!("╔══════════════════════════════════════════════╗");
    println!("║         SECURE BOOT STATUS REPORT           ║");
    println!("╠══════════════════════════════════════════════╣");
    println!("║  TEE Present        : {:5}                  ║",
        if s.tee_present { "YES ✓" } else { "NO  ✗" });
    println!("║  TEE Type           : {:<24}║", s.tee_type);
    println!("║  TEE Impl           : {:<24}║", s.tee_impl);
    println!("║  OP-TEE Version     : {:<24}║",
        s.optee_version.as_deref().unwrap_or("n/a"));
    println!("║  Secure Boot Fused  : {:5}                  ║",
        if s.secure_boot_fused { "YES ✓" } else { "NO  ✗" });
    println!("║  JTAG Disabled      : {:5}                  ║",
        if s.jtag_disabled { "YES ✓" } else { "NO  ✗" });
    println!("╚══════════════════════════════════════════════╝");

    if !s.secure_boot_fused {
        println!("  ⚠  WARNING: Device is NOT production-fused!");
        println!("     Secure boot is NOT enforced by hardware.");
    }
}

fn main() {
    let status = inspect_secure_boot();
    print_status(&status);
}
```

---

## 11. Key Management & PKI

### 11.1 Trust Chain

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                         TRUST CHAIN / PKI HIERARCHY                      │
  │                                                                          │
  │   ┌──────────────────────────────────┐                                   │
  │   │   Root of Trust (RoT)            │  ← Burned into OTP/eFUSE at MFG   │
  │   │   ROTPK (Root-of-Trust Pub Key)  │    (SHA256 hash of OEM pub key)   │
  │   └────────────────┬─────────────────┘                                   │
  │                    │  Signs                                              │
  │                    ▼                                                     │
  │   ┌──────────────────────────────────┐                                   │
  │   │   BL2 Content Certificate        │  ROT private key (offline/HSM)    │
  │   │   Trusted Key Certificate        │                                   │
  │   └──────┬──────────────────┬────────┘                                   │
  │          │  Signs           │  Signs                                     │
  │          ▼                  ▼                                            │
  │   ┌─────────────┐    ┌──────────────┐                                    │
  │   │ BL31 Cert   │    │ BL32 Cert    │   SoC Firmware Key (intermediate)  │
  │   │ (TF-A Mon.) │    │ (OP-TEE)     │                                    │
  │   └─────────────┘    └──────────────┘                                    │
  │           │                  │                                           │
  │           ▼                  ▼                                           │
  │   ┌─────────────┐    ┌──────────────┐                                    │
  │   │  BL31 Image │    │  BL32 Image  │                                    │
  │   │  (binary)   │    │  (binary)    │                                    │
  │   └─────────────┘    └──────────────┘                                    │
  │                                                                          │
  │   ┌──────────────────────────────────┐                                   │
  │   │   Non-Trusted Firmware Cert      │   Non-Trusted World Key           │
  │   └────────────────┬─────────────────┘                                   │
  │                    │  Signs                                              │
  │                    ▼                                                     │
  │   ┌──────────────────────────────────┐                                   │
  │   │   BL33 (U-Boot) Image Cert       │                                   │
  │   └──────────────────────────────────┘                                   │
  │                    │  Signs (via mkimage)                                │
  │                    ▼                                                     │
  │   ┌──────────────────────────────────┐                                   │
  │   │   FIT Image (kernel + DTB)       │   Development/Release Key         │
  │   └──────────────────────────────────┘                                   │
  └──────────────────────────────────────────────────────────────────────────┘

  HSM = Hardware Security Module (stores private keys offline)
  OTP = One-Time Programmable memory
```

### 11.2 Key Generation & Certificate Signing

```bash
# 1. Generate OEM Root Key (store ONLY in HSM, never on build server)
openssl ecparam -name prime256v1 -genkey -noout -out oem-root.key.pem
openssl req -new -x509 -key oem-root.key.pem -out oem-root.crt.pem \
    -subj "/CN=OEM Root CA/O=MyCompany/C=DE" -days 7300

# 2. Generate Intermediate SoC Firmware Key
openssl ecparam -name prime256v1 -genkey -noout -out soc-fw.key.pem
openssl req -new -key soc-fw.key.pem -out soc-fw.csr \
    -subj "/CN=SoC Firmware Key/O=MyCompany"
openssl x509 -req -in soc-fw.csr -CA oem-root.crt.pem \
    -CAkey oem-root.key.pem -CAcreateserial \
    -out soc-fw.crt.pem -days 3650

# 3. Generate FIT image signing key (RSA-4096 for U-Boot compatibility)
openssl genrsa -out fit-sign.key.pem 4096
openssl req -new -x509 -key fit-sign.key.pem -out fit-sign.crt.pem \
    -subj "/CN=FIT Image Signing/O=MyCompany"

# 4. Sign FIT image
mkimage -F -k ./keys/ -K output/images/u-boot.dtb -r output/images/kernel.itb

# 5. Burn ROTPK hash to eFUSE (platform-specific tool, irreversible!)
# sha256sum oem-root.crt.pem | cut -c1-64 > rotpk.hash
# platform-fuse-tool --write-rotpk rotpk.hash   # ⚠ ONE-TIME OPERATION
```

---

## 12. Debugging & Attestation

### 12.1 QEMU Test Environment

```bash
# Launch full Secure Boot chain in QEMU (no hardware required)
qemu-system-aarch64 \
    -M virt,secure=on \
    -cpu cortex-a57 \
    -m 1024 \
    -nographic \
    -bios output/images/bl1.bin \
    -device loader,file=output/images/fip.bin,addr=0x08000000 \
    -device loader,file=output/images/kernel.itb,addr=0x40200000 \
    -drive if=none,file=output/images/rootfs.ext2,id=drive0 \
    -device virtio-blk-pci,drive=drive0

# Expected boot log:
# NOTICE:  BL1: Non-Secure code at 0x...
# NOTICE:  BL2: Loading BL31
# NOTICE:  BL31: Secure world ...
# I/TC: OP-TEE version 4.x.y
# U-Boot 2024.x ...
# Verifying Hash Integrity ... OK
# ## Verified configuration: conf@1
```

### 12.2 OP-TEE Log Levels

```
# In Buildroot config or OP-TEE build vars:
BR2_PACKAGE_OPTEE_OS_ADDITIONAL_VARIABLES="CFG_TEE_CORE_LOG_LEVEL=3"

# Log levels:
#  0 = No log
#  1 = Error only
#  2 = Error + Info
#  3 = Error + Info + Debug    ← recommended for development
#  4 = Error + Info + Debug + Flow (very verbose)
```

### 12.3 Remote Attestation Flow (ASCII)

```
  Device (Prover)                          Verifier (Server)
  ───────────────                          ─────────────────
  Boot: TF-A + OP-TEE                            │
  └─► PCR measurements logged                    │
      in OP-TEE secure storage                   │
              │                                  │
              │  ← Challenge nonce ──────────────┤
              │                                  │
  OP-TEE TA:                                     │
  1. Collect measurements                        │
     (BL1 hash, BL2 hash,                        │
      kernel hash, config)                       │
  2. Sign with device identity key               │
     (ECC P-256, stored in TEE)                  │
  3. Build attestation report:                   │
     { nonce, measurements, signature }          │
              │                                  │
              ├─── Attestation Report ──────────►│
              │                                  │
              │                         Verify:  │
              │                         1. Sig with device cert
              │                         2. Cert chain → OEM Root CA
              │                         3. Measurements vs golden
              │                         4. Nonce freshness
              │                                  │
              │  ← Access Token / ACK ───────────┤
```

---

## 13. Security Hardening Checklist

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECURE BOOT HARDENING CHECKLIST                          │
├──────────────────────────────────────────────────────┬──────────────────────┤
│  Item                                                │  Status              │
├──────────────────────────────────────────────────────┼──────────────────────┤
│  ROTPK burned to OTP eFUSE                           │  [ ] TODO / [x] OK   │
│  Secure boot fuse blown (prevents bypass)            │  [ ] TODO / [x] OK   │
│  JTAG disabled in production fuse                    │  [ ] TODO / [x] OK   │
│  BL2 signed with ROT private key                     │  [ ] TODO / [x] OK   │
│  BL31, BL32, BL33 all signed and verified            │  [ ] TODO / [x] OK   │
│  FIT image signed (kernel + DTB)                     │  [ ] TODO / [x] OK   │
│  U-Boot has NO interactive console in production     │  [ ] TODO / [x] OK   │
│  U-Boot CONFIG_BOOTCOMMAND is locked (SHA-signed)    │  [ ] TODO / [x] OK   │
│  OP-TEE CFG_RPMB_FS=y (eMMC replay-protected store)  │  [ ] TODO / [x] OK   │
│  OP-TEE CFG_CORE_ASLR=y                              │  [ ] TODO / [x] OK   │
│  Linux kernel: CONFIG_INTEGRITY_PLATFORM_KEYRING=y   │  [ ] TODO / [x] OK   │
│  Linux kernel: IMA/EVM enabled                       │  [ ] TODO / [x] OK   │
│  Linux kernel: dm-verity on rootfs                   │  [ ] TODO / [x] OK   │
│  Secure storage uses RPMB (hardware replay protect)  │  [ ] TODO / [x] OK   │
│  Private keys NEVER leave HSM                        │  [ ] TODO / [x] OK   │
│  Build server has no access to private keys          │  [ ] TODO / [x] OK   │
│  Key rotation procedure documented & tested          │  [ ] TODO / [x] OK   │
│  Anti-rollback counters enforced in eFUSE            │  [ ] TODO / [x] OK   │
└──────────────────────────────────────────────────────┴──────────────────────┘
```

---

## 14. Summary

Secure Boot and Trusted Firmware form the security backbone of production ARM
embedded Linux systems built with Buildroot. The key concepts and their
relationships are:

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                       CHAPTER 14 — SUMMARY MAP                          │
  │                                                                         │
  │  ┌───────────────┐   fuses →  ┌──────────────┐   signs →  ┌─────────┐   │
  │  │  OEM Root CA  │            │ OTP / eFUSE  │            │  BL2    │   │
  │  │ (offline HSM) │ ──────────►│  ROTPK hash  │ ─────────► │ binary  │   │
  │  └───────────────┘            └──────────────┘            └────┬────┘   │
  │                                                                │        │
  │                                                         loads & verifies│
  │                                                                │        │
  │                                                                ▼        │
  │  ┌───────────────────────────────────────────────────────────────────┐  │
  │  │                  FIP.BIN  (Firmware Image Package)                │  │
  │  │  BL31 (TF-A Mon.) │ BL32 (OP-TEE) │ BL33 (U-Boot)                 │  │
  │  │  all signed with SoC Firmware Key / Non-Trusted World Key         │  │
  │  └────────────────────────────────────┬──────────────────────────────┘  │
  │                                       │                                 │
  │              ┌────────────────────────┴──────────────────────┐          │
  │              │                                               │          │
  │              ▼                                               ▼          │
  │  ┌──────────────────────────┐               ┌──────────────────────┐    │
  │  │  OP-TEE (Secure World)   │               │  U-Boot (BL33)       │    │
  │  │  • Trusted Applications  │               │  • Verifies FIT ITB  │    │
  │  │  • Secure Storage(RPMB)  │               │  • Boots Linux       │    │
  │  │  • Crypto Services       │               └──────────┬───────────┘    │
  │  │  • Key Management        │                          │                │
  │  └──────────────────────────┘                          ▼                │
  │              ▲                               ┌─────────────────────┐    │
  │              │ SMC / TEE Client API          │  Linux + OP-TEE drv │    │
  │              └───────────────────────────────│  + optee-client     │    │
  │                                              │  + Your CA/TA apps  │    │
  │                                              └─────────────────────┘    │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

**TF-A (arm-trusted-firmware)** provides BL1/BL2/BL31 — the Secure Monitor that
lives at EL3 for the lifetime of the system. It mediates all world switches and
PSCI power management calls. Buildroot's `arm-trusted-firmware` package compiles
and integrates it with minimal configuration.

**OP-TEE (optee-os + optee-client)** provides a full GlobalPlatform-compliant TEE
with: a secure kernel (BL32, S-EL1), Trusted Application sandboxes (S-EL0),
hardware-backed secure storage via RPMB, and a Linux-side client library
(`libteec`) plus kernel driver (`tee.ko`).

**FIT Image Signing** closes the chain in the Normal World: U-Boot reads an
RSA public key embedded in its own DTB (placed there during the build) and refuses
to boot any kernel/DTB combination whose SHA-256 hash signature fails to verify.

**In C**: TEE Client API and TA development use the GlobalPlatform C API directly
— `TEEC_OpenSession`, `TEEC_InvokeCommand`, and the `TEE_*` internal API in TAs.

**In C++**: RAII wrappers prevent resource leaks across complex session lifecycles.
`std::span` and `std::format` (C++20) make buffer handling and logging clean and safe.

**In Rust**: The `optee-teec` crate provides safe bindings with zero-cost
abstractions. `Zeroizing<Vec<u8>>` ensures cryptographic key material is scrubbed
from memory on drop — something C requires manual discipline to achieve.

The complete Buildroot integration requires coordinating four packages
(`arm-trusted-firmware`, `optee-os`, `optee-client`, `uboot`) and a post-image
script that assembles the FIP bundle and signs the FIT image, all gated on a
single private key that must never leave an HSM in production deployments.

---

*End of Chapter 14 — Secure Boot & Trusted Firmware (TF-A / OP-TEE)*

---

> **References**
> - [TF-A Documentation](https://trustedfirmware-a.readthedocs.io/)
> - [OP-TEE Documentation](https://optee.readthedocs.io/)
> - [GlobalPlatform TEE Client API Specification v1.0](https://globalplatform.org/specs-library/)
> - [U-Boot FIT Signature Verification](https://docs.u-boot.org/en/latest/usage/fit/signature.html)
> - [Buildroot Manual — Chapter: Bootloaders](https://buildroot.org/downloads/manual/manual.html)
> - [ARM Architecture Reference Manual — ARMv8-A](https://developer.arm.com/documentation/)