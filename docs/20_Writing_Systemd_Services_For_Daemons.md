# 20. Writing systemd Services for Embedded C++/Rust Daemons

**Structure:**
- **10 sections** with a linked table of contents
- **ASCII architecture diagrams** for every major concept: service lifecycle state machine, sd_notify communication flow, sandboxing layer stack, socket activation flow, watchdog timing diagram, and a concept map summary

**Code examples:**
- **C** — a zero-dependency `sd_notify` header-only implementation (no `libsystemd` needed for minimal footprint), plus a complete epoll-based daemon with watchdog timerfd and socket activation, plus a Buildroot `.mk` file
- **C++** — a full RAII daemon class hierarchy (`Fd`, `Watchdog`, `EventLoop`, `Daemon`) with C++20, plus a `CMakeLists.txt` for Buildroot integration
- **Rust** — async Tokio daemon using the `sd-notify` crate with socket activation via `OwnedFd`, a background watchdog task, `select!` event loop, and a `cargo-package` Buildroot `.mk` file

**Key topics covered:** `Type=notify` protocol internals, all `sd_notify` message types, `CapabilityBoundingSet`, `PrivateTmp`, `ProtectSystem=strict`, `SystemCallFilter` seccomp, socket unit files, `LISTEN_FDS` fd passing, watchdog timing math, `systemd-analyze security` verification, and a language comparison table.


> **Buildroot Embedded Linux Series — Chapter 20**
> Topics: `Type=notify` · `sd_notify` · Sandboxing · Socket Activation · Watchdog Integration

---

## Table of Contents

