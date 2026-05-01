# Buildroot Book

## Core Architecture & Build System

[01. **Buildroot Architecture & Build Flow**](docs/01_Buildroot_Architecture_And_Build_Flow.md)<br>
How Buildroot orchestrates downloads, extraction, patching, configuration, compilation and installation in stages; understanding `make` internals, `output/` directory layout and `BR2_*` variables.

[02. **Kconfig System & menuconfig**](docs/02_Kconfig_System_And_Menuconfig.md)<br>
Writing and extending `.config`, `Config.in` files, dependency expressions (`depends on`, `select`, `if`), and how Kconfig maps to package enablement.

[03. **Toolchain Configuration & Management**](docs/03_Toolchain_Configuration_And_Management.md)<br>
Internal vs. external toolchains, `crosstool-NG` integration, `musl` vs `glibc` vs `uClibc-ng`, ABI selection, and floating-point options.

[04. **Cross-Compilation Fundamentals**](docs/04_Cross_Compilation_Fundamentals.md)<br>
`CROSS_COMPILE`, sysroot, `pkg-config` wrappers, `HOST_` vs `TARGET_` make variables, and common pitfalls when porting C++/Rust code.

---

## 📦 Package Infrastructure

[05. **Package Infrastructures (autotools, cmake, meson, generic)**](docs/05_Package_Infrastructures.md)<br>
When to use `autotools-package`, `cmake-package`, `meson-package` or `generic-package`; hook variables (`*_CONF_OPTS`, `*_INSTALL_TARGET_CMDS`) and override functions.

[06. **Writing Custom Packages from Scratch**](docs/06_Writing_Custom_Packages.md)<br>
Full anatomy of a `.mk` + `Config.in` pair, local source packages (`BR2_OVERRIDE_SRCDIR`), versioned tarballs, and Git-sourced packages.

[07. **Package Dependencies, Ordering & Virtual Packages**](docs/07_Package_Dependencies_Ordering_And_Virtual_Packages.md)<br>
`*_DEPENDENCIES`, `*_PROVIDES`, `*_CONFLICTS`, propagation of host/target split and how `Provides:` virtual packages let you swap implementations (e.g. OpenSSL vs LibreSSL).

[08. **Patching Packages**](docs/08_Patching_Packages.md)<br>
`series` files, `quilt`, naming conventions (`0001-*.patch`), upstream submission workflow, and applying patches conditionally per architecture.

[09. **C++ Packages — Cross-Compiled Libraries & Applications**](docs/09_Cpp_Packages_Cross_Compiled.md)<br>
Packaging a CMake C++ project, passing `CMAKE_CXX_FLAGS`, handling C++ standard selection (`-std=c++17`), linking against target sysroot libraries, and stripping debug symbols.

```cpp
// Example: CMakeLists.txt fragment for a cross-compiled C++ daemon
cmake_minimum_required(VERSION 3.16)
project(embedded_daemon CXX)
set(CMAKE_CXX_STANDARD 17)

find_package(Threads REQUIRED)

add_executable(embedded_daemon
    src/main.cpp
    src/sensor_reader.cpp
)
target_link_libraries(embedded_daemon PRIVATE Threads::Threads)
install(TARGETS embedded_daemon DESTINATION /usr/bin)
```

```makefile
# 09_embedded_daemon.mk
EMBEDDED_DAEMON_VERSION = 1.2.0
EMBEDDED_DAEMON_SITE    = $(BR2_EXTERNAL_MY_PATH)/package/embedded_daemon/src
EMBEDDED_DAEMON_SITE_METHOD = local
EMBEDDED_DAEMON_DEPENDENCIES = libcurl openssl

$(eval $(cmake-package))
```

[10. **Rust Packages with cargo-package Infrastructure**](docs/10_Rust_Packages_Cargo_Infrastructure.md)<br>
Using `cargo-package` (Buildroot ≥ 2023.02), `BR2_PACKAGE_RUST`, vendored dependencies (`cargo vendor`), cross-compilation targets (`thumbv7m-none-eabi`, `aarch64-unknown-linux-musl`), and `Cargo.lock` reproducibility.

