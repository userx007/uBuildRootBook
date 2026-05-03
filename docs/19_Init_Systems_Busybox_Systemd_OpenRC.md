# 19. Init Systems in Buildroot: BusyBox init, systemd, and OpenRC

---

## Table of Contents

1. [Introduction](#introduction)
2. [Boot Sequence Overview (ASCII)](#boot-sequence-overview)
3. [BusyBox init](#busybox-init)
   - [Selecting BusyBox init in Buildroot](#selecting-busybox-init-in-buildroot)
   - [inittab Syntax and Configuration](#inittab-syntax-and-configuration)
   - [C Example: Writing a Supervised Service](#c-example-writing-a-supervised-service)
   - [Rust Example: BusyBox-compatible Init Helper](#rust-example-busybox-compatible-init-helper)
4. [systemd](#systemd)
   - [Selecting systemd in Buildroot](#selecting-systemd-in-buildroot)
   - [Unit File Types and Structure](#unit-file-types-and-structure)
   - [C++ Example: DBus-activated systemd Service](#c-example-dbus-activated-systemd-service)
   - [Rust Example: systemd-notify Watchdog Service](#rust-example-systemd-notify-watchdog-service)
   - [Minimising Boot Time with systemd](#minimising-boot-time-with-systemd)
5. [OpenRC](#openrc)
   - [Selecting OpenRC in Buildroot](#selecting-openrc-in-buildroot)
   - [Writing OpenRC Init Scripts](#writing-openrc-init-scripts)
   - [C Example: OpenRC-supervised Daemon](#c-example-openrc-supervised-daemon)
   - [Rust Example: OpenRC Service with supervise-daemon](#rust-example-openrc-service-with-supervise-daemon)
6. [Comparison Table](#comparison-table)
7. [Boot Time Optimisation Techniques](#boot-time-optimisation-techniques)
8. [Summary](#summary)

---

## Introduction

In an embedded Linux system built with Buildroot, the **init system** is PID 1 — the very first
user-space process spawned by the kernel. Every other process on the system is a direct or indirect
child of PID 1. Selecting the right init system is therefore one of the most consequential decisions
you make when designing a Buildroot target image.

Buildroot supports three mainstream init systems out of the box:

- **BusyBox init** — ultra-lightweight, reads `/etc/inittab`, no parallelism, ideal for tiny
  footprint devices.
- **systemd** — feature-rich, parallel start-up, socket activation, journald logging, cgroup
  accounting; the de-facto standard for modern Linux distributions.
- **OpenRC** — dependency-based, shell-script oriented, lighter than systemd, popular in Alpine
  Linux and Gentoo.

Each has a radically different philosophy, API surface, and resource budget.

---

## Boot Sequence Overview

```
Power-on / Reset
      |
      v
+---------------------+
|   CPU Boot ROM      |
+---------------------+
      |
      v
+---------------------+
| First-Stage Loader  |  (e.g. U-Boot SPL / barebox)
+---------------------+
      |
      v
+---------------------+
| Second-Stage Loader |  (e.g. U-Boot proper)
+---------------------+
      |  Loads kernel + device tree + initramfs
      v
+=====================+
|   Linux Kernel      |
|  - Decompress       |
|  - Setup memory     |
|  - Mount rootfs     |
|  - exec /sbin/init  |  <-- PID 1 handed off here
+=====================+
      |
      +----------------+----------------+
      |                |                |
      v                v                v
 +---------+    +-----------+    +----------+
 | BusyBox |    |  systemd  |    |  OpenRC  |
 |  init   |    |           |    |          |
 | inittab |    | unit files|    | rc scripts|
 +---------+    +-----------+    +----------+
      |                |                |
      v                v                v
  Services         Services          Services
  (sequential)    (parallel)    (dep-ordered)
```

### Detailed systemd Parallel Start-up

```
  PID 1 (systemd)
       |
       |-- default.target
             |
             +-- multi-user.target
             |         |
             |         +-- network.target ----+
             |         |                      |
             |         +-- syslog.service      |
             |         |                      v
             |         +-- myapp.service  [socket activated]
             |
             +-- local-fs.target
                       |
                       +-- /etc/fstab mounts (parallel)

  Time axis:
  t=0ms  |==kernel==|
  t=300ms            |=systemd PID1=|
  t=320ms                           |=local-fs=|=network=|
  t=400ms                                      |=syslog==|
  t=410ms                                      |=myapp===|-->READY
```

### BusyBox init Sequential Start-up

```
  /sbin/init (BusyBox)
       |
       | reads /etc/inittab line by line
       |
       +--> sysinit  : /etc/init.d/rcS      (runs ONCE at boot)
       |         |
       |         +--> mount /proc /sys /dev
       |         +--> ifconfig lo up
       |         +--> start daemons sequentially
       |              [daemon A] --> [daemon B] --> [daemon C]
       |
       +--> respawn : /sbin/getty           (restarts on exit)
       +--> respawn : /usr/sbin/dropbear    (restarts on exit)
       |
       +--> shutdown: /etc/init.d/rcK       (runs on halt)
```

---

## BusyBox init

### Selecting BusyBox init in Buildroot

BusyBox init is the **default** in Buildroot. Confirm or switch in menuconfig:

```
System configuration
  └─> Init system  (BusyBox)
```

BusyBox itself is enabled via:

```
Target packages
  └─> BusyBox
        └─> [*] BusyBox
```

The BusyBox configuration (`.config` fragment) must include:

```
CONFIG_INIT=y
CONFIG_FEATURE_USE_INITTAB=y
CONFIG_FEATURE_KILL_REMOVED=y
CONFIG_GETTY=y
```

### inittab Syntax and Configuration

BusyBox reads `/etc/inittab` at startup. Each line follows the format:

```
<id>:<runlevels>:<action>:<process>
```

| Field       | Meaning                                                              |
|-------------|----------------------------------------------------------------------|
| `id`        | Console device suffix (e.g. `ttyS0`) or a label                     |
| `runlevels` | Ignored by BusyBox init (SysV artefact)                              |
| `action`    | One of the keywords below                                            |
| `process`   | Command to execute                                                   |

**Action keywords:**

```
sysinit   - Run once before everything else (filesystem setup)
wait      - Run once; init waits for it to exit before continuing
once      - Run once; init does NOT wait
respawn   - Restart automatically whenever it exits
askfirst  - Like respawn but prompts "press Enter" first
shutdown  - Run on system shutdown
restart   - Run on SIGHUP to init (re-exec)
ctrlaltdel- Run when Ctrl-Alt-Del is pressed
```

**Minimal inittab example:**

```
# /etc/inittab — BusyBox init

# Mount virtual filesystems and run startup scripts
null::sysinit:/etc/init.d/rcS

# Start a shell on the serial console
ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100

# Start a shell on the main console (HDMI/framebuffer)
tty1::respawn:/sbin/getty 38400 tty1

# What to do on Ctrl-Alt-Delete
::ctrlaltdel:/sbin/reboot

# What to do when init shuts the system down
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a
```

**`/etc/init.d/rcS` — the main startup script:**

```sh
#!/bin/sh
# Mount proc/sys/devtmpfs
mount -t proc     proc     /proc
mount -t sysfs    sysfs    /sys
mount -t devtmpfs devtmpfs /dev

# Bring up loopback
ip link set lo up

# Run all S* scripts in /etc/init.d in alphabetical order
for script in /etc/init.d/S??*; do
    [ -x "$script" ] && "$script" start
done
```

**Service script skeleton (`/etc/init.d/S50myapp`):**

```sh
#!/bin/sh
DAEMON=/usr/bin/myapp
PIDFILE=/var/run/myapp.pid
NAME=myapp

start() {
    printf "Starting %s: " "$NAME"
    start-stop-daemon -S -q -b -m -p "$PIDFILE" -x "$DAEMON"
    echo "OK"
}

stop() {
    printf "Stopping %s: " "$NAME"
    start-stop-daemon -K -q -p "$PIDFILE"
    rm -f "$PIDFILE"
    echo "OK"
}

case "$1" in
    start)  start  ;;
    stop)   stop   ;;
    restart) stop; start ;;
    *) echo "Usage: $0 {start|stop|restart}"; exit 1 ;;
esac
```

---

### C Example: Writing a Supervised Service

A minimal daemon designed to work with BusyBox `start-stop-daemon` and `respawn`:

```c
/* myapp.c — BusyBox-init-compatible daemon
 *
 * Compile (cross):
 *   ${CROSS_COMPILE}gcc -Wall -O2 -o myapp myapp.c
 *
 * Place in /usr/bin/myapp; use inittab respawn or S50 script.
 */

#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <syslog.h>
#include <string.h>
#include <errno.h>
#include <sys/stat.h>
#include <fcntl.h>

/* ------------------------------------------------------------------ */
/* Signal handling                                                      */
/* ------------------------------------------------------------------ */

static volatile sig_atomic_t g_running = 1;
static volatile sig_atomic_t g_reload  = 0;

static void sig_handler(int signo)
{
    switch (signo) {
    case SIGTERM:
    case SIGINT:
        g_running = 0;
        break;
    case SIGHUP:
        g_reload = 1;
        break;
    default:
        break;
    }
}

/* ------------------------------------------------------------------ */
/* Daemonise — double-fork, setsid, redirect stdio                     */
/* ------------------------------------------------------------------ */

static void daemonise(const char *pidfile)
{
    pid_t pid;
    int   fd;

    /* First fork */
    pid = fork();
    if (pid < 0) { perror("fork"); exit(EXIT_FAILURE); }
    if (pid > 0) { exit(EXIT_SUCCESS); }   /* parent exits */

    /* New session */
    if (setsid() < 0) { perror("setsid"); exit(EXIT_FAILURE); }

    /* Second fork — prevent re-acquiring a terminal */
    pid = fork();
    if (pid < 0) { perror("fork"); exit(EXIT_FAILURE); }
    if (pid > 0) { exit(EXIT_SUCCESS); }

    /* Change working directory */
    if (chdir("/") < 0) { perror("chdir"); exit(EXIT_FAILURE); }

    /* Redirect stdin/stdout/stderr to /dev/null */
    fd = open("/dev/null", O_RDWR);
    if (fd >= 0) {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > 2) close(fd);
    }

    /* Write PID file */
    if (pidfile) {
        FILE *fp = fopen(pidfile, "w");
        if (fp) {
            fprintf(fp, "%d\n", (int)getpid());
            fclose(fp);
        }
    }
}

/* ------------------------------------------------------------------ */
/* Load / reload configuration                                          */
/* ------------------------------------------------------------------ */

static void load_config(void)
{
    syslog(LOG_INFO, "Loading configuration");
    /* TODO: parse /etc/myapp.conf */
}

/* ------------------------------------------------------------------ */
/* Main work loop                                                       */
/* ------------------------------------------------------------------ */

static void run_loop(void)
{
    unsigned int cycle = 0;

    while (g_running) {
        if (g_reload) {
            g_reload = 0;
            load_config();
        }

        /* Application work here */
        syslog(LOG_DEBUG, "Work cycle %u", cycle++);

        sleep(5);
    }
}

/* ------------------------------------------------------------------ */
/* Entry point                                                          */
/* ------------------------------------------------------------------ */

int main(int argc, char *argv[])
{
    const char *pidfile = "/var/run/myapp.pid";
    int         foreground = 0;

    /* Simple argument parsing */
    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "-f") == 0) foreground = 1;
        if (strcmp(argv[i], "-p") == 0 && i+1 < argc) pidfile = argv[++i];
    }

    /* Install signal handlers BEFORE daemonising */
    signal(SIGTERM, sig_handler);
    signal(SIGINT,  sig_handler);
    signal(SIGHUP,  sig_handler);

    if (!foreground)
        daemonise(pidfile);

    openlog("myapp", LOG_PID | LOG_CONS, LOG_DAEMON);
    syslog(LOG_INFO, "myapp started (PID %d)", (int)getpid());

    load_config();
    run_loop();

    syslog(LOG_INFO, "myapp stopped");
    closelog();

    return EXIT_SUCCESS;
}
```

---

### Rust Example: BusyBox-compatible Init Helper

A Rust daemon that integrates cleanly with BusyBox `respawn` semantics — it handles signals,
logs to syslog via the `syslog` crate, and writes its own PID file:

```toml
# Cargo.toml
[package]
name    = "busybox-service"
version = "0.1.0"
edition = "2021"

[dependencies]
syslog  = "6"
signal-hook = "0.3"

[profile.release]
opt-level = "z"   # minimise binary size for embedded
strip     = true
lto       = true
```

```rust
// src/main.rs — Rust daemon for BusyBox init respawn

use signal_hook::{
    consts::{SIGTERM, SIGHUP, SIGINT},
    iterator::Signals,
};
use std::{
    fs,
    io::Write,
    path::Path,
    sync::{
        atomic::{AtomicBool, Ordering},
        Arc,
    },
    thread,
    time::Duration,
};
use syslog::{BasicLogger, Facility, Formatter3164};

/* ------------------------------------------------------------------ */
/* PID file management                                                  */
/* ------------------------------------------------------------------ */

fn write_pid(path: &str) -> std::io::Result<()> {
    let pid = std::process::id();
    let mut f = fs::File::create(path)?;
    writeln!(f, "{}", pid)?;
    Ok(())
}

fn remove_pid(path: &str) {
    let _ = fs::remove_file(path);
}

/* ------------------------------------------------------------------ */
/* Application logic                                                    */
/* ------------------------------------------------------------------ */

fn do_work(cycle: u64) {
    // Replace with real application logic
    log::debug!("Work cycle {}", cycle);
}

/* ------------------------------------------------------------------ */
/* Entry point                                                          */
/* ------------------------------------------------------------------ */

fn main() {
    // Initialise syslog
    let formatter = Formatter3164 {
        facility: Facility::LOG_DAEMON,
        hostname: None,
        process:  "busybox-service".into(),
        pid:      std::process::id(),
    };
    let logger = syslog::unix(formatter).expect("Could not connect to syslog");
    log::set_boxed_logger(Box::new(BasicLogger::new(logger)))
        .expect("log::set_boxed_logger failed");
    log::set_max_level(log::LevelFilter::Debug);

    let pidfile = "/var/run/busybox-service.pid";
    write_pid(pidfile).unwrap_or_else(|e| log::warn!("PID file write failed: {}", e));

    log::info!("busybox-service started (PID {})", std::process::id());

    // Shared running flag
    let running = Arc::new(AtomicBool::new(true));
    let reload  = Arc::new(AtomicBool::new(false));

    // Signal thread
    let running_sig = Arc::clone(&running);
    let reload_sig  = Arc::clone(&reload);
    let mut signals = Signals::new([SIGTERM, SIGINT, SIGHUP])
        .expect("Signals::new failed");

    thread::spawn(move || {
        for sig in &mut signals {
            match sig {
                SIGTERM | SIGINT => {
                    log::info!("Caught termination signal {}", sig);
                    running_sig.store(false, Ordering::SeqCst);
                }
                SIGHUP => {
                    log::info!("Caught SIGHUP — scheduling reload");
                    reload_sig.store(true, Ordering::SeqCst);
                }
                _ => {}
            }
        }
    });

    // Main work loop
    let mut cycle: u64 = 0;
    while running.load(Ordering::SeqCst) {
        if reload.swap(false, Ordering::SeqCst) {
            log::info!("Reloading configuration");
            // TODO: reload config
        }

        do_work(cycle);
        cycle += 1;

        thread::sleep(Duration::from_secs(5));
    }

    log::info!("busybox-service stopping");
    remove_pid(pidfile);
}
```

Cross-compile for ARM:

```bash
# Add the target once
rustup target add armv7-unknown-linux-musleabihf

# Build a fully static binary (musl) — ideal for Buildroot
cargo build --release --target armv7-unknown-linux-musleabihf
```

---

## systemd

### Selecting systemd in Buildroot

```
System configuration
  └─> Init system  (systemd)

Target packages
  └─> System tools
        └─> [*] systemd
              └─> [*] enable kdbus support   (optional)
              └─> [*] install in /usr        (recommended)
```

Because systemd requires a relatively modern kernel (≥ 3.15, ideally ≥ 5.x) and several kernel
config options, Buildroot will warn if these are missing:

```
CONFIG_CGROUPS=y
CONFIG_INOTIFY_USER=y
CONFIG_SIGNALFD=y
CONFIG_TIMERFD=y
CONFIG_EPOLL=y
CONFIG_NET=y
CONFIG_UNIX=y
CONFIG_SYSFS=y
CONFIG_PROC_FS=y
CONFIG_DEVTMPFS=y
CONFIG_TMPFS=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_SECCOMP=y        # recommended
CONFIG_FANOTIFY=y       # for systemd-udev
```

### Unit File Types and Structure

```
Unit file types:
  .service  — a process or pool of processes
  .socket   — socket activation (starts service on first connection)
  .target   — grouping / synchronisation point (like SysV runlevels)
  .timer    — cron-like scheduling
  .mount    — /etc/fstab equivalent
  .path     — file-system path watch (inotify-based)
  .device   — udev device events
  .slice    — cgroup resource control boundary
  .scope    — externally created processes

Unit file search path (Buildroot):
  /usr/lib/systemd/system/    (package-installed, read-only)
  /etc/systemd/system/        (site-local overrides — higher priority)
  /run/systemd/system/        (runtime, lost on reboot)
```

**Minimal service unit:**

```ini
# /usr/lib/systemd/system/myapp.service

[Unit]
Description=My Embedded Application
Documentation=man:myapp(8)
After=network.target
Wants=network.target

[Service]
Type=notify
ExecStart=/usr/bin/myapp --foreground
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
WatchdogSec=30s

# Security hardening
User=myapp
Group=myapp
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

**Socket-activated service pair:**

```ini
# /usr/lib/systemd/system/myapp.socket
[Unit]
Description=My App Socket

[Socket]
ListenStream=/run/myapp.sock
SocketMode=0660
SocketUser=myapp
SocketGroup=myapp

[Install]
WantedBy=sockets.target
```

**Enabling at build time via Buildroot `rootfs-overlay`:**

```
board/myboard/rootfs-overlay/
  usr/lib/systemd/system/
    myapp.service
    myapp.socket
  etc/systemd/system/
    multi-user.target.wants/
      myapp.service -> /usr/lib/systemd/system/myapp.service
    sockets.target.wants/
      myapp.socket  -> /usr/lib/systemd/system/myapp.socket
```

---

### C++ Example: DBus-activated systemd Service

```cpp
// myapp_service.cpp — systemd Type=notify service with sd-daemon integration
//
// Build:
//   pkg-config --cflags --libs libsystemd
//   ${CROSS_COMPILE}g++ -std=c++17 -Wall -O2 \
//       $(pkg-config --cflags libsystemd) \
//       myapp_service.cpp -o myapp_service \
//       $(pkg-config --libs libsystemd)
//
// Requires: BR2_PACKAGE_SYSTEMD=y (provides libsystemd)

#include <systemd/sd-daemon.h>
#include <systemd/sd-journal.h>

#include <atomic>
#include <chrono>
#include <csignal>
#include <cstdlib>
#include <cstring>
#include <iostream>
#include <stdexcept>
#include <thread>

/* ------------------------------------------------------------------ */
/* Signal management                                                    */
/* ------------------------------------------------------------------ */

namespace {
    std::atomic<bool> g_running{true};
    std::atomic<bool> g_reload{false};
}

extern "C" void signal_handler(int sig) noexcept
{
    if (sig == SIGTERM || sig == SIGINT)
        g_running.store(false, std::memory_order_relaxed);
    else if (sig == SIGHUP)
        g_reload.store(true,  std::memory_order_relaxed);
}

/* ------------------------------------------------------------------ */
/* Application class                                                    */
/* ------------------------------------------------------------------ */

class Application {
public:
    Application()
    {
        sd_journal_print(LOG_INFO, "Application constructing");
    }

    ~Application()
    {
        sd_journal_print(LOG_INFO, "Application destructing");
    }

    void load_config()
    {
        sd_journal_print(LOG_INFO, "Loading configuration");
        // TODO: parse /etc/myapp.conf
    }

    void run()
    {
        using namespace std::chrono_literals;

        // Notify systemd: initialisation complete
        sd_notify(0, "READY=1\n"
                     "STATUS=Running main loop\n");

        uint64_t cycle = 0;

        while (g_running.load(std::memory_order_relaxed)) {

            // Reset watchdog timer (must happen within WatchdogSec=)
            sd_notify(0, "WATCHDOG=1");

            if (g_reload.exchange(false, std::memory_order_relaxed)) {
                sd_notify(0, "RELOADING=1\n"
                             "STATUS=Reloading configuration\n");
                load_config();
                sd_notify(0, "READY=1\n"
                             "STATUS=Running main loop\n");
            }

            // Real work goes here
            sd_journal_print(LOG_DEBUG, "Cycle %" PRIu64, cycle++);

            std::this_thread::sleep_for(10s);
        }

        // Notify systemd: shutting down
        sd_notify(0, "STOPPING=1\n"
                     "STATUS=Shutting down\n");
    }
};

/* ------------------------------------------------------------------ */
/* Entry point                                                          */
/* ------------------------------------------------------------------ */

int main()
{
    std::signal(SIGTERM, signal_handler);
    std::signal(SIGINT,  signal_handler);
    std::signal(SIGHUP,  signal_handler);

    try {
        Application app;
        app.load_config();
        app.run();
    } catch (const std::exception &ex) {
        sd_journal_print(LOG_ERR, "Fatal exception: %s", ex.what());
        // Tell systemd the service failed
        sd_notifyf(0, "ERRNO=%d", EXIT_FAILURE);
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

**Corresponding unit file:**

```ini
# /usr/lib/systemd/system/myapp_service.service

[Unit]
Description=C++ systemd-notify Service Example
After=local-fs.target

[Service]
Type=notify
ExecStart=/usr/bin/myapp_service
Restart=on-failure
WatchdogSec=60s
NotifyAccess=main

StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp_service

[Install]
WantedBy=multi-user.target
```

---

### Rust Example: systemd-notify Watchdog Service

```toml
# Cargo.toml
[package]
name    = "systemd-service"
version = "0.1.0"
edition = "2021"

[dependencies]
systemd  = { version = "0.10", features = ["journal"] }
signal-hook = "0.3"

[profile.release]
opt-level = "z"
strip     = true
lto       = true
```

```rust
// src/main.rs — Rust service integrated with systemd sd_notify + watchdog

use signal_hook::{consts::{SIGTERM, SIGINT, SIGHUP}, iterator::Signals};
use std::{
    sync::{atomic::{AtomicBool, Ordering}, Arc},
    thread,
    time::Duration,
};
use systemd::{daemon, journal};

fn main() {
    // Log to the systemd journal instead of stderr
    journal::print(6 /*LOG_INFO*/, "systemd-service starting");

    let running = Arc::new(AtomicBool::new(true));
    let reload  = Arc::new(AtomicBool::new(false));

    // Signal handler thread
    {
        let running = Arc::clone(&running);
        let reload  = Arc::clone(&reload);
        let mut signals = Signals::new([SIGTERM, SIGINT, SIGHUP])
            .expect("Signals::new");

        thread::spawn(move || {
            for sig in &mut signals {
                match sig {
                    SIGTERM | SIGINT => {
                        journal::print(5 /*LOG_NOTICE*/,
                            &format!("Signal {} received, stopping", sig));
                        running.store(false, Ordering::SeqCst);
                    }
                    SIGHUP => {
                        journal::print(6, "SIGHUP — reloading");
                        reload.store(true, Ordering::SeqCst);
                    }
                    _ => {}
                }
            }
        });
    }

    // Query watchdog interval from systemd
    let watchdog_usec = daemon::watchdog_enabled(false)
        .unwrap_or(0);
    let watchdog_interval = if watchdog_usec > 0 {
        Duration::from_micros(watchdog_usec / 2) // pet at half the interval
    } else {
        Duration::from_secs(30)
    };

    journal::print(6,
        &format!("Watchdog interval: {}ms", watchdog_interval.as_millis()));

    // Notify systemd: ready
    daemon::notify(false, [("READY", "1"), ("STATUS", "Running")].iter())
        .expect("sd_notify READY");

    let mut cycle: u64 = 0;
    let mut last_watchdog = std::time::Instant::now();

    while running.load(Ordering::SeqCst) {
        // Pet the watchdog if it's time
        if last_watchdog.elapsed() >= watchdog_interval {
            daemon::notify(false, [("WATCHDOG", "1")].iter()).ok();
            last_watchdog = std::time::Instant::now();
        }

        if reload.swap(false, Ordering::SeqCst) {
            daemon::notify(false, [("RELOADING", "1")].iter()).ok();
            journal::print(6, "Reloading configuration");
            // TODO: reload config
            daemon::notify(false, [("READY", "1")].iter()).ok();
        }

        // Application work
        journal::print(7 /*LOG_DEBUG*/,
            &format!("Cycle {}", cycle));
        cycle += 1;

        thread::sleep(Duration::from_secs(5));
    }

    daemon::notify(false, [("STOPPING", "1")].iter()).ok();
    journal::print(6, "systemd-service stopped");
}
```

---

### Minimising Boot Time with systemd

**Analyse current boot:**

```bash
# On the target
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain

# Generate an SVG boot timeline (view on host)
systemd-analyze plot > /tmp/boot.svg
```

**ASCII representation of boot critical chain:**

```
graphical.target @3.240s
└─multi-user.target @3.239s
  └─myapp.service @2.891s +0.348s
    └─network.target @2.889s
      └─eth0.network @1.203s +1.686s
        └─systemd-networkd.service @1.128s +0.073s
          └─systemd-udevd.service @0.921s +0.201s
            └─systemd-tmpfiles-setup-dev.service @0.841s +0.078s
              └─local-fs.target @0.830s
                └─squashfs.mount @0.419s +0.411s
                  └─-.mount @0.388s
                    └─systemd-journald.service @0.292s +0.095s
                      └─sysinit.target @0.288s
```

**Key optimisation techniques:**

```ini
# 1. Disable unused units
systemctl disable avahi-daemon bluetooth ModemManager

# 2. Use socket activation — service starts on first connection
# myapp.socket starts myapp.service only when needed

# 3. Reduce journal overhead in /etc/systemd/journald.conf
[Journal]
Storage=volatile       # RAM only — no disk I/O
Compress=no
RateLimitIntervalSec=0

# 4. Drop to emergency.target to audit what is actually needed
# kernel cmdline: systemd.unit=emergency.target

# 5. Mask unused targets
systemctl mask systemd-update-utmp.service     \
               systemd-random-seed.service     \
               systemd-resolved.service        \
               systemd-timesyncd.service

# 6. Reduce default timeout
# /etc/systemd/system.conf
[Manager]
DefaultTimeoutStartSec=10s
DefaultTimeoutStopSec=5s
```

---

## OpenRC

### Selecting OpenRC in Buildroot

```
System configuration
  └─> Init system  (OpenRC)

Target packages
  └─> System tools
        └─> [*] openrc
```

Kernel requirements for OpenRC are lighter than systemd — essentially just `/proc`, `/sys`,
and `devtmpfs`.

OpenRC **runlevel layout:**

```
Runlevels (directories under /etc/runlevels/):
  sysinit   — very early: mount virtual filesystems
  boot      — early services: hostname, sysctl, fsck, swap
  default   — normal operation services
  nonetwork — like default but without networking
  single    — single-user maintenance mode
  shutdown  — graceful power-off
  reboot    — graceful reboot
```

**Adding a service to a runlevel:**

```bash
rc-update add myapp default
rc-update del myapp default
rc-update show              # list all enabled services
```

### Writing OpenRC Init Scripts

OpenRC scripts live in `/etc/init.d/`. They source `/lib/rc/sh/openrc-run.sh` which provides the
`ebegin`/`eend` logging helpers and dependency resolution.

**Full-featured OpenRC service script:**

```sh
#!/sbin/openrc-run
# /etc/init.d/myapp — OpenRC service script

description="My Embedded Application"
command="/usr/bin/myapp"
command_args="--foreground"
command_user="myapp:myapp"
command_background=true         # daemonise via supervise-daemon
pidfile="/var/run/myapp.pid"
output_log="/var/log/myapp.log"
error_log="/var/log/myapp.err"

# Dependency declarations
depend() {
    need net          # won't start without network
    use logger        # start syslog first if available, but not mandatory
    after firewall    # start after firewall if it exists
    keyword -shutdown # do not stop during shutdown sequence
}

# Optional: custom start hook
start_pre() {
    # Ensure config exists
    if [ ! -f /etc/myapp.conf ]; then
        eerror "Configuration file /etc/myapp.conf not found"
        return 1
    fi
    checkpath --directory --owner myapp:myapp --mode 0755 /var/lib/myapp
    checkpath --file      --owner myapp:myapp --mode 0640 /var/log/myapp.log
}

# Optional: after service is running
start_post() {
    ewarn "myapp started — check /var/log/myapp.log"
}

# Optional: before stopping
stop_pre() {
    einfo "Draining connections before stop..."
    sleep 1
}
```

**Minimal script variant (using `supervise-daemon`):**

```sh
#!/sbin/openrc-run
description="Minimal OpenRC service"
supervisor=supervise-daemon          # OpenRC's built-in supervisor
command=/usr/bin/myapp
command_args="--foreground --config /etc/myapp.conf"
command_user=myapp
pidfile=/var/run/myapp.pid

depend() {
    need net
    after syslog
}
```

---

### C Example: OpenRC-supervised Daemon

A foreground daemon designed to integrate with OpenRC's `supervise-daemon` restart policy:

```c
/* openrc_daemon.c — designed for OpenRC supervisor=supervise-daemon
 *
 * Run in foreground; OpenRC will restart it if it exits unexpectedly.
 * Log via syslog; OpenRC redirects stdout/stderr for you.
 *
 * Compile:
 *   ${CROSS_COMPILE}gcc -Wall -O2 -o openrc_daemon openrc_daemon.c
 */

#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <syslog.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

/* ------------------------------------------------------------------ */
/* Signal flags                                                         */
/* ------------------------------------------------------------------ */

static volatile sig_atomic_t g_terminate = 0;
static volatile sig_atomic_t g_reload    = 0;

static void handle_signal(int s)
{
    if (s == SIGTERM || s == SIGINT)  g_terminate = 1;
    else if (s == SIGHUP)             g_reload    = 1;
}

/* ------------------------------------------------------------------ */
/* Config                                                               */
/* ------------------------------------------------------------------ */

#define CONFIG_PATH "/etc/openrc_daemon.conf"

typedef struct {
    int  interval_sec;
    char log_prefix[64];
} Config;

static Config g_cfg = { .interval_sec = 5, .log_prefix = "openrc-daemon" };

static void reload_config(void)
{
    FILE *fp = fopen(CONFIG_PATH, "r");
    if (!fp) {
        syslog(LOG_WARNING, "Cannot open %s: %s", CONFIG_PATH, strerror(errno));
        return;
    }
    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        if (sscanf(line, "interval=%d",      &g_cfg.interval_sec)  == 1) continue;
        if (sscanf(line, "prefix=%63s",       g_cfg.log_prefix)    == 1) continue;
    }
    fclose(fp);
    syslog(LOG_INFO, "Config reloaded: interval=%d prefix=%s",
           g_cfg.interval_sec, g_cfg.log_prefix);
}

/* ------------------------------------------------------------------ */
/* OpenRC readiness notification via /run/openrc/started/<name>        */
/* Alternatively write to NOTIFY_SOCKET if set (not standard OpenRC)  */
/* ------------------------------------------------------------------ */

static void notify_ready(void)
{
    const char *rc_svcdir = getenv("RC_SVCDIR");
    if (!rc_svcdir) rc_svcdir = "/run/openrc";

    char started_path[256];
    snprintf(started_path, sizeof(started_path),
             "%s/started/openrc_daemon", rc_svcdir);

    /* Touch the file to signal OpenRC we are ready */
    int fd = open(started_path, O_CREAT | O_WRONLY | O_TRUNC, 0644);
    if (fd >= 0) close(fd);
}

/* ------------------------------------------------------------------ */
/* Main loop                                                            */
/* ------------------------------------------------------------------ */

int main(void)
{
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handle_signal;
    sigaction(SIGTERM, &sa, NULL);
    sigaction(SIGINT,  &sa, NULL);
    sigaction(SIGHUP,  &sa, NULL);

    openlog("openrc_daemon", LOG_PID, LOG_DAEMON);
    syslog(LOG_INFO, "Starting (PID %d)", (int)getpid());

    reload_config();
    notify_ready();

    unsigned long cycle = 0;

    while (!g_terminate) {
        if (g_reload) {
            g_reload = 0;
            reload_config();
        }

        syslog(LOG_DEBUG, "[%s] cycle %lu", g_cfg.log_prefix, cycle++);
        sleep((unsigned)g_cfg.interval_sec);
    }

    syslog(LOG_INFO, "Stopping cleanly");
    closelog();
    return EXIT_SUCCESS;
}
```

---

### Rust Example: OpenRC Service with supervise-daemon

```toml
# Cargo.toml
[package]
name    = "openrc-service"
version = "0.1.0"
edition = "2021"

[dependencies]
log        = "0.4"
env_logger = "0.11"
signal-hook = "0.3"
```

```rust
// src/main.rs — OpenRC supervise-daemon compatible Rust service

use signal_hook::{consts::{SIGTERM, SIGINT, SIGHUP}, iterator::Signals};
use std::{
    env,
    fs,
    io::Write,
    path::PathBuf,
    sync::{atomic::{AtomicBool, Ordering}, Arc},
    thread,
    time::Duration,
};

/* ------------------------------------------------------------------ */
/* OpenRC readiness touch-file                                          */
/* ------------------------------------------------------------------ */

fn notify_started(service_name: &str) {
    let rc_svcdir = env::var("RC_SVCDIR")
        .unwrap_or_else(|_| "/run/openrc".into());

    let mut path = PathBuf::from(rc_svcdir);
    path.push("started");
    path.push(service_name);

    let _ = fs::OpenOptions::new()
        .write(true)
        .create(true)
        .truncate(true)
        .open(&path);

    log::info!("Notified OpenRC (touched {:?})", path);
}

/* ------------------------------------------------------------------ */
/* Configuration (simple key=value parser)                              */
/* ------------------------------------------------------------------ */

#[derive(Debug)]
struct Config {
    interval_secs: u64,
    label: String,
}

impl Default for Config {
    fn default() -> Self {
        Self { interval_secs: 5, label: "openrc-rs".into() }
    }
}

impl Config {
    fn load(path: &str) -> Self {
        let mut cfg = Config::default();
        if let Ok(contents) = fs::read_to_string(path) {
            for line in contents.lines() {
                let line = line.trim();
                if let Some(val) = line.strip_prefix("interval=") {
                    cfg.interval_secs = val.parse().unwrap_or(5);
                }
                if let Some(val) = line.strip_prefix("label=") {
                    cfg.label = val.to_string();
                }
            }
        }
        cfg
    }
}

/* ------------------------------------------------------------------ */
/* Entry point                                                          */
/* ------------------------------------------------------------------ */

fn main() {
    env_logger::Builder::from_env(
        env_logger::Env::default().default_filter_or("info"),
    ).init();

    log::info!("openrc-service starting (PID {})", std::process::id());

    let running = Arc::new(AtomicBool::new(true));
    let reload  = Arc::new(AtomicBool::new(false));

    // Signal handling thread
    {
        let running = Arc::clone(&running);
        let reload  = Arc::clone(&reload);
        let mut signals = Signals::new([SIGTERM, SIGINT, SIGHUP])
            .expect("Signals::new");

        thread::spawn(move || {
            for sig in &mut signals {
                match sig {
                    SIGTERM | SIGINT => {
                        log::info!("Signal {} — terminating", sig);
                        running.store(false, Ordering::SeqCst);
                    }
                    SIGHUP => {
                        log::info!("SIGHUP — reload requested");
                        reload.store(true, Ordering::SeqCst);
                    }
                    _ => {}
                }
            }
        });
    }

    let config_path = "/etc/openrc-service.conf";
    let mut cfg = Config::load(config_path);
    log::info!("Config loaded: {:?}", cfg);

    // Signal OpenRC that we are ready
    notify_started("openrc-service");

    let mut cycle: u64 = 0;

    while running.load(Ordering::SeqCst) {
        if reload.swap(false, Ordering::SeqCst) {
            cfg = Config::load(config_path);
            log::info!("Config reloaded: {:?}", cfg);
        }

        log::debug!("[{}] cycle {}", cfg.label, cycle);
        cycle += 1;

        thread::sleep(Duration::from_secs(cfg.interval_secs));
    }

    log::info!("openrc-service stopped cleanly");
}
```

**Corresponding OpenRC script:**

```sh
#!/sbin/openrc-run
description="Rust OpenRC Service"
supervisor=supervise-daemon
command=/usr/bin/openrc-service
command_user=myapp
pidfile=/var/run/openrc-service.pid
output_log=/var/log/openrc-service.log
error_log=/var/log/openrc-service.err

depend() {
    need net
    after syslog logger
}
```

---

## Comparison Table

```
+------------------+------------------+------------------+------------------+
| Feature          | BusyBox init     | systemd          | OpenRC           |
+------------------+------------------+------------------+------------------+
| Binary size      | ~1 MB (BusyBox   | ~5–15 MB         | ~500 KB          |
|                  |  total)          |                  |                  |
+------------------+------------------+------------------+------------------+
| RAM usage        | <1 MB            | 10–25 MB         | 2–5 MB           |
+------------------+------------------+------------------+------------------+
| Parallel start   | No (sequential)  | Yes              | Yes (optional)   |
+------------------+------------------+------------------+------------------+
| Service restart  | Via respawn only | Restart= policy  | supervise-daemon |
+------------------+------------------+------------------+------------------+
| Logging          | syslog/klog      | journald         | syslog           |
+------------------+------------------+------------------+------------------+
| Dependencies     | Manual script    | Declarative unit | depend() shell   |
|                  |  order           |  file sections   |  function        |
+------------------+------------------+------------------+------------------+
| Resource limits  | ulimit in script | CGroup slices    | ulimit in script |
+------------------+------------------+------------------+------------------+
| Socket activate  | No               | Yes              | No               |
+------------------+------------------+------------------+------------------+
| Watchdog         | External only    | Built-in         | External only    |
+------------------+------------------+------------------+------------------+
| D-Bus            | No               | Yes (sd-bus)     | No               |
+------------------+------------------+------------------+------------------+
| Container-ready  | Limited          | Yes (nspawn)     | Limited          |
+------------------+------------------+------------------+------------------+
| Config format    | inittab (text)   | INI unit files   | Shell scripts    |
+------------------+------------------+------------------+------------------+
| Ideal for        | MCU-class Linux, | Industrial PCs,  | Mid-range SBCs,  |
|                  |  initramfs,      |  gateways,       |  Yocto alt,      |
|                  |  rescue images   |  automotive IVI  |  Alpine-style    |
+------------------+------------------+------------------+------------------+
```

---

## Boot Time Optimisation Techniques

### Universal Techniques (all init systems)

```
1. Reduce kernel to only essential drivers (loadable → built-in for initcall order)
2. Use initramfs for root filesystem (eliminates disk seek latency)
3. Mount read-only squashfs rootfs (instant — no journal replay)
4. Disable VT (console) switching if headless: CONFIG_VT=n
5. Disable kernel module loading if image is fixed: CONFIG_MODULES=n
6. Use faster clocks for UART / I2C / SPI early in boot
7. Reduce kernel command-line: drop 'quiet' for debug, add for production
   console=ttyS0,115200 rootfstype=squashfs ro loglevel=3
```

### BusyBox init Optimisation

```
Strategy: BusyBox is already minimal.
Gains come from trimming rcS script content.

✓ Replace sequential script with parallel background launches:
    /usr/bin/daemon_a &
    /usr/bin/daemon_b &
    wait                # only if needed
    
✓ Use mdev instead of udev (already part of BusyBox)
✓ Avoid sleep/polling; use hotplug events
✓ Strip the BusyBox binary with --strip-all
```

**ASCII boot timeline — BusyBox (before vs after):**

```
BEFORE optimisation (sequential):
  |=kernel=|=rcS=|=daemonA=|=daemonB=|=daemonC=|=getty=|
  0      300ms  400ms     600ms     800ms    1000ms  1200ms

AFTER optimisation (parallel background):
  |=kernel=|=rcS=|
                  |=daemonA=====|
                  |=daemonB===|
                  |=daemonC=======|
                  |=getty=|
  0      300ms  400ms            800ms
                                    ^ All services ready
```

### systemd Optimisation

```
1. systemd-analyze blame  → find the slowest unit
2. Mask units not needed for your workload
3. Use Type=notify — systemd knows EXACTLY when service is ready
4. Use socket activation — defer service start until first use
5. Set aggressive timeouts in system.conf
6. Use systemd-boot or UEFI boot (faster than GRUB)
7. Enable read-ahead: systemd-readahead (on older kernels)
8. Move /tmp to tmpfs: systemd-tmpfs already does this
9. Remove getty instances on unused VTs
```

**ASCII parallel systemd timeline:**

```
t=0   |===== Kernel ======|
t=300  |systemd PID1|
t=310               |local-fs.target|
t=320               |sysinit.target  |
t=340                                |=syslog.service=|
t=340                                |=network.service=====|
t=340                                |=dbus.service===|
t=400                                                 |=myapp.service=|READY
```

### OpenRC Optimisation

```
1. rc parallel=yes in /etc/rc.conf
2. rc_hotplug="*" to defer udev-like events
3. Remove services from boot runlevel; use default only
4. Use supervise-daemon instead of background & trick
5. Set rc_timeout_stopsec=5 for faster shutdown
```

---

## Summary

This chapter covered the three init systems available in Buildroot and how to configure, extend,
and tune each one for embedded Linux targets.

**BusyBox init** is the simplest approach: a single binary, an `inittab` file, and shell scripts.
It has zero parallel start-up capability, but for devices that boot in under two seconds from flash
and run fewer than ten services, it is perfectly adequate and almost unbeatable for footprint. The
key skills are writing well-ordered `rcS` scripts, using `start-stop-daemon` for PID management,
and understanding the `respawn` action for automatic service restart.

**systemd** offers the most powerful feature set: declarative unit files, automatic dependency
resolution, parallel start-up, socket activation, watchdog integration, cgroup-based resource
accounting, and the `sd_notify` protocol for precise readiness signalling. The cost is a larger
memory footprint (~10–25 MB) and a minimum kernel version requirement. For industrial gateways,
automotive infotainment, or any device running 20+ services, systemd's parallelism and tooling
(`systemd-analyze`, `journalctl`) more than justify the overhead. The C++ and Rust examples showed
how to use `libsystemd` / the `systemd` crate to implement proper Type=notify services with
watchdog heartbeats.

**OpenRC** occupies the middle ground: shell-script service files with a structured `depend()`
function for dependency ordering, optional parallelism, and the `supervise-daemon` built-in for
restart policies — all at a fraction of systemd's footprint. It is a natural fit for Alpine
Linux-style embedded systems where operator familiarity with shell scripts is valued and systemd's
complexity is unwanted.

Across all three systems, boot-time optimisation follows the same high-level pattern: eliminate
unnecessary work (mask/disable unused services), parallelise what can run concurrently, and use
read-only or memory-backed file systems to avoid journal replay. Combined with a lean kernel and
a squashfs root filesystem, sub-second user-space boot times are achievable even on modest
ARM Cortex-A class hardware.

---

*Part of the Buildroot Embedded Linux Programming Series — Chapter 19 of 30*