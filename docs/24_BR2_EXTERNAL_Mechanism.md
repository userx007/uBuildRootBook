# 24. BR2_EXTERNAL Mechanism

**Structure & Architecture**
- Conceptual ASCII diagram showing how Buildroot core, platform layer, and product layer relate
- Full canonical directory layout for an external layer
- Deep-dive into all three mandatory files (`external.desc`, `external.mk`, `Config.in`)

**Three Language Examples**
- **C** (`hello_sensor`) — a minimal sensor daemon using `generic-package` with a hand-written `Makefile`, `ioctl`-based sensor abstraction, and syslog integration
- **C++17** (`cpp_telemetry`) — a thread-safe telemetry aggregator using `cmake-package`, with type-safe channels, callbacks, and an ASCII stats table rendered via `std::setw`
- **Rust** (`rust_monitor`) — a `/proc`-based system monitor using `cargo-package`, with a live ASCII bar-chart dashboard for memory and load average

**Stacking & CI**
- How `:` path stacking works, layer variable naming, and persistence via `.br2-external.mk`
- Monorepo layout with git submodule pinning, a convenience wrapper `Makefile`, and full GitHub Actions + GitLab CI pipeline configs

**Pitfalls table** covering the five most common mistakes (hardcoded paths, duplicate names, non-deterministic wildcard, committing output, missing `O=`) with correct/incorrect examples side by side.