```toml
# Cargo.toml for a Buildroot-targeted Rust binary
[package]
name    = "sensor-agent"
version = "0.3.1"
edition = "2021"

[dependencies]
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
tokio       = { version = "1", features = ["rt", "net", "io-util"] }

[profile.release]
opt-level = "z"   # size-optimised for embedded
lto       = true
strip     = true
```

```makefile
# 10_sensor_agent.mk
SENSOR_AGENT_VERSION      = 0.3.1
SENSOR_AGENT_SITE         = $(BR2_EXTERNAL_MY_PATH)/package/sensor_agent
SENSOR_AGENT_SITE_METHOD  = local
SENSOR_AGENT_CARGO_MODE   = release

$(eval $(cargo-package))
```

---

## 🐧 Kernel, Bootloader & Board Support

[11. **Linux Kernel Configuration & Custom defconfig**](docs/11_Linux_Kernel_Configuration_And_Defconfig.md)<br>
`BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE`, kernel fragments (`kernel_config_frag`), `make linux-menuconfig`, saving defconfigs, and in-tree vs. out-of-tree driver modules.

[12. **Bootloader Integration (U-Boot & GRUB2)**](docs/12_Bootloader_Integration_Uboot_And_Grub2.md)<br>
U-Boot `defconfig`, SPL/TPL stages, FIT images, `BR2_TARGET_UBOOT_*` variables, and GRUB2 for x86 embedded targets.

[13. **Device Tree & Board Support Packages (BSP)**](docs/13_Device_Tree_And_Board_Support_Packages.md)<br>
Maintaining `.dts`/`.dtsi` overlays in `BR2_EXTERNAL`, `BR2_LINUX_KERNEL_DTS_SUPPORT`, and structuring a complete BSP layer.

[14. **Secure Boot & Trusted Firmware (TF-A / OP-TEE)**](docs/14_Secure_Boot_And_Trusted_Firmware.md)<br>
Integrating ARM Trusted Firmware-A, OP-TEE OS, signing images with `fit-image`, and Buildroot packages `arm-trusted-firmware`, `optee-os`, `optee-client`.

---

## 🗂️ Filesystem, Storage & Images

[15. **Root Filesystem Configuration & Skeleton**](docs/15_Root_Filesystem_Configuration_And_Skeleton.md)<br>
`BR2_ROOTFS_SKELETON_CUSTOM`, permissions tables (`BR2_ROOTFS_DEVICE_TABLE`), user tables, and post-build hooks.

[16. **Filesystem Overlays**](docs/16_Filesystem_Overlays.md)<br>
`BR2_ROOTFS_OVERLAY` directories, merge order, overlayfs at runtime, and managing per-board config differences.

[17. **Partition Layout & Image Generation (genimage)**](docs/17_Partition_Layout_And_Image_Generation.md)<br>
`genimage.cfg`, producing `.img` / `.wic` / SD-card images, GPT/MBR layouts, and `post-image.sh` scripting.

[18. **Read-Only Root & Persistent Data Strategies**](docs/18_Read_Only_Root_And_Persistent_Data.md)<br>
`squashfs` + `overlayfs`, `tmpfs` mounts for `/var`, data partition mounting, and A/B update slot design.

---

## ⚙️ Init Systems & Service Management

[19. **Init Systems (BusyBox init, systemd, OpenRC)**](docs/19_Init_Systems_Busybox_Systemd_OpenRC.md)<br>
Selecting and tuning each init system, writing `inittab` for BusyBox, unit files for systemd, and minimising boot time.

[20. **Writing systemd Services for Embedded C++/Rust Daemons**](docs/20_Writing_Systemd_Services_For_Daemons.md)<br>
`Type=notify` with `sd_notify`, sandboxing (`CapabilityBoundingSet`, `PrivateTmp`), socket activation, and watchdog integration.

```cpp
// C++ daemon with systemd watchdog (libsystemd)
#include <systemd/sd-daemon.h>
#include <chrono>
#include <thread>

int main() {
    sd_notify(0, "READY=1");
    uint64_t watchdog_usec = 0;
    sd_watchdog_enabled(0, &watchdog_usec);

    while (true) {
        // ... do work ...
        sd_notify(0, "WATCHDOG=1");
        std::this_thread::sleep_for(std::chrono::seconds(5));
    }
}
```

```rust
// Rust daemon with systemd watchdog (libsystemd crate)
use std::time::Duration;
use libsystemd::daemon::{notify, NotifyState};

fn main() {
    notify(false, &[NotifyState::Ready]).unwrap();
    loop {
        // ... do work ...
        notify(false, &[NotifyState::Watchdog]).unwrap();
        std::thread::sleep(Duration::from_secs(5));
    }
}
```

---

## 🌐 Networking & Security

[21. **Networking Stack Configuration**](docs/21_Networking_Stack_Configuration.md)<br>
`iproute2`, `wpa_supplicant`, `connman`, `NetworkManager` packages; kernel `Kconfig` options for networking, and `udev` rules for interface naming.

[22. **Security Hardening**](docs/22_Security_Hardening.md)<br>
Compiler hardening flags (`-fstack-protector-strong`, `-D_FORTIFY_SOURCE=2`, `-fPIE`), stripping SUID binaries, removing dev tools, and `BR2_SSP_*` / `BR2_RELRO_*` options.

[23. **TLS & Cryptographic Libraries**](docs/23_TLS_And_Cryptographic_Libraries.md)<br>
OpenSSL vs. mbedTLS vs. WolfSSL selection, certificate deployment, hardware crypto engine integration, and PKCS#11 in C++/Rust.

---

## 🔁 BR2_EXTERNAL & Project Structure

[24. **BR2_EXTERNAL Mechanism**](docs/24_BR2_EXTERNAL_Mechanism.md)<br>
Structuring an out-of-tree layer (`external.mk`, `external.desc`, `Config.in`), stacking multiple `BR2_EXTERNAL` paths, and CI-friendly repo layouts.

[25. **Board Defconfigs & Multi-Board Repositories**](docs/25_Board_Defconfigs_And_Multi_Board_Repositories.md)<br>
Storing `*_defconfig` files in `BR2_EXTERNAL/configs/`, using fragments, sharing common config snippets across boards, and `make savedefconfig`.

---

## 🧪 Testing, Debugging & CI/CD

[26. **QEMU Emulation & Runtime Testing**](docs/26_QEMU_Emulation_And_Runtime_Testing.md)<br>
`BR2_TARGET_ROOTFS_EXT2` + QEMU for `x86_64`/`aarch64`/`RISC-V`, `start-qemu.sh`, and automated boot tests with `expect` scripts.

[27. **ptest Runtime Package Testing**](docs/27_Ptest_Runtime_Package_Testing.md)<br>
Enabling `BR2_PACKAGE_*_INSTALL_TESTS`, running `ptest-runner`, writing `ptest` harnesses for C++ (Catch2) and Rust (`#[test]`) packages.

[28. **CI/CD Pipeline Integration (GitLab / GitHub Actions)**](docs/28_CI_CD_Pipeline_Integration.md)<br>
Caching `dl/` and `ccache`, parallelism (`-j$(nproc)`), artifact promotion, Docker-based build containers, and `make legal-info` gating.

[29. **Debugging Embedded Targets (GDB, strace, perf)**](docs/29_Debugging_Embedded_Targets.md)<br>
`BR2_ENABLE_DEBUG`, remote GDB with `gdbserver`, `BR2_PACKAGE_STRACE`, `perf` cross-build, and JTAG integration via OpenOCD.

---

## 🔄 Updates, Licensing & Reproducibility

[30. **OTA Update Strategies (SWUpdate, RAUC, Mender)**](docs/30_OTA_Update_Strategies.md)<br>
A/B partition switching, `SWUpdate` `.swu` bundles, `RAUC` bundle signing, and Buildroot integration packages.

[31. **License Compliance & Legal Manifest**](docs/31_License_Compliance_And_Legal_Manifest.md)<br>
`make legal-info`, understanding `*_LICENSE`, `*_LICENSE_FILES`, SPDX identifiers, GPL boundary management, and export control.

[32. **Reproducible Builds**](docs/32_Reproducible_Builds.md)<br>
`BR2_REPRODUCIBLE`, SOURCE_DATE_EPOCH, deterministic toolchain versions, `diffoscope` for binary comparison, and pinning package versions.