1. [Overview](#overview)
2. [systemd Service Lifecycle in Embedded Systems](#systemd-service-lifecycle)
3. [Type=notify and sd_notify Protocol](#type-notify-and-sd_notify)
4. [Sandboxing and Security Hardening](#sandboxing-and-security-hardening)
5. [Socket Activation](#socket-activation)
6. [Watchdog Integration](#watchdog-integration)
7. [C Implementation Examples](#c-implementation-examples)
8. [C++ Implementation Examples](#cpp-implementation-examples)
9. [Rust Implementation Examples](#rust-implementation-examples)
10. [Buildroot Integration](#buildroot-integration)
11. [Summary](#summary)

---

## Overview

In embedded Linux systems built with Buildroot, system daemons require careful lifecycle management. `systemd` — increasingly used even in embedded contexts — provides a rich protocol for daemons to signal readiness, report health, and recover from failures. This chapter covers the full stack: from `.service` unit files to implementing the notification protocol in C, C++, and Rust.

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   EMBEDDED LINUX SYSTEM                     │
  │                                                             │
  │  ┌──────────┐     fork/exec      ┌───────────────────────┐  │
  │  │          │ ──────────────────▶│   Daemon Process      │  │
  │  │ systemd  │                    │                       │  │
  │  │  (PID 1) │ ◀──────────────────| sd_notify("READY=1")  │  │
  │  │          │   UNIX socket      │                       │  │
  │  │          │ ──────────────────▶│   WATCHDOG ping       │  │
  │  │          │ ◀──────────────────│   sd_notify("WATCHDOG │  │
  │  └──────────┘                    │            =1")       │  │
  │        │                         └───────────────────────┘  │
  │        │ manages                                            │
  │        ▼                                                    │
  │  ┌──────────────────────────────────────────────────┐       │
  │  │            cgroup / namespace / caps             │       │
  │  │   PrivateTmp · CapabilityBoundingSet · NoNewPriv │       │
  │  └──────────────────────────────────────────────────┘       │
  └─────────────────────────────────────────────────────────────┘
```

### Why This Matters in Embedded Systems

- **Boot time**: `Type=notify` ensures dependents start only after the daemon is truly ready, preventing race conditions.
- **Security**: Sandboxing limits attack surface on resource-constrained devices that cannot easily be patched.
- **Reliability**: Watchdog integration enables automatic recovery from firmware-level hangs — critical for unattended devices.
- **Socket activation**: Enables on-demand service startup, reducing idle memory footprint.

---

## systemd Service Lifecycle

```
  BOOT
   │
   ▼
  [Loaded]──▶[Activating]──▶[Active(running)]──▶[Deactivating]──▶[Dead]
                   │                │                    │
                   │          sd_notify               SIGTERM
                   │         "READY=1"               received
                   │
              Type=notify
              waits HERE
              until ready
              signal arrives
```

### Basic Unit File Anatomy

```ini
# /etc/systemd/system/mydevd.service

[Unit]
Description=My Embedded Device Daemon
Documentation=https://example.com/mydevd
After=network.target
Requires=mydevd.socket

[Service]
Type=notify
ExecStart=/usr/sbin/mydevd --config /etc/mydevd.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
TimeoutStartSec=30s
WatchdogSec=10s
NotifyAccess=main

[Install]
WantedBy=multi-user.target
```

---

## Type=notify and sd_notify Protocol

`Type=notify` tells systemd to wait for an `sd_notify()` datagram on a UNIX socket before considering the service active. The socket path is passed to the daemon via `$NOTIFY_SOCKET`.

### Notification Protocol Messages

| Message              | Meaning                                              |
|----------------------|------------------------------------------------------|
| `READY=1`            | Daemon initialised, ready to accept work             |
| `RELOADING=1`        | Configuration reload in progress                     |
| `STOPPING=1`         | Clean shutdown initiated                             |
| `STATUS=<text>`      | Human-readable status string                         |
| `ERRNO=<num>`        | Error code (service failed)                          |
| `WATCHDOG=1`         | Watchdog keepalive ping                              |
| `WATCHDOG=trigger`   | Force watchdog failure (self-report crash)           |
| `MAINPID=<pid>`      | Override main PID (after fork)                       |
| `FDSTORE=1`          | Store file descriptors across restarts               |
| `FDNAME=<name>`      | Name for stored file descriptor                      |

### sd_notify Communication Flow

```
  DAEMON                          systemd
    │                                │
    │  [open $NOTIFY_SOCKET]         │
    │                                │
    │  sendmsg("READY=1\n")          │
    │ ────────────────────────────▶  │
    │                                │  [service now "active"]
    │  ... doing work ...            │
    │                                │
    │  sendmsg("WATCHDOG=1\n")       │  every WatchdogSec/2
    │ ────────────────────────────▶  │
    │                                │
    │  sendmsg("STATUS=Processing    │
    │           42 events\n")        │
    │ ────────────────────────────▶  │  [visible in systemctl status]
    │                                │
    │  [SIGTERM received]            │
    │  sendmsg("STOPPING=1\n")       │
    │ ────────────────────────────▶  │
    │  [clean shutdown]              │
```

---

## Sandboxing and Security Hardening

Buildroot embedded targets are often deployed without human oversight. The `systemd` sandbox directives limit what a compromised daemon can do.

### Security Hardening Layers

```
  ┌─────────────────────────────────────────────────────┐
  │              SERVICE SECURITY LAYERS                │
  │                                                     │
  │  ┌─────────────────────────────────────────────┐    │
  │  │  User/Group Isolation                       │    │
  │  │  User=mydevd  Group=mydevd  DynamicUser=yes │    │
  │  └─────────────────────────────────────────────┘    │
  │  ┌─────────────────────────────────────────────┐    │
  │  │  Filesystem Isolation                       │    │
  │  │  PrivateTmp=yes  ProtectSystem=strict       │    │
  │  │  ReadWritePaths=/var/lib/mydevd             │    │
  │  └─────────────────────────────────────────────┘    │
  │  ┌─────────────────────────────────────────────┐    │
  │  │  Capability Restriction                     │    │
  │  │  CapabilityBoundingSet=CAP_NET_BIND_SERVICE │    │
  │  │  AmbientCapabilities=CAP_NET_BIND_SERVICE   │    │
  │  │  NoNewPrivileges=yes                        │    │
  │  └─────────────────────────────────────────────┘    │
  │  ┌─────────────────────────────────────────────┐    │
  │  │  Syscall Filtering                          │    │
  │  │  SystemCallFilter=@system-service           │    │
  │  │  SystemCallArchitectures=native             │    │
  │  └─────────────────────────────────────────────┘    │
  │  ┌─────────────────────────────────────────────┐    │
  │  │  Network/IPC Isolation                      │    │
  │  │  PrivateNetwork=no  RestrictAddressFamilies │    │
  │  │  IPAddressAllow=192.168.0.0/16              │    │
  │  └─────────────────────────────────────────────┘    │
  └─────────────────────────────────────────────────────┘
```

### Full Hardened Service Unit

```ini
# /etc/systemd/system/mydevd.service

[Unit]
Description=My Hardened Embedded Daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/sbin/mydevd
Restart=on-failure
RestartSec=5s
WatchdogSec=30s
NotifyAccess=main

# --- User Isolation ---
User=mydevd
Group=mydevd
DynamicUser=yes

# --- Filesystem Hardening ---
PrivateTmp=yes
PrivateDevices=yes
ProtectSystem=strict
ProtectHome=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
ReadWritePaths=/var/lib/mydevd /run/mydevd

# --- Capability Restriction ---
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_DAC_READ_SEARCH
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes

# --- Syscall Hardening ---
SystemCallFilter=@system-service
SystemCallArchitectures=native
SystemCallErrorNumber=EPERM

# --- Network Restriction ---
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
IPAddressDeny=any
IPAddressAllow=localhost 192.168.0.0/16

# --- Other Restrictions ---
LockPersonality=yes
MemoryDenyWriteExecute=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
RemoveIPC=yes
UMask=0077

[Install]
WantedBy=multi-user.target
```

---

## Socket Activation

Socket activation allows systemd to pre-create sockets and pass them to the daemon. The daemon starts only when a connection arrives, reducing idle overhead — valuable on RAM-constrained embedded targets.

### Socket Unit Architecture

```
  systemd                  Socket Unit (.socket)        Service Unit (.service)
     │                            │                             │
     │  [boot / install]          │                             │
     │──▶ creates socket          │                             │
     │    binds & listens         │                             │
     │                            │                             │
  Client ─── connect() ─────────▶ │                             │
     │                            │  [trigger]                  │
     │                            │──────────────────────────▶  │
     │                            │    pass fd via              │
     │                            │    SD_LISTEN_FDS            │
     │                            │                             │  [daemon starts]
     │                            │                             │  sd_listen_fds()
     │                            │                             │  accepts on fd 3
     │ ◀──────────────────────────────────────────────────────  │
     │         response                                         │
```

### Socket Unit File

```ini
# /etc/systemd/system/mydevd.socket

[Unit]
Description=My Device Daemon Socket
PartOf=mydevd.service

[Socket]
ListenStream=/run/mydevd/mydevd.sock
ListenStream=0.0.0.0:8765
SocketMode=0660
SocketUser=root
SocketGroup=mydevd
Accept=no
Backlog=128
KeepAlive=yes
ReusePort=yes

[Install]
WantedBy=sockets.target
```

### Filesystem Socket Layout

```
  /run/mydevd/
      │
      ├── mydevd.sock          ← UNIX domain socket (created by systemd)
      ├── mydevd.pid           ← PID file (optional with Type=notify)
      └── mydevd.state         ← Runtime state (PrivateTmp not applicable here)
```

---

## Watchdog Integration

The watchdog mechanism ensures systemd can detect and restart a deadlocked or hung daemon. The daemon must call `sd_notify(0, "WATCHDOG=1")` within `WatchdogSec` interval. Missing the deadline triggers a SIGABRT + restart.

### Watchdog Timing Diagram

```
  WatchdogSec = 10s

  Time:  0s      5s      10s     15s     20s
         │       │       │       │       │
  ping:  ●───────●───────●       X       │
         │       │       │       │       │
                                 ↑
                         WATCHDOG TIMEOUT
                         systemd sends SIGABRT
                         service restarts
```

### Watchdog with Hardware Watchdog

```
  ┌────────────┐    ping     ┌────────────┐    ping     ┌─────────────────┐
  │   Daemon   │ ──────────▶ │  systemd   │ ──────────▶ │  /dev/watchdog  │
  │            │  WATCHDOG=1 │            │  ioctl()    │  (HW Watchdog)  │
  │            │  via socket │            │  KEEPALIVE  │                 │
  └────────────┘             └────────────┘             └─────────────────┘

  RuntimeWatchdogSec=30s   → systemd pings /dev/watchdog
  WatchdogSec=10s          → daemon must ping systemd
```

---

## C Implementation Examples

### sd_notify Helper (Pure C, no libsystemd dependency)

This implementation avoids linking `libsystemd` — useful when Buildroot config
excludes it to minimise footprint:

```c
/* sd_notify_simple.h — minimal sd_notify without libsystemd */
#ifndef SD_NOTIFY_SIMPLE_H
#define SD_NOTIFY_SIMPLE_H

#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>
#include <stdio.h>

/**
 * sd_notify_simple - send a notification message to systemd
 * @unset_env: if non-zero, unset NOTIFY_SOCKET after first send
 * @state:     NUL-terminated state string e.g. "READY=1"
 *
 * Returns 0 on success, -errno on failure, 1 if NOTIFY_SOCKET unset.
 */
static inline int sd_notify_simple(int unset_env, const char *state)
{
    const char *socket_path;
    struct sockaddr_un addr;
    int fd, r;
    ssize_t n;

    socket_path = getenv("NOTIFY_SOCKET");
    if (!socket_path)
        return 1; /* not running under systemd, silently ignore */

    /* Abstract or path socket */
    if (socket_path[0] != '@' && socket_path[0] != '/')
        return -EINVAL;

    fd = socket(AF_UNIX, SOCK_DGRAM | SOCK_CLOEXEC, 0);
    if (fd < 0)
        return -errno;

    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;

    if (socket_path[0] == '@') {
        /* Abstract socket namespace */
        addr.sun_path[0] = '\0';
        strncpy(addr.sun_path + 1, socket_path + 1,
                sizeof(addr.sun_path) - 2);
    } else {
        strncpy(addr.sun_path, socket_path,
                sizeof(addr.sun_path) - 1);
    }

    n = sendto(fd, state, strlen(state), MSG_NOSIGNAL,
               (struct sockaddr *)&addr,
               offsetof(struct sockaddr_un, sun_path) + strlen(addr.sun_path) + 1);
    r = (n < 0) ? -errno : 0;

    close(fd);

    if (unset_env)
        unsetenv("NOTIFY_SOCKET");

    return r;
}

#define SD_NOTIFY_READY()    sd_notify_simple(0, "READY=1")
#define SD_NOTIFY_STOPPING() sd_notify_simple(0, "STOPPING=1")
#define SD_NOTIFY_RELOADING() sd_notify_simple(0, "RELOADING=1")
#define SD_NOTIFY_WATCHDOG() sd_notify_simple(0, "WATCHDOG=1")
#define SD_NOTIFY_STATUS(s)  sd_notify_simple(0, "STATUS=" s)

#endif /* SD_NOTIFY_SIMPLE_H */
```

### Full C Daemon with Watchdog and Socket Activation

```c
/* mydevd.c — embedded daemon with systemd integration */

#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <errno.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <sys/timerfd.h>
#include <fcntl.h>
#include <time.h>
#include <stdint.h>

#include "sd_notify_simple.h"

/* SD_LISTEN_FDS_START: first fd passed by systemd socket activation */
#define SD_LISTEN_FDS_START 3

static volatile sig_atomic_t g_running = 1;
static volatile sig_atomic_t g_reload  = 0;

/* ------------------------------------------------------------------ */
/* Signal handling                                                     */
/* ------------------------------------------------------------------ */
static void sig_handler(int signo)
{
    if (signo == SIGTERM || signo == SIGINT)
        g_running = 0;
    else if (signo == SIGHUP)
        g_reload = 1;
}

static void setup_signals(void)
{
    struct sigaction sa = { .sa_handler = sig_handler };
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;

    sigaction(SIGTERM, &sa, NULL);
    sigaction(SIGINT,  &sa, NULL);
    sigaction(SIGHUP,  &sa, NULL);
    signal(SIGPIPE, SIG_IGN);
}

/* ------------------------------------------------------------------ */
/* Watchdog helpers                                                    */
/* ------------------------------------------------------------------ */
typedef struct {
    int      timerfd;
    uint64_t interval_usec;
} watchdog_t;

/**
 * watchdog_init - parse WATCHDOG_USEC and set up a timerfd
 *
 * Returns 0 if watchdog is active, 1 if not requested, -errno on error.
 */
static int watchdog_init(watchdog_t *wd)
{
    const char *env = getenv("WATCHDOG_USEC");
    uint64_t usec;
    struct itimerspec its;

    if (!env) {
        wd->timerfd = -1;
        return 1; /* watchdog not requested */
    }

    usec = (uint64_t)strtoull(env, NULL, 10);
    if (usec == 0)
        return -EINVAL;

    /* Ping at half the interval for safety margin */
    wd->interval_usec = usec / 2;

    wd->timerfd = timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC | TFD_NONBLOCK);
    if (wd->timerfd < 0)
        return -errno;

    its.it_value.tv_sec  = wd->interval_usec / 1000000;
    its.it_value.tv_nsec = (wd->interval_usec % 1000000) * 1000;
    its.it_interval      = its.it_value;

    if (timerfd_settime(wd->timerfd, 0, &its, NULL) < 0) {
        int err = errno;
        close(wd->timerfd);
        wd->timerfd = -1;
        return -err;
    }

    fprintf(stderr, "Watchdog active: ping every %lu ms\n",
            (unsigned long)(wd->interval_usec / 1000));
    return 0;
}

static void watchdog_ping(watchdog_t *wd)
{
    if (wd->timerfd < 0)
        return;

    /* Drain the timerfd */
    uint64_t exp;
    ssize_t n = read(wd->timerfd, &exp, sizeof(exp));
    if (n == sizeof(exp) && exp > 0)
        SD_NOTIFY_WATCHDOG();
}

/* ------------------------------------------------------------------ */
/* Socket activation                                                   */
/* ------------------------------------------------------------------ */
/**
 * get_socket_fds - retrieve fds passed by systemd socket activation
 *
 * Returns number of fds available (0 = not socket-activated).
 */
static int get_socket_fds(void)
{
    const char *env = getenv("LISTEN_FDS");
    if (!env)
        return 0;

    int n = atoi(env);

    /* Verify PID matches */
    const char *pid_env = getenv("LISTEN_PID");
    if (pid_env && (pid_t)atol(pid_env) != getpid())
        return 0;

    /* Set CLOEXEC on inherited fds */
    for (int i = 0; i < n; i++) {
        int fd = SD_LISTEN_FDS_START + i;
        int flags = fcntl(fd, F_GETFD);
        if (flags >= 0)
            fcntl(fd, F_SETFD, flags | FD_CLOEXEC);
    }

    return n;
}

/* ------------------------------------------------------------------ */
/* Main event loop                                                     */
/* ------------------------------------------------------------------ */
static int run_event_loop(int listen_fd, watchdog_t *wd)
{
    int epfd;
    struct epoll_event ev, events[16];

    epfd = epoll_create1(EPOLL_CLOEXEC);
    if (epfd < 0) return -errno;

    /* Add listening socket */
    ev.events  = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    /* Add watchdog timerfd */
    if (wd->timerfd >= 0) {
        ev.events  = EPOLLIN;
        ev.data.fd = wd->timerfd;
        epoll_ctl(epfd, EPOLL_CTL_ADD, wd->timerfd, &ev);
    }

    /* Signal readiness — the moment dependents can proceed */
    sd_notify_simple(0, "READY=1\nSTATUS=Waiting for connections");
    fprintf(stderr, "Daemon ready\n");

    while (g_running) {
        int n = epoll_wait(epfd, events, 16, 1000);
        if (n < 0) {
            if (errno == EINTR) continue;
            break;
        }

        if (g_reload) {
            g_reload = 0;
            sd_notify_simple(0, "RELOADING=1");
            /* reload config here */
            sd_notify_simple(0, "READY=1\nSTATUS=Reloaded configuration");
        }

        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;

            if (fd == wd->timerfd) {
                watchdog_ping(wd);
            } else if (fd == listen_fd) {
                int client = accept4(listen_fd, NULL, NULL,
                                     SOCK_CLOEXEC | SOCK_NONBLOCK);
                if (client >= 0) {
                    /* handle client — simplified */
                    const char msg[] = "PONG\n";
                    write(client, msg, sizeof(msg) - 1);
                    close(client);
                }
            }
        }
    }

    sd_notify_simple(0, "STOPPING=1");
    close(epfd);
    return 0;
}

/* ------------------------------------------------------------------ */
/* Entry point                                                         */
/* ------------------------------------------------------------------ */
int main(void)
{
    int listen_fd;
    watchdog_t wd = { .timerfd = -1 };

    setup_signals();
    watchdog_init(&wd);

    int n_fds = get_socket_fds();
    if (n_fds > 0) {
        /* Socket-activated: use fd passed by systemd */
        listen_fd = SD_LISTEN_FDS_START;
        fprintf(stderr, "Socket-activated on fd %d\n", listen_fd);
    } else {
        fprintf(stderr, "ERROR: must be socket-activated\n");
        return EXIT_FAILURE;
    }

    int r = run_event_loop(listen_fd, &wd);

    if (wd.timerfd >= 0)
        close(wd.timerfd);

    fprintf(stderr, "Daemon exiting cleanly\n");
    return r == 0 ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

**Buildroot Makefile snippet for C daemon:**

```makefile
# package/mydevd/mydevd.mk

MYDEVD_VERSION = 1.0.0
MYDEVD_SITE    = $(TOPDIR)/package/mydevd/src
MYDEVD_SITE_METHOD = local

define MYDEVD_BUILD_CMDS
    $(TARGET_CC) $(TARGET_CFLAGS) -o $(@D)/mydevd \
        $(@D)/mydevd.c \
        $(TARGET_LDFLAGS)
endef

define MYDEVD_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/mydevd \
        $(TARGET_DIR)/usr/sbin/mydevd
    $(INSTALL) -D -m 0644 $(PKGDIR)/mydevd.service \
        $(TARGET_DIR)/etc/systemd/system/mydevd.service
    $(INSTALL) -D -m 0644 $(PKGDIR)/mydevd.socket \
        $(TARGET_DIR)/etc/systemd/system/mydevd.socket
endef

$(eval $(generic-package))
```

---

## C++ Implementation Examples

### C++ Daemon Class with sd_notify, Watchdog, and Socket Activation

```cpp
// daemon.hpp — C++ RAII daemon with systemd integration

#pragma once
#include <string>
#include <functional>
#include <atomic>
#include <stdexcept>
#include <cstring>
#include <cerrno>

#include <unistd.h>
#include <signal.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <sys/timerfd.h>
#include <fcntl.h>

#include "sd_notify_simple.h"

namespace systemd {

/* ------------------------------------------------------------------ */
/* Scoped UNIX file descriptor                                         */
/* ------------------------------------------------------------------ */
class Fd {
public:
    explicit Fd(int fd = -1) noexcept : fd_(fd) {}
    ~Fd() { if (fd_ >= 0) ::close(fd_); }

    Fd(Fd &&o) noexcept : fd_(o.release()) {}
    Fd &operator=(Fd &&o) noexcept {
        if (this != &o) { reset(o.release()); }
        return *this;
    }

    Fd(const Fd &) = delete;
    Fd &operator=(const Fd &) = delete;

    int get() const noexcept { return fd_; }
    bool valid() const noexcept { return fd_ >= 0; }
    int release() noexcept { int r = fd_; fd_ = -1; return r; }
    void reset(int fd = -1) noexcept {
        if (fd_ >= 0) ::close(fd_);
        fd_ = fd;
    }

private:
    int fd_;
};

/* ------------------------------------------------------------------ */
/* Watchdog timer                                                      */
/* ------------------------------------------------------------------ */
class Watchdog {
public:
    Watchdog() {
        const char *env = getenv("WATCHDOG_USEC");
        if (!env) return;

        uint64_t usec = std::stoull(env);
        if (usec == 0) return;

        interval_usec_ = usec / 2;  // ping at half interval

        int fd = timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC | TFD_NONBLOCK);
        if (fd < 0)
            throw std::system_error(errno, std::generic_category(),
                                    "timerfd_create");

        fd_.reset(fd);
        arm();
    }

    int fd() const noexcept { return fd_.get(); }
    bool active() const noexcept { return fd_.valid(); }

    void ping() {
        if (!fd_.valid()) return;
        uint64_t exp = 0;
        if (read(fd_.get(), &exp, sizeof(exp)) == sizeof(exp) && exp > 0)
            sd_notify_simple(0, "WATCHDOG=1");
    }

private:
    void arm() {
        struct itimerspec its{};
        its.it_value.tv_sec  = interval_usec_ / 1'000'000;
        its.it_value.tv_nsec = (interval_usec_ % 1'000'000) * 1000;
        its.it_interval      = its.it_value;

        if (timerfd_settime(fd_.get(), 0, &its, nullptr) < 0)
            throw std::system_error(errno, std::generic_category(),
                                    "timerfd_settime");
    }

    Fd       fd_;
    uint64_t interval_usec_ = 0;
};

/* ------------------------------------------------------------------ */
/* Epoll event loop                                                    */
/* ------------------------------------------------------------------ */
class EventLoop {
public:
    using Handler = std::function<void(int fd, uint32_t events)>;

    EventLoop() {
        int fd = epoll_create1(EPOLL_CLOEXEC);
        if (fd < 0)
            throw std::system_error(errno, std::generic_category(),
                                    "epoll_create1");
        epfd_.reset(fd);
    }

    void add(int fd, uint32_t events, Handler h) {
        handlers_[fd] = std::move(h);
        struct epoll_event ev{};
        ev.events   = events;
        ev.data.fd  = fd;
        if (epoll_ctl(epfd_.get(), EPOLL_CTL_ADD, fd, &ev) < 0)
            throw std::system_error(errno, std::generic_category(),
                                    "epoll_ctl ADD");
    }

    void remove(int fd) {
        epoll_ctl(epfd_.get(), EPOLL_CTL_DEL, fd, nullptr);
        handlers_.erase(fd);
    }

    void run(std::atomic<bool> &running) {
        constexpr int MAX_EVENTS = 32;
        struct epoll_event events[MAX_EVENTS];

        while (running.load(std::memory_order_acquire)) {
            int n = epoll_wait(epfd_.get(), events, MAX_EVENTS, 500);
            if (n < 0) {
                if (errno == EINTR) continue;
                throw std::system_error(errno, std::generic_category(),
                                        "epoll_wait");
            }
            for (int i = 0; i < n; i++) {
                int fd = events[i].data.fd;
                if (auto it = handlers_.find(fd); it != handlers_.end())
                    it->second(fd, events[i].events);
            }
        }
    }

private:
    Fd epfd_;
    std::unordered_map<int, Handler> handlers_;
};

/* ------------------------------------------------------------------ */
/* Main daemon class                                                   */
/* ------------------------------------------------------------------ */
class Daemon {
public:
    Daemon() {
        // Validate socket activation
        const char *listen_fds = getenv("LISTEN_FDS");
        if (!listen_fds || std::atoi(listen_fds) < 1)
            throw std::runtime_error("Must be socket-activated (LISTEN_FDS not set)");

        listen_fd_ = SD_LISTEN_FDS_START;  // first activated socket
    }

    void run() {
        Watchdog wd;
        EventLoop loop;

        // Register watchdog timer
        if (wd.active()) {
            loop.add(wd.fd(), EPOLLIN, [&wd](int, uint32_t) {
                wd.ping();
            });
        }

        // Register listening socket
        loop.add(listen_fd_, EPOLLIN, [this](int fd, uint32_t) {
            handle_accept(fd);
        });

        // Signal ready — dependents will now start
        sd_notify_simple(0, "READY=1\nSTATUS=Event loop running");

        loop.run(running_);

        sd_notify_simple(0, "STOPPING=1");
    }

    void stop() noexcept {
        running_.store(false, std::memory_order_release);
    }

private:
    void handle_accept(int listen_fd) {
        int client = accept4(listen_fd, nullptr, nullptr,
                             SOCK_CLOEXEC | SOCK_NONBLOCK);
        if (client < 0) return;

        Fd cfd(client);
        constexpr char reply[] = "PONG\n";
        write(cfd.get(), reply, sizeof(reply) - 1);
        // cfd destructor closes socket
    }

    std::atomic<bool> running_{true};
    int               listen_fd_{-1};
};

} // namespace systemd
```

```cpp
// main.cpp — entry point with signal handling

#include "daemon.hpp"
#include <iostream>
#include <csignal>
#include <memory>

static systemd::Daemon *g_daemon = nullptr;

static void on_signal(int sig) noexcept {
    if (g_daemon) g_daemon->stop();
}

int main() {
    try {
        auto daemon = std::make_unique<systemd::Daemon>();
        g_daemon = daemon.get();

        struct sigaction sa{};
        sa.sa_handler = on_signal;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = SA_RESTART;
        sigaction(SIGTERM, &sa, nullptr);
        sigaction(SIGINT,  &sa, nullptr);
        signal(SIGPIPE, SIG_IGN);

        daemon->run();

        g_daemon = nullptr;
        std::cerr << "Daemon exited cleanly\n";
        return EXIT_SUCCESS;

    } catch (const std::exception &ex) {
        std::cerr << "FATAL: " << ex.what() << '\n';
        sd_notify_simple(0, "STATUS=Fatal error during startup");
        return EXIT_FAILURE;
    }
}
```

**CMakeLists.txt for Buildroot external tree:**

```cmake
cmake_minimum_required(VERSION 3.16)
project(mydevd CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(mydevd
    src/main.cpp
)

target_include_directories(mydevd PRIVATE include)

# Optionally link real libsystemd if available
find_package(PkgConfig QUIET)
if(PKG_CONFIG_FOUND)
    pkg_check_modules(SYSTEMD QUIET libsystemd)
    if(SYSTEMD_FOUND)
        target_link_libraries(mydevd ${SYSTEMD_LIBRARIES})
        target_compile_definitions(mydevd PRIVATE HAVE_LIBSYSTEMD=1)
    endif()
endif()

target_compile_options(mydevd PRIVATE
    -Wall -Wextra -Wpedantic
    -fstack-protector-strong
    -D_FORTIFY_SOURCE=2
)

install(TARGETS mydevd DESTINATION sbin)
install(FILES systemd/mydevd.service systemd/mydevd.socket
        DESTINATION etc/systemd/system)
```

---

## Rust Implementation Examples

### Cargo.toml

```toml
[package]
name    = "mydevd"
version = "1.0.0"
edition = "2021"

[[bin]]
name = "mydevd"
path = "src/main.rs"

[dependencies]
# systemd notification — supports sd_notify, watchdog, socket activation
sd-notify    = "0.4"
# async runtime for embedded-friendly async I/O
tokio        = { version = "1", features = ["rt", "net", "signal", "time", "io-util"] }
# structured logging
tracing      = "0.1"
tracing-subscriber = { version = "0.3", features = ["fmt"] }
# error handling
anyhow       = "1"

[profile.release]
opt-level    = "z"     # size-optimised for embedded
lto          = true
codegen-units = 1
strip        = true
panic        = "abort"
```

### Rust Daemon: Full Implementation

```rust
// src/main.rs — Async Rust daemon with full systemd integration

use std::time::Duration;
use anyhow::{Context, Result};
use sd_notify::NotifyState;
use tokio::{
    net::{TcpListener, UnixListener},
    signal::unix::{signal, SignalKind},
    time,
};
use tracing::{error, info, warn};

/* ------------------------------------------------------------------ */
/* Socket activation                                                   */
/* ------------------------------------------------------------------ */

/// Retrieve file descriptors passed by systemd socket activation.
/// Returns `None` if not socket-activated.
fn socket_activation_fds() -> Option<Vec<std::os::fd::OwnedFd>> {
    use std::os::fd::FromRawFd;

    let listen_fds: usize = std::env::var("LISTEN_FDS")
        .ok()?
        .parse()
        .ok()?;

    let listen_pid: u32 = std::env::var("LISTEN_PID")
        .ok()?
        .parse()
        .ok()?;

    if listen_pid != std::process::id() {
        return None;
    }

    let fds: Vec<_> = (0..listen_fds)
        .map(|i| unsafe {
            std::os::fd::OwnedFd::from_raw_fd((3 + i) as i32)
        })
        .collect();

    Some(fds)
}

/* ------------------------------------------------------------------ */
/* Watchdog task                                                       */
/* ------------------------------------------------------------------ */

/// Spawn a Tokio task that pings the systemd watchdog at the
/// recommended interval (half of WATCHDOG_USEC).
fn spawn_watchdog() -> Option<tokio::task::JoinHandle<()>> {
    let usec: u64 = std::env::var("WATCHDOG_USEC")
        .ok()
        .and_then(|s| s.parse().ok())?;

    // Ping at half the watchdog interval for safety
    let interval = Duration::from_micros(usec / 2);
    info!("Watchdog active: pinging every {:?}", interval);

    Some(tokio::spawn(async move {
        let mut ticker = time::interval(interval);
        ticker.set_missed_tick_behavior(time::MissedTickBehavior::Delay);
        loop {
            ticker.tick().await;
            if let Err(e) = sd_notify::notify(false, &[NotifyState::Watchdog]) {
                error!("Watchdog notify failed: {}", e);
            }
        }
    }))
}

/* ------------------------------------------------------------------ */
/* Connection handler                                                  */
/* ------------------------------------------------------------------ */

/// Handle a single accepted connection.
async fn handle_connection<S>(mut stream: S)
where
    S: tokio::io::AsyncReadExt + tokio::io::AsyncWriteExt + Unpin,
{
    use tokio::io::AsyncBufReadExt;
    let mut reader = tokio::io::BufReader::new(&mut stream);
    let mut line   = String::new();

    match reader.read_line(&mut line).await {
        Ok(0) => { /* EOF */ }
        Ok(_) => {
            let line = line.trim();
            info!("Received: {:?}", line);

            let response = match line {
                "PING" => "PONG\n".to_string(),
                cmd    => format!("UNKNOWN: {}\n", cmd),
            };

            if let Err(e) = stream.write_all(response.as_bytes()).await {
                warn!("Write error: {}", e);
            }
        }
        Err(e) => warn!("Read error: {}", e),
    }
}

/* ------------------------------------------------------------------ */
/* Service status reporting                                            */
/* ------------------------------------------------------------------ */

struct StatusReporter {
    connections: std::sync::atomic::AtomicU64,
}

impl StatusReporter {
    fn new() -> Self {
        Self {
            connections: std::sync::atomic::AtomicU64::new(0),
        }
    }

    fn increment(&self) {
        self.connections
            .fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    }

    fn report(&self) -> Result<()> {
        let count = self.connections
            .load(std::sync::atomic::Ordering::Relaxed);

        sd_notify::notify(
            false,
            &[NotifyState::Status(
                format!("Served {} connections", count)
            )],
        ).context("sd_notify status failed")
    }
}

/* ------------------------------------------------------------------ */
/* Main                                                                */
/* ------------------------------------------------------------------ */

#[tokio::main(flavor = "current_thread")]   // single-thread for embedded
async fn main() -> Result<()> {
    tracing_subscriber::fmt()
        .with_writer(std::io::stderr)
        .with_target(false)
        .init();

    info!("mydevd starting");

    // --- Socket Activation ---
    let fds = socket_activation_fds()
        .context("Must be socket-activated (LISTEN_FDS not set)")?;

    if fds.is_empty() {
        anyhow::bail!("No socket fds received from systemd");
    }

    // Convert the first fd to a Tokio TcpListener (or UnixListener).
    // Here we use TcpListener; for UNIX sockets use UnixListener::from_std.
    use std::os::fd::IntoRawFd;
    let raw_fd = fds.into_iter().next().unwrap().into_raw_fd();
    let std_listener = unsafe {
        use std::net::TcpListener as StdTcp;
        StdTcp::from_raw_fd(raw_fd)
    };
    std_listener.set_nonblocking(true)?;
    let listener = TcpListener::from_std(std_listener)?;
    info!("Listening on {}", listener.local_addr()?);

    // --- Watchdog ---
    let _watchdog = spawn_watchdog();

    // --- Signal handling ---
    let mut sigterm = signal(SignalKind::terminate())?;
    let mut sighup  = signal(SignalKind::hangup())?;

    // --- Status reporter (shared via Arc) ---
    let reporter = std::sync::Arc::new(StatusReporter::new());

    // --- Notify systemd: daemon is ready ---
    sd_notify::notify(
        false,
        &[
            NotifyState::Ready,
            NotifyState::Status("Accepting connections".into()),
        ],
    ).context("sd_notify READY failed")?;

    info!("Daemon ready");

    // --- Event loop ---
    loop {
        tokio::select! {
            // Accept new connections
            result = listener.accept() => {
                match result {
                    Ok((stream, peer)) => {
                        info!("Connection from {}", peer);
                        let rep = reporter.clone();
                        tokio::spawn(async move {
                            handle_connection(stream).await;
                            rep.increment();
                        });
                        reporter.report()?;
                    }
                    Err(e) => error!("Accept error: {}", e),
                }
            }

            // Reload signal (SIGHUP)
            _ = sighup.recv() => {
                info!("Reloading configuration");
                sd_notify::notify(false, &[NotifyState::Reloading])?;
                // reload logic here …
                sd_notify::notify(
                    false,
                    &[
                        NotifyState::Ready,
                        NotifyState::Status("Reloaded".into()),
                    ],
                )?;
            }

            // Shutdown signal (SIGTERM)
            _ = sigterm.recv() => {
                info!("Received SIGTERM, shutting down");
                break;
            }
        }
    }

    // Announce graceful shutdown to systemd
    sd_notify::notify(false, &[NotifyState::Stopping])?;
    info!("Daemon exiting cleanly");
    Ok(())
}
```

### Rust: Cross-compilation for ARM (Buildroot)

```bash
# In Buildroot external tree: package/mydevd-rust/mydevd-rust.mk

MYDEVD_RUST_VERSION = 1.0.0
MYDEVD_RUST_SITE    = $(TOPDIR)/../mydevd-rust
MYDEVD_RUST_SITE_METHOD = local

MYDEVD_RUST_CARGO_ENV = \
    CARGO_HOME=$(HOST_DIR)/share/cargo

MYDEVD_RUST_CARGO_OPTS = \
    --release \
    --target $(RUSTC_TARGET_NAME)

define MYDEVD_RUST_BUILD_CMDS
    $(TARGET_MAKE_ENV) $(MYDEVD_RUST_CARGO_ENV) \
        cargo build $(MYDEVD_RUST_CARGO_OPTS) \
        --manifest-path $(@D)/Cargo.toml
endef

define MYDEVD_RUST_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 \
        $(@D)/target/$(RUSTC_TARGET_NAME)/release/mydevd \
        $(TARGET_DIR)/usr/sbin/mydevd
    $(INSTALL) -D -m 0644 $(PKGDIR)/mydevd.service \
        $(TARGET_DIR)/etc/systemd/system/mydevd.service
    $(INSTALL) -D -m 0644 $(PKGDIR)/mydevd.socket \
        $(TARGET_DIR)/etc/systemd/system/mydevd.socket
endef

$(eval $(cargo-package))
```

---

## Buildroot Integration

### Directory Layout

```
  buildroot-external/
  ├── Config.in
  ├── external.mk
  └── package/
      └── mydevd/
          ├── Config.in             ← menu entry
          ├── mydevd.mk             ← package rules
          ├── mydevd.hash           ← source integrity
          ├── mydevd.service        ← systemd unit
          ├── mydevd.socket         ← socket unit
          └── S50mydevd             ← sysvinit fallback script
```

### Buildroot Config.in

```kconfig
config BR2_PACKAGE_MYDEVD
    bool "mydevd"
    depends on BR2_INIT_SYSTEMD
    select BR2_PACKAGE_SYSTEMD
    help
      My embedded device daemon with systemd socket activation,
      watchdog integration, and sandbox hardening.

      Requires systemd init system.
```

### Buildroot systemd Post-install Hook

```makefile
# Ensure service is enabled in the image
define MYDEVD_INSTALL_INIT_SYSTEMD
    mkdir -p $(TARGET_DIR)/etc/systemd/system/sockets.target.wants
    ln -sf ../mydevd.socket \
        $(TARGET_DIR)/etc/systemd/system/sockets.target.wants/mydevd.socket
endef
```

### Verifying the Service on Target

```
  TARGET SHELL SESSION
  ─────────────────────────────────────────────────────────────

  # Check socket-activated service status
  $ systemctl status mydevd.socket mydevd.service

  ● mydevd.socket - My Device Daemon Socket
       Loaded: loaded (/etc/systemd/system/mydevd.socket; enabled)
       Active: active (listening) since Mon 2024-01-15 08:00:01 UTC
     Listen: /run/mydevd/mydevd.sock (Stream)
              0.0.0.0:8765 (Stream)

  ● mydevd.service - My Hardened Embedded Daemon
       Loaded: loaded (/etc/systemd/system/mydevd.service; indirect)
       Active: active (running) since Mon 2024-01-15 08:00:05 UTC
     Main PID: 312 (mydevd)
       Status: "Served 3 connections"
      CGroup: /system.slice/mydevd.service
              └─312 /usr/sbin/mydevd

  # Inspect security score
  $ systemd-analyze security mydevd.service

  NAME                        DESCRIPTION          EXPOSURE
  ✓ PrivateTmp=yes
  ✓ NoNewPrivileges=yes
  ✓ CapabilityBoundingSet=~...
  ✓ SystemCallFilter=@system-service
  → Overall exposure level: 2.1 SAFE

  # Live watchdog check
  $ journalctl -fu mydevd.service
  Jan 15 08:00:05 target mydevd[312]: Daemon ready
  Jan 15 08:00:05 target mydevd[312]: Watchdog active: pinging every 5000ms
  Jan 15 08:00:10 target mydevd[312]: Served 3 connections
```

---

## Summary

This chapter covered the full lifecycle of writing robust, secure, production-grade daemons for embedded Linux systems managed by systemd, in the context of Buildroot-based targets.

### Key Concepts Recap

```
  ┌────────────────────────────────────────────────────────────────┐
  │              CHAPTER 20 — CONCEPT MAP                          │
  │                                                                │
  │  Type=notify ──────────▶ sd_notify("READY=1")                  │
  │       │                        │                               │
  │       │                  sent via UNIX socket                  │
  │       │                  ($NOTIFY_SOCKET)                      │
  │       ▼                                                        │
  │  systemd waits          NotifyAccess=main                      │
  │  until READY            restricts who can notify               │
  │                                                                │
  │  ─────────────────────────────────────────────────────────     │
  │                                                                │
  │  WatchdogSec=N ────────▶ daemon must sd_notify("WATCHDOG=1")   │
  │                          every N/2 seconds or get killed       │
  │                          and auto-restarted                    │
  │                                                                │
  │  ─────────────────────────────────────────────────────────     │
  │                                                                │
  │  mydevd.socket ────────▶ systemd holds socket at boot          │
  │       │                  daemon starts ON DEMAND               │
  │       └──triggers──────▶ mydevd.service receives fd via        │
  │                          LISTEN_FDS environment variable       │
  │                                                                │
  │  ─────────────────────────────────────────────────────────     │
  │                                                                │
  │  Sandboxing layers:                                            │
  │    PrivateTmp            → isolated /tmp namespace             │
  │    CapabilityBoundingSet → drop all unneeded Linux caps        │
  │    NoNewPrivileges       → block setuid/capability escalation  │
  │    SystemCallFilter      → seccomp allowlist                   │
  │    ProtectSystem=strict  → read-only rootfs view               │
  └────────────────────────────────────────────────────────────────┘
```

### Language Comparison

| Feature                  | C (manual)         | C++ (RAII)          | Rust (async)             |
|--------------------------|--------------------|---------------------|--------------------------|
| sd_notify                | manual socket send | header-only wrapper | `sd-notify` crate        |
| Watchdog                 | timerfd + epoll    | timerfd + EventLoop | tokio interval task      |
| Socket activation        | LISTEN_FDS env     | LISTEN_FDS env      | LISTEN_FDS + OwnedFd     |
| Resource safety          | manual close()     | RAII Fd class       | OwnedFd / Drop trait     |
| Async I/O                | epoll manual       | epoll EventLoop     | tokio select! macro      |
| Binary size (stripped)   | ~20 KB             | ~60–120 KB          | ~300 KB (musl static)    |
| Buildroot integration    | generic-package    | cmake-package       | cargo-package            |

### Best Practices for Embedded Targets

- Always use `Type=notify` — never `Type=simple` — for daemons with non-trivial initialisation.
- Set `WatchdogSec` conservatively: 3–5× your worst-case initialisation time.
- Ping the watchdog at `WatchdogSec / 2`, not at the deadline.
- Enable `PrivateTmp=yes` and `NoNewPrivileges=yes` as a baseline; add further hardening incrementally and verify with `systemd-analyze security`.
- Use socket activation for optional or infrequently-used services to reduce boot time and idle memory footprint.
- Avoid linking `libsystemd` if Buildroot's footprint budget is tight — the raw socket `sd_notify` implementation adds zero overhead.
- In Rust, prefer `#[tokio::main(flavor = "current_thread")]` on single-core targets to avoid multi-threaded runtime overhead.
- Always send `STOPPING=1` before exiting to allow dependents to handle orderly shutdown.

---

*End of Chapter 20 — Next: Chapter 21: D-Bus Integration in Embedded Daemons*