> **Buildroot Series — Topic 24**
> Structuring an out-of-tree layer (`external.mk`, `external.desc`, `Config.in`),
> stacking multiple `BR2_EXTERNAL` paths, and CI-friendly repo layouts.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Conceptual Architecture](#2-conceptual-architecture)
3. [Directory Structure of a BR2_EXTERNAL Layer](#3-directory-structure-of-a-br2_external-layer)
4. [The Three Mandatory Files](#4-the-three-mandatory-files)
   - 4.1 `external.desc`
   - 4.2 `external.mk`
   - 4.3 `Config.in`
5. [Registering Packages in the External Layer](#5-registering-packages-in-the-external-layer)
6. [C Example Package — `hello_sensor`](#6-c-example-package--hello_sensor)
7. [C++ Example Package — `cpp_telemetry`](#7-c-example-package--cpp_telemetry)
8. [Rust Example Package — `rust_monitor`](#8-rust-example-package--rust_monitor)
9. [Stacking Multiple BR2_EXTERNAL Paths](#9-stacking-multiple-br2_external-paths)
10. [CI-Friendly Repository Layouts](#10-ci-friendly-repository-layouts)
11. [Common Pitfalls and Best Practices](#11-common-pitfalls-and-best-practices)
12. [Summary](#12-summary)

---

## 1. Introduction

Buildroot's `BR2_EXTERNAL` mechanism provides a **clean, non-invasive way** to extend
the standard Buildroot tree with custom packages, board support files, toolchain
configurations, and defconfigs — without touching the upstream Buildroot source at all.

This separation between the vendor/project layer and the upstream Buildroot tree brings
several critical engineering benefits:

- **Upgrade safety**: The Buildroot tree can be updated (e.g., via `git pull`) without
  risk of merge conflicts with in-house customizations.
- **Multi-project scalability**: One Buildroot installation can serve many products by
  pointing `BR2_EXTERNAL` to different layer repositories.
- **Clear IP boundary**: Proprietary or confidential code stays in a private repository;
  the open-source Buildroot tree remains unmodified.
- **Layer composability**: Multiple `BR2_EXTERNAL` paths can be stacked with a `:` separator,
  allowing shared infrastructure layers to coexist with product-specific layers.

The variable `BR2_EXTERNAL` is set on the `make` command line and persists in the build
output directory via `$(O)/.br2-external.mk`.

---

## 2. Conceptual Architecture

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                        BUILD INVOCATION                             │
  │                                                                     │
  │  make BR2_EXTERNAL=/layers/platform:/layers/product O=build/ ...    │
  └───────────────────┬─────────────────────────────────────────────────┘
                      │
          ┌───────────▼────────────────────────────────┐
          │           Buildroot Core Tree              │
          │  (upstream, unmodified, read-only)         │
          │                                            │
          │  ┌──────────────┐  ┌────────────────────┐  │
          │  │  package/    │  │  arch/, boot/,     │  │
          │  │  (upstream)  │  │  fs/, linux/, ...  │  │
          │  └──────────────┘  └────────────────────┘  │
          │                                            │
          │  Includes ──────────────────────────────── │
          └──────────┬──────────────┬──────────────────┘
                     │              │
      ┌──────────────▼───┐   ┌──────▼───────────────────┐
      │  LAYER: platform │   │   LAYER: product         │
      │  (shared infra)  │   │   (product-specific)     │
      │                  │   │                          │
      │  external.desc   │   │  external.desc           │
      │  external.mk     │   │  external.mk             │
      │  Config.in       │   │  Config.in               │
      │  package/        │   │  package/                │
      │  board/          │   │  board/                  │
      │  configs/        │   │  configs/                │
      └──────────────────┘   └──────────────────────────┘
                     │              │
                     └──────┬───────┘
                            │
              ┌─────────────▼───────────────┐
              │      BUILD OUTPUT (O=)      │
              │  .br2-external.mk           │
              │  (persists BR2_EXTERNAL)    │
              └─────────────────────────────┘
```

---

## 3. Directory Structure of a BR2_EXTERNAL Layer

A well-structured external layer follows this canonical layout:

```
my-external-layer/
│
├── external.desc          ← Mandatory: layer name and description
├── external.mk            ← Mandatory: Makefile hook point
├── Config.in              ← Mandatory: Kconfig entry point
│
├── package/               ← Custom packages (each in sub-directory)
│   ├── hello_sensor/
│   │   ├── Config.in
│   │   └── hello_sensor.mk
│   ├── cpp_telemetry/
│   │   ├── Config.in
│   │   └── cpp_telemetry.mk
│   └── rust_monitor/
│       ├── Config.in
│       └── rust_monitor.mk
│
├── board/                 ← Board-specific files
│   └── myboard/
│       ├── rootfs-overlay/
│       │   └── etc/
│       │       └── init.d/
│       │           └── S99myapp
│       ├── post-build.sh
│       ├── post-image.sh
│       └── patches/
│           └── linux/
│               └── 0001-custom-driver.patch
│
├── configs/               ← Defconfigs for this layer's targets
│   ├── myboard_defconfig
│   └── myboard_debug_defconfig
│
├── linux/                 ← Optional: Linux kernel config fragments
│   └── myboard.config
│
└── docs/
    └── layer.md
```

---

## 4. The Three Mandatory Files

### 4.1 `external.desc`

This file **identifies the layer** to Buildroot. It must contain exactly two fields:

```
name: PLATFORM_LAYER
desc: Shared platform infrastructure for MyCompany embedded products
```

- **`name`**: A unique identifier in `UPPER_CASE`. Used internally and exposed as
  `BR2_EXTERNAL_PLATFORM_LAYER_PATH` pointing to the layer root.
- **`desc`**: A human-readable description shown in `make menuconfig`.

> ⚠️ **No spaces in the name.** Only letters, digits, and underscores are allowed.
> The name must be unique across all stacked layers.

---

### 4.2 `external.mk`

This Makefile is **included by the top-level Buildroot Makefile**. Its primary role is
to pull in the `.mk` files of all packages defined in the layer:

```makefile
# external.mk — include all custom package makefiles

include $(sort $(wildcard $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/*/*.mk))
```

The `$(sort $(wildcard ...))` idiom is idiomatic Buildroot: it ensures deterministic
ordering and that every package `.mk` gets included exactly once.

You may also include layer-level make logic here, such as setting default variables:

```makefile
# Define a layer-level variable available to all packages
PLATFORM_LAYER_VERSION ?= 2.4.1

include $(sort $(wildcard $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/*/*.mk))
```

---

### 4.3 `Config.in`

The top-level `Config.in` of the layer is the **Kconfig entry point**. It must source
every package's own `Config.in`:

```kconfig
menu "MyCompany Platform Packages"

source "$BR2_EXTERNAL_PLATFORM_LAYER_PATH/package/hello_sensor/Config.in"
source "$BR2_EXTERNAL_PLATFORM_LAYER_PATH/package/cpp_telemetry/Config.in"
source "$BR2_EXTERNAL_PLATFORM_LAYER_PATH/package/rust_monitor/Config.in"

endmenu
```

The variable `$BR2_EXTERNAL_PLATFORM_LAYER_PATH` is automatically set by Buildroot and
always resolves to the absolute path of the layer root.

---

## 5. Registering Packages in the External Layer

Every custom package inside `package/<name>/` needs its own pair of files:

### `package/<name>/Config.in`

```kconfig
config BR2_PACKAGE_HELLO_SENSOR
    bool "hello_sensor"
    help
      A minimal C sensor daemon that reads /dev/sensor0
      and logs values to syslog.

      This package demonstrates the BR2_EXTERNAL mechanism.
```

### `package/<name>/<name>.mk`

```makefile
################################################################################
#
# hello_sensor
#
################################################################################

HELLO_SENSOR_VERSION = 1.2.3
HELLO_SENSOR_SITE    = $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/hello_sensor/src
HELLO_SENSOR_SITE_METHOD = local

define HELLO_SENSOR_BUILD_CMDS
    $(MAKE) $(TARGET_CONFIGURE_OPTS) -C $(@D)
endef

define HELLO_SENSOR_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/hello_sensor $(TARGET_DIR)/usr/bin/hello_sensor
endef

$(eval $(generic-package))
```

The macro `$(generic-package)` (or `$(autotools-package)`, `$(cmake-package)`,
`$(cargo-package)` etc.) wires the package into Buildroot's dependency and build graph.

---

## 6. C Example Package — `hello_sensor`

This package demonstrates a **minimal C sensor daemon** built entirely within the
external layer using a hand-written `Makefile`.

### Package layout

```
package/hello_sensor/
├── Config.in
├── hello_sensor.mk
└── src/
    ├── Makefile
    ├── main.c
    ├── sensor.c
    └── sensor.h
```

### `src/sensor.h`

```c
/* sensor.h — Sensor abstraction for hello_sensor */
#ifndef SENSOR_H
#define SENSOR_H

#include <stdint.h>

#define SENSOR_DEVICE   "/dev/sensor0"
#define SAMPLE_INTERVAL 2   /* seconds between readings */

typedef struct {
    int32_t temperature_mdeg;   /* millidegrees Celsius */
    uint32_t pressure_pa;       /* Pascals */
    uint8_t  valid;
} sensor_reading_t;

/**
 * sensor_open  - Open the sensor device; returns fd or -1 on error.
 */
int sensor_open(const char *path);

/**
 * sensor_read  - Populate *r from the open fd.
 *                Returns 0 on success, -1 on error.
 */
int sensor_read(int fd, sensor_reading_t *r);

/**
 * sensor_close - Release the fd.
 */
void sensor_close(int fd);

#endif /* SENSOR_H */
```

### `src/sensor.c`

```c
/* sensor.c — Platform sensor driver wrapper */
#include "sensor.h"

#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

/* Simulated ioctl command — replace with your driver's header */
#define SENSOR_IOCTL_READ   _IOR('S', 1, sensor_reading_t)

int sensor_open(const char *path)
{
    int fd = open(path, O_RDONLY | O_CLOEXEC);
    if (fd < 0)
        fprintf(stderr, "sensor_open: %s: %s\n", path, strerror(errno));
    return fd;
}

int sensor_read(int fd, sensor_reading_t *r)
{
    if (!r) {
        errno = EINVAL;
        return -1;
    }

    /* Try ioctl first; fall back to read() for simple char devices */
    if (ioctl(fd, SENSOR_IOCTL_READ, r) == 0) {
        r->valid = 1;
        return 0;
    }

    /* Fallback: read raw bytes — device-specific interpretation */
    uint8_t raw[8];
    ssize_t n = read(fd, raw, sizeof(raw));
    if (n != sizeof(raw)) {
        r->valid = 0;
        return -1;
    }

    /* Example: first 4 bytes = temp in millidegrees, next 4 = pressure */
    memcpy(&r->temperature_mdeg, raw,     4);
    memcpy(&r->pressure_pa,      raw + 4, 4);
    r->valid = 1;
    return 0;
}

void sensor_close(int fd)
{
    if (fd >= 0)
        close(fd);
}
```

### `src/main.c`

```c
/* main.c — Sensor daemon entry point */
#include "sensor.h"

#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <syslog.h>
#include <unistd.h>

static volatile int g_running = 1;

static void handle_signal(int sig)
{
    (void)sig;
    g_running = 0;
}

int main(int argc, char *argv[])
{
    const char *dev = (argc > 1) ? argv[1] : SENSOR_DEVICE;

    openlog("hello_sensor", LOG_PID | LOG_CONS, LOG_DAEMON);
    syslog(LOG_INFO, "Starting — device: %s", dev);

    signal(SIGTERM, handle_signal);
    signal(SIGINT,  handle_signal);

    int fd = sensor_open(dev);
    if (fd < 0) {
        syslog(LOG_ERR, "Failed to open sensor device, exiting");
        closelog();
        return EXIT_FAILURE;
    }

    sensor_reading_t r;

    while (g_running) {
        if (sensor_read(fd, &r) == 0 && r.valid) {
            syslog(LOG_INFO, "Temp: %+.3f °C  Pressure: %u Pa",
                   r.temperature_mdeg / 1000.0,
                   r.pressure_pa);
        } else {
            syslog(LOG_WARNING, "Sensor read failed");
        }
        sleep(SAMPLE_INTERVAL);
    }

    syslog(LOG_INFO, "Shutting down");
    sensor_close(fd);
    closelog();
    return EXIT_SUCCESS;
}
```

### `src/Makefile`

```makefile
CC      ?= gcc
CFLAGS  ?= -Wall -Wextra -O2
TARGET   = hello_sensor
SRCS     = main.c sensor.c
OBJS     = $(SRCS:.c=.o)

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(LDFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: all clean
```

### `package/hello_sensor/hello_sensor.mk`

```makefile
################################################################################
# hello_sensor — minimal C sensor daemon
################################################################################

HELLO_SENSOR_VERSION     = 1.2.3
HELLO_SENSOR_SITE        = $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/hello_sensor/src
HELLO_SENSOR_SITE_METHOD = local
HELLO_SENSOR_LICENSE     = MIT
HELLO_SENSOR_LICENSE_FILES = LICENSE

define HELLO_SENSOR_BUILD_CMDS
    $(MAKE) $(TARGET_CONFIGURE_OPTS) \
        CFLAGS="$(TARGET_CFLAGS)" \
        LDFLAGS="$(TARGET_LDFLAGS)" \
        -C $(@D)
endef

define HELLO_SENSOR_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/hello_sensor \
        $(TARGET_DIR)/usr/bin/hello_sensor
    $(INSTALL) -D -m 0644 \
        $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/board/myboard/rootfs-overlay/etc/init.d/S99hello_sensor \
        $(TARGET_DIR)/etc/init.d/S99hello_sensor
endef

$(eval $(generic-package))
```

---

## 7. C++ Example Package — `cpp_telemetry`

This package is a **C++17 telemetry aggregator** built with CMake, demonstrating a
more realistic cross-compilation scenario within a BR2_EXTERNAL layer.

### Package layout

```
package/cpp_telemetry/
├── Config.in
├── cpp_telemetry.mk
└── src/
    ├── CMakeLists.txt
    ├── include/
    │   ├── Channel.hpp
    │   └── Aggregator.hpp
    └── src/
        ├── Channel.cpp
        ├── Aggregator.cpp
        └── main.cpp
```

### `include/Channel.hpp`

```cpp
// Channel.hpp — Type-safe telemetry channel
#pragma once

#include <cstdint>
#include <functional>
#include <optional>
#include <string>
#include <string_view>

namespace telemetry {

enum class ChannelType : uint8_t {
    Temperature,
    Pressure,
    Humidity,
    Voltage,
    Custom
};

struct Sample {
    double      value;
    uint64_t    timestamp_ms;
    bool        valid;
};

class Channel {
public:
    using SampleCallback = std::function<void(const Sample&)>;

    explicit Channel(std::string_view name, ChannelType type)
        : name_(name), type_(type) {}

    Channel(const Channel&)            = delete;
    Channel& operator=(const Channel&) = delete;
    Channel(Channel&&)                 = default;

    std::string_view    name()  const noexcept { return name_;  }
    ChannelType         type()  const noexcept { return type_;  }

    void push(Sample s);
    std::optional<Sample> last() const noexcept { return last_; }

    void set_callback(SampleCallback cb) { callback_ = std::move(cb); }

private:
    std::string             name_;
    ChannelType             type_;
    std::optional<Sample>   last_;
    SampleCallback          callback_;
};

} // namespace telemetry
```

### `src/Channel.cpp`

```cpp
// Channel.cpp
#include "Channel.hpp"

#include <chrono>

namespace telemetry {

static uint64_t now_ms()
{
    using namespace std::chrono;
    return static_cast<uint64_t>(
        duration_cast<milliseconds>(
            steady_clock::now().time_since_epoch()
        ).count()
    );
}

void Channel::push(Sample s)
{
    if (!s.timestamp_ms)
        s.timestamp_ms = now_ms();

    last_ = s;

    if (callback_ && s.valid)
        callback_(s);
}

} // namespace telemetry
```

### `include/Aggregator.hpp`

```cpp
// Aggregator.hpp — Thread-safe multi-channel aggregator
#pragma once

#include "Channel.hpp"

#include <memory>
#include <mutex>
#include <string>
#include <unordered_map>
#include <vector>

namespace telemetry {

struct ChannelStats {
    double   min, max, mean;
    uint64_t count;
};

class Aggregator {
public:
    Aggregator()  = default;
    ~Aggregator() = default;

    /**
     * add_channel — register a new channel by name.
     * Throws std::runtime_error if the name is already registered.
     */
    Channel& add_channel(std::string name, ChannelType type);

    /**
     * push_sample — push a named sample from any thread.
     */
    void push_sample(std::string_view name, double value, bool valid = true);

    /**
     * stats — compute statistics over all valid samples in channel.
     */
    std::optional<ChannelStats> stats(std::string_view name) const;

    /**
     * report — print a formatted ASCII table of all channel stats to stdout.
     */
    void report() const;

private:
    mutable std::mutex                               mu_;
    std::unordered_map<std::string,
        std::unique_ptr<Channel>>                    channels_;
    std::unordered_map<std::string,
        std::vector<double>>                         history_;
};

} // namespace telemetry
```

### `src/Aggregator.cpp`

```cpp
// Aggregator.cpp
#include "Aggregator.hpp"

#include <algorithm>
#include <cmath>
#include <iomanip>
#include <iostream>
#include <numeric>
#include <stdexcept>

namespace telemetry {

Channel& Aggregator::add_channel(std::string name, ChannelType type)
{
    std::lock_guard<std::mutex> lock(mu_);
    if (channels_.count(name))
        throw std::runtime_error("Duplicate channel: " + name);

    auto ch = std::make_unique<Channel>(name, type);

    // Record every valid sample in history
    ch->set_callback([this, n = name](const Sample& s) {
        std::lock_guard<std::mutex> lk(mu_);
        history_[n].push_back(s.value);
    });

    auto& ref = *ch;
    channels_.emplace(std::move(name), std::move(ch));
    return ref;
}

void Aggregator::push_sample(std::string_view name, double value, bool valid)
{
    std::lock_guard<std::mutex> lock(mu_);
    auto it = channels_.find(std::string(name));
    if (it == channels_.end())
        throw std::runtime_error("Unknown channel");
    it->second->push({value, 0, valid});
}

std::optional<ChannelStats> Aggregator::stats(std::string_view name) const
{
    std::lock_guard<std::mutex> lock(mu_);
    auto it = history_.find(std::string(name));
    if (it == history_.end() || it->second.empty())
        return std::nullopt;

    const auto& v = it->second;
    ChannelStats s{};
    s.count = v.size();
    s.min   = *std::min_element(v.begin(), v.end());
    s.max   = *std::max_element(v.begin(), v.end());
    s.mean  = std::accumulate(v.begin(), v.end(), 0.0) / s.count;
    return s;
}

void Aggregator::report() const
{
    std::lock_guard<std::mutex> lock(mu_);

    // ASCII table header
    std::cout << "\n";
    std::cout << "+----------------------+----------+----------+----------+-------+\n";
    std::cout << "| Channel              |      Min |      Max |     Mean | Count |\n";
    std::cout << "+----------------------+----------+----------+----------+-------+\n";

    for (const auto& [name, ch] : channels_) {
        auto it = history_.find(name);
        if (it == history_.end() || it->second.empty()) {
            std::cout << "| " << std::left << std::setw(20) << name
                      << " | (no data)                              |\n";
            continue;
        }
        const auto& v = it->second;
        double mn  = *std::min_element(v.begin(), v.end());
        double mx  = *std::max_element(v.begin(), v.end());
        double avg = std::accumulate(v.begin(), v.end(), 0.0) / v.size();

        std::cout << "| " << std::left  << std::setw(20) << name << " | "
                  << std::right << std::setw(8) << std::fixed << std::setprecision(2) << mn  << " | "
                  << std::setw(8) << mx  << " | "
                  << std::setw(8) << avg << " | "
                  << std::setw(5) << v.size() << " |\n";
    }

    std::cout << "+----------------------+----------+----------+----------+-------+\n\n";
}

} // namespace telemetry
```

### `src/main.cpp`

```cpp
// main.cpp — Telemetry aggregator demo
#include "Aggregator.hpp"

#include <chrono>
#include <cstdlib>
#include <iostream>
#include <random>
#include <thread>

int main()
{
    telemetry::Aggregator agg;

    auto& temp  = agg.add_channel("temperature",  telemetry::ChannelType::Temperature);
    auto& press = agg.add_channel("pressure",      telemetry::ChannelType::Pressure);
    auto& volt  = agg.add_channel("supply_voltage",telemetry::ChannelType::Voltage);

    (void)temp; (void)press; (void)volt; // callbacks wired via aggregator

    std::mt19937 rng(42);
    std::normal_distribution<double> d_temp(25.0, 1.5);
    std::normal_distribution<double> d_press(101325.0, 500.0);
    std::normal_distribution<double> d_volt(3.3, 0.05);

    std::cout << "Collecting 20 samples per channel...\n";
    for (int i = 0; i < 20; ++i) {
        agg.push_sample("temperature",   d_temp(rng));
        agg.push_sample("pressure",      d_press(rng));
        agg.push_sample("supply_voltage",d_volt(rng));
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }

    agg.report();
    return EXIT_SUCCESS;
}
```

### Sample output from `agg.report()`

```
+----------------------+----------+----------+----------+-------+
| Channel              |      Min |      Max |     Mean | Count |
+----------------------+----------+----------+----------+-------+
| temperature          |    22.54 |    27.81 |    25.12 |    20 |
| pressure             | 100342.7 | 102415.3 | 101318.6 |    20 |
| supply_voltage       |     3.19 |     3.41 |     3.30 |    20 |
+----------------------+----------+----------+----------+-------+
```

### `package/cpp_telemetry/cpp_telemetry.mk`

```makefile
################################################################################
# cpp_telemetry — C++17 telemetry aggregator
################################################################################

CPP_TELEMETRY_VERSION     = 0.9.0
CPP_TELEMETRY_SITE        = $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/cpp_telemetry/src
CPP_TELEMETRY_SITE_METHOD = local
CPP_TELEMETRY_LICENSE     = Apache-2.0

CPP_TELEMETRY_CONF_OPTS = \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=17

$(eval $(cmake-package))
```

---

## 8. Rust Example Package — `rust_monitor`

This package implements a **system resource monitor** in Rust that periodically
reads `/proc/meminfo` and `/proc/loadavg`, then prints an ASCII bar-chart dashboard.

### Package layout

```
package/rust_monitor/
├── Config.in
├── rust_monitor.mk
└── src/
    ├── Cargo.toml
    └── src/
        ├── main.rs
        ├── procfs.rs
        └── display.rs
```

### `src/Cargo.toml`

```toml
[package]
name    = "rust_monitor"
version = "0.3.0"
edition = "2021"

[[bin]]
name = "rust_monitor"
path = "src/main.rs"

[dependencies]
# No external crates — uses only std and /proc filesystem
```

### `src/src/procfs.rs`

```rust
//! procfs.rs — Read system metrics from /proc
use std::fs;
use std::io::{self, BufRead, BufReader};
use std::str::FromStr;

#[derive(Debug, Default)]
pub struct MemInfo {
    pub total_kb:   u64,
    pub free_kb:    u64,
    pub available_kb: u64,
    pub buffers_kb: u64,
    pub cached_kb:  u64,
}

impl MemInfo {
    pub fn used_kb(&self) -> u64 {
        self.total_kb.saturating_sub(self.available_kb)
    }

    pub fn used_pct(&self) -> f64 {
        if self.total_kb == 0 { return 0.0; }
        self.used_kb() as f64 / self.total_kb as f64 * 100.0
    }
}

pub fn read_meminfo() -> io::Result<MemInfo> {
    let f = fs::File::open("/proc/meminfo")?;
    let reader = BufReader::new(f);
    let mut m = MemInfo::default();

    for line in reader.lines() {
        let line = line?;
        let mut parts = line.split_ascii_whitespace();
        let key = parts.next().unwrap_or("");
        let val: u64 = parts
            .next()
            .and_then(|v| u64::from_str(v).ok())
            .unwrap_or(0);

        match key {
            "MemTotal:"     => m.total_kb     = val,
            "MemFree:"      => m.free_kb      = val,
            "MemAvailable:" => m.available_kb = val,
            "Buffers:"      => m.buffers_kb   = val,
            "Cached:"       => m.cached_kb    = val,
            _ => {}
        }
    }
    Ok(m)
}

#[derive(Debug, Default)]
pub struct LoadAvg {
    pub load1:  f64,
    pub load5:  f64,
    pub load15: f64,
}

pub fn read_loadavg() -> io::Result<LoadAvg> {
    let content = fs::read_to_string("/proc/loadavg")?;
    let mut parts = content.split_ascii_whitespace();
    Ok(LoadAvg {
        load1:  parts.next().and_then(|v| v.parse().ok()).unwrap_or(0.0),
        load5:  parts.next().and_then(|v| v.parse().ok()).unwrap_or(0.0),
        load15: parts.next().and_then(|v| v.parse().ok()).unwrap_or(0.0),
    })
}
```

### `src/src/display.rs`

```rust
//! display.rs — ASCII bar chart renderer for terminal dashboards

use crate::procfs::{LoadAvg, MemInfo};

const BAR_WIDTH: usize = 40;

fn bar(pct: f64) -> String {
    let filled = ((pct / 100.0) * BAR_WIDTH as f64).round() as usize;
    let filled = filled.min(BAR_WIDTH);
    let empty   = BAR_WIDTH - filled;
    format!("[{}{}]", "█".repeat(filled), "░".repeat(empty))
}

fn load_bar(load: f64, max_load: f64) -> String {
    let pct = (load / max_load * 100.0).min(100.0);
    let filled = ((pct / 100.0) * BAR_WIDTH as f64).round() as usize;
    let empty  = BAR_WIDTH - filled;
    format!("[{}{}]", "▓".repeat(filled), "░".repeat(empty))
}

pub fn render(mem: &MemInfo, load: &LoadAvg, cpu_count: usize) {
    let separator = "+------------------------------------------------------+";
    let title     = "|          rust_monitor — System Resource Dashboard    |";

    println!("{}", separator);
    println!("{}", title);
    println!("{}", separator);

    // Memory
    let total_mb = mem.total_kb / 1024;
    let used_mb  = mem.used_kb() / 1024;
    let pct      = mem.used_pct();

    println!("| Memory  {:>6} MiB / {:>6} MiB  ({:5.1}%)           |",
             used_mb, total_mb, pct);
    println!("|   {}  |", bar(pct));
    println!("|                                                      |");

    // Load average
    println!("| Load Average  (1m / 5m / 15m)                        |");
    println!("|   1m  {:>5.2}  {}  |",
             load.load1,  load_bar(load.load1,  cpu_count as f64));
    println!("|   5m  {:>5.2}  {}  |",
             load.load5,  load_bar(load.load5,  cpu_count as f64));
    println!("|  15m  {:>5.2}  {}  |",
             load.load15, load_bar(load.load15, cpu_count as f64));

    println!("{}", separator);
}
```

### `src/src/main.rs`

```rust
//! main.rs — rust_monitor entry point
mod display;
mod procfs;

use std::thread;
use std::time::Duration;

fn cpu_count() -> usize {
    // Parse /proc/cpuinfo for embedded targets (no std::thread::available_parallelism
    // on older Linux kernels)
    std::fs::read_to_string("/proc/cpuinfo")
        .map(|s| s.lines().filter(|l| l.starts_with("processor")).count())
        .unwrap_or(1)
        .max(1)
}

fn main() {
    let interval_secs: u64 = std::env::args()
        .nth(1)
        .and_then(|a| a.parse().ok())
        .unwrap_or(5);

    let cpus = cpu_count();
    eprintln!("rust_monitor: {} CPU(s), refresh every {}s", cpus, interval_secs);

    loop {
        // Clear screen — ANSI escape or just newlines for dumb terminals
        print!("\x1B[2J\x1B[H");

        let mem  = procfs::read_meminfo().unwrap_or_default();
        let load = procfs::read_loadavg().unwrap_or_default();

        display::render(&mem, &load, cpus);

        thread::sleep(Duration::from_secs(interval_secs));
    }
}
```

### Sample ASCII dashboard output

```
+------------------------------------------------------+
|          rust_monitor — System Resource Dashboard    |
+------------------------------------------------------+
| Memory    312 MiB /    512 MiB  ( 61.0%)             |
|   [████████████████████████░░░░░░░░░░░░░░░░░░░░░░]   |
|                                                      |
| Load Average  (1m / 5m / 15m)                        |
|   1m   0.42  [██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  |
|   5m   0.68  [██████████░░░░░░░░░░░░░░░░░░░░░░░░░░]  |
|  15m   0.31  [████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  |
+------------------------------------------------------+
```

### `package/rust_monitor/rust_monitor.mk`

```makefile
################################################################################
# rust_monitor — Rust system resource monitor
################################################################################

RUST_MONITOR_VERSION     = 0.3.0
RUST_MONITOR_SITE        = $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/rust_monitor/src
RUST_MONITOR_SITE_METHOD = local
RUST_MONITOR_LICENSE     = MIT
RUST_MONITOR_CARGO_BIN   = rust_monitor

$(eval $(cargo-package))
```

> **Note**: For Rust packages, Buildroot's `cargo-package` infrastructure handles
> cross-compilation via `cargo build --target $(RUSTC_TARGET_NAME)` automatically.
> The `BR2_PACKAGE_HOST_RUSTC` dependency is implied by `cargo-package`.

---

## 9. Stacking Multiple BR2_EXTERNAL Paths

Buildroot supports stacking layers by separating paths with `:` on the command line:

```bash
make BR2_EXTERNAL=/opt/layers/platform:/opt/layers/product O=build/
```

### How stacking works

```
  BR2_EXTERNAL="/opt/layers/platform:/opt/layers/product"
       │
       ├─── Layer 1: /opt/layers/platform
       │    external.desc  →  name: PLATFORM_LAYER
       │    Config.in      →  sourced as menu "Platform Packages"
       │    external.mk    →  includes platform package .mk files
       │
       └─── Layer 2: /opt/layers/product
            external.desc  →  name: PRODUCT_LAYER
            Config.in      →  sourced as menu "Product Packages"
            external.mk    →  includes product package .mk files
```

Buildroot synthesizes a **virtual wrapper layer** in `$(O)/.br2-external.mk` that
includes all stacked layers in order. Each layer's `external.mk` and `Config.in` are
included sequentially.

### Variable naming per layer

Each layer gets its own path variable derived from its `name` in `external.desc`:

```
Layer name: PLATFORM_LAYER  →  $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)
Layer name: PRODUCT_LAYER   →  $(BR2_EXTERNAL_PRODUCT_LAYER_PATH)
```

These are fully independent and can be used in any `.mk` or `Config.in` file:

```makefile
# In a product package .mk file:
MY_APP_SITE = $(BR2_EXTERNAL_PRODUCT_LAYER_PATH)/package/my_app/src

# Referencing shared files from the platform layer:
MY_APP_PATCHES = $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/common_patches/
```

### Stacking rules

| Rule | Details |
|------|---------|
| Names must be unique | Duplicate `name` in `external.desc` causes a build error |
| Order matters for Config.in | Packages from layer 1 appear before layer 2 in `menuconfig` |
| No circular references | Layer A may not `source` Config.in from layer B via path |
| Defconfigs are independent | Each layer's `configs/` is searched for defconfigs |
| Packages from all layers coexist | No namespace collision as long as `BR2_PACKAGE_` symbols differ |

### Persisting the stack

After the first `make` invocation, the paths are saved to `$(O)/.br2-external.mk`:

```
# $(O)/.br2-external.mk — auto-generated, do not edit
BR2_EXTERNAL := /opt/layers/platform:/opt/layers/product
```

Subsequent `make` invocations in the same `O=` directory do **not** need to repeat
`BR2_EXTERNAL=...` on the command line.

---

## 10. CI-Friendly Repository Layouts

### Monorepo layout

```
mycompany/
├── buildroot/               ← Buildroot submodule (pinned tag)
│   └── ...
│
├── layers/
│   ├── platform/            ← Shared platform layer (git submodule or dir)
│   │   ├── external.desc
│   │   ├── external.mk
│   │   ├── Config.in
│   │   ├── package/
│   │   └── configs/
│   │
│   └── product-alpha/       ← Product-specific layer
│       ├── external.desc
│       ├── external.mk
│       ├── Config.in
│       ├── package/
│       └── configs/
│           └── alpha_defconfig
│
├── output/                  ← Build output (gitignored)
│   └── .br2-external.mk
│
├── Makefile                 ← Convenience wrapper
└── .gitlab-ci.yml / .github/workflows/build.yml
```

### Convenience top-level `Makefile`

```makefile
# Top-level Makefile — thin wrapper around Buildroot
BUILDROOT   := $(CURDIR)/buildroot
LAYERS      := $(CURDIR)/layers/platform:$(CURDIR)/layers/product-alpha
OUTPUT      := $(CURDIR)/output

BR2_OPTS     ?=
DEFCONFIG    ?= alpha_defconfig

.PHONY: menuconfig defconfig build clean distclean sdk

defconfig:
	$(MAKE) -C $(BUILDROOT) \
	    BR2_EXTERNAL="$(LAYERS)" \
	    O="$(OUTPUT)" \
	    $(DEFCONFIG)

menuconfig:
	$(MAKE) -C $(BUILDROOT) \
	    BR2_EXTERNAL="$(LAYERS)" \
	    O="$(OUTPUT)" \
	    menuconfig

build:
	$(MAKE) -C $(BUILDROOT) \
	    O="$(OUTPUT)" \
	    $(BR2_OPTS)

sdk:
	$(MAKE) -C $(BUILDROOT) \
	    O="$(OUTPUT)" \
	    sdk

clean:
	$(MAKE) -C $(BUILDROOT) \
	    O="$(OUTPUT)" \
	    clean

distclean:
	rm -rf $(OUTPUT)
```

### GitHub Actions CI workflow

```yaml
# .github/workflows/build.yml
name: Buildroot CI

on:
  push:
    branches: [main, release/*]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-22.04
    timeout-minutes: 120

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Buildroot host dependencies
        run: |
          sudo apt-get update -qq
          sudo apt-get install -y --no-install-recommends \
            bc build-essential cpio file libncurses-dev \
            rsync unzip wget python3 python3-dev

      - name: Install Rust toolchain (for cargo-package)
        uses: dtolnay/rust-toolchain@stable

      - name: Cache Buildroot DL directory
        uses: actions/cache@v4
        with:
          path: dl/
          key: br2-dl-${{ hashFiles('layers/**/**.mk') }}
          restore-keys: |
            br2-dl-

      - name: Cache ccache
        uses: actions/cache@v4
        with:
          path: ~/.buildroot-ccache
          key: br2-ccache-${{ github.ref_name }}-${{ github.sha }}
          restore-keys: |
            br2-ccache-${{ github.ref_name }}-
            br2-ccache-

      - name: Configure
        run: make defconfig DEFCONFIG=alpha_defconfig

      - name: Build
        run: make build BR2_OPTS="BR2_CCACHE=y"

      - name: Upload images
        uses: actions/upload-artifact@v4
        with:
          name: images-${{ github.sha }}
          path: output/images/
          retention-days: 14
```

### GitLab CI equivalent snippet

```yaml
# .gitlab-ci.yml (excerpt)
buildroot:build:
  stage: build
  image: ubuntu:22.04
  before_script:
    - apt-get update -qq && apt-get install -y bc build-essential cpio
        libncurses-dev rsync unzip wget python3
  script:
    - make defconfig DEFCONFIG=alpha_defconfig
    - make build
  cache:
    key: br2-dl-${CI_COMMIT_REF_SLUG}
    paths:
      - dl/
  artifacts:
    paths:
      - output/images/
    expire_in: 1 week
```

### Pinning Buildroot with Git submodule

```bash
# Initial setup — pin to a stable tag
git submodule add https://git.buildroot.net/buildroot buildroot
cd buildroot
git checkout 2024.02.3      # LTS release
cd ..
git add .gitmodules buildroot
git commit -m "chore: pin Buildroot 2024.02.3"

# Updating Buildroot later
cd buildroot
git fetch --tags
git checkout 2024.08         # new release
cd ..
git add buildroot
git commit -m "chore: upgrade Buildroot to 2024.08"
```

---

## 11. Common Pitfalls and Best Practices

### Pitfall 1 — Absolute paths in `Config.in`

```kconfig
# WRONG — breaks when layer is moved or used by another developer
source "/home/alice/layers/platform/package/foo/Config.in"

# CORRECT — always use the layer path variable
source "$BR2_EXTERNAL_PLATFORM_LAYER_PATH/package/foo/Config.in"
```

### Pitfall 2 — Duplicate layer names

When stacking, each `external.desc` name must be globally unique. Buildroot will
print an explicit error if two layers share the same name:

```
ERROR: BR2_EXTERNAL_NAMES has duplicate entries: MYAPP
```

### Pitfall 3 — Forgetting `$(sort $(wildcard ...))` in `external.mk`

Without `sort`, the inclusion order is filesystem-dependent and
non-deterministic across different host systems.

```makefile
# WRONG — non-deterministic ordering
include $(wildcard $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/*/*.mk)

# CORRECT — deterministic, sorted
include $(sort $(wildcard $(BR2_EXTERNAL_PLATFORM_LAYER_PATH)/package/*/*.mk))
```

### Pitfall 4 — Committing `$(O)/` to version control

The build output directory contains generated files including the persisted
`.br2-external.mk`. Always add `output/` (or whatever your `O=` path is)
to `.gitignore`.

```gitignore
# .gitignore
output/
*.o
*.a
```

### Pitfall 5 — Using `BR2_EXTERNAL` without `O=` in CI

Without an explicit `O=`, Buildroot writes output into the source tree. In CI
this pollutes the workspace and causes cache pollution.

```bash
# Always specify an explicit output directory in CI
make BR2_EXTERNAL=/layers/platform O=/tmp/br-build alpha_defconfig
make O=/tmp/br-build
```

### Best Practices Summary

```
  ┌─────────────────────────────────────────────────────┐
  │             BR2_EXTERNAL Best Practices             │
  ├────────────────────────────┬────────────────────────┤
  │ Always use                 │ Never do               │
  ├────────────────────────────┼────────────────────────┤
  │ $BR2_EXTERNAL_*_PATH       │ Hardcoded paths        │
  │ $(sort $(wildcard ...))    │ Unsorted wildcard      │
  │ Explicit O= in CI          │ In-tree builds         │
  │ Git submodule for BR       │ Copying BR into repo   │
  │ Unique layer names         │ Duplicate names        │
  │ .gitignore for output/     │ Committing build output│
  │ Pinned BR version          │ Floating HEAD          │
  └────────────────────────────┴────────────────────────┘
```

---

## 12. Summary

The `BR2_EXTERNAL` mechanism is the **cornerstone of professional, maintainable
Buildroot-based embedded Linux projects**. It cleanly separates the upstream
Buildroot tree — which you do not own and should not modify — from the
project-specific code, board support, and packages that you do.

The three mandatory files form a minimal but complete integration contract:

- **`external.desc`** — names the layer and gives Buildroot a unique path variable.
- **`external.mk`** — hooks the layer's package makefiles into the Buildroot build graph.
- **`Config.in`** — exposes the layer's packages to `menuconfig` via Kconfig.

Stacking multiple `BR2_EXTERNAL` paths with `:` allows organizations to build a
**layered architecture**: shared infrastructure layers (drivers, libraries, daemons)
coexist cleanly with product-specific layers, each in its own versioned repository.

For CI, the recommended pattern is:

1. Pin Buildroot as a **git submodule** at a known stable tag.
2. Place each layer in a separate directory (or separate repository, referenced as a submodule).
3. Use a thin top-level `Makefile` wrapper that assembles the `BR2_EXTERNAL` path.
4. Cache the `dl/` download directory and `ccache` between CI runs.
5. Always pass an explicit `O=` output directory to keep build artifacts out of the source tree.

The C, C++, and Rust examples in this document illustrate that all three language
ecosystems integrate naturally: C via `generic-package` with a hand-written
`Makefile`, C++ via `cmake-package`, and Rust via `cargo-package` — in every case
the cross-compilation toolchain is handled transparently by Buildroot once the
package is correctly declared inside the external layer.

> **The result**: A reproducible, upgrade-safe, multi-product embedded Linux build
> system where the boundary between "Buildroot" and "your code" is always crystal clear.

---

*End of Topic 24 — BR2_EXTERNAL Mechanism*