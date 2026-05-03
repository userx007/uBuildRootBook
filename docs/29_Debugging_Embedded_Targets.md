# 29. Debugging Embedded Targets (GDB, strace, perf)

> **Buildroot Topic Series — Chapter 29**
> Covers: `BR2_ENABLE_DEBUG`, remote GDB with `gdbserver`, `BR2_PACKAGE_STRACE`,
> `perf` cross-build, and JTAG integration via OpenOCD.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Debug Build Configuration — BR2_ENABLE_DEBUG](#1-debug-build-configuration--br2_enable_debug)
3. [Remote GDB Debugging with gdbserver](#2-remote-gdb-debugging-with-gdbserver)
4. [System Call Tracing with strace](#3-system-call-tracing-with-strace)
5. [Performance Profiling with perf](#4-performance-profiling-with-perf)
6. [JTAG Integration via OpenOCD](#5-jtag-integration-via-openocd)
7. [Code Examples — C](#6-code-examples--c)
8. [Code Examples — C++](#7-code-examples--c)
9. [Code Examples — Rust](#8-code-examples--rust)
10. [Debugging Workflow Diagrams](#9-debugging-workflow-diagrams)
11. [Summary](#10-summary)

---

## Introduction

Debugging embedded Linux systems presents unique challenges compared to desktop
development. The target hardware is often resource-constrained, may lack a
display, and runs a stripped-down userspace. Buildroot addresses these
challenges by providing a tightly integrated set of debugging tools that can be
cross-compiled and deployed to the target with minimal overhead.

This chapter covers the four primary debugging pillars available in Buildroot:

- **Symbolic debug builds** — preserving DWARF debug info via `BR2_ENABLE_DEBUG`
- **Remote GDB** — source-level interactive debugging over a network or serial link
- **strace** — lightweight system call tracing without rebuilding the target
- **perf** — kernel-assisted performance profiling and hardware counter access
- **OpenOCD + JTAG** — bare-metal and OS-aware hardware-level debugging

Together these tools cover the full spectrum from high-level application logic
down to CPU pipeline events and hardware register states.

---

## 1. Debug Build Configuration — BR2_ENABLE_DEBUG

### What It Does

By default Buildroot compiles all packages with `-Os` (optimize for size) and
strips binaries before installing them into the target filesystem. Enabling
`BR2_ENABLE_DEBUG` changes this behavior across the entire build system:

- Compiler flags switch from `-Os` to `-g` (DWARF debug information)
- The optimization level drops to `-Og` (debug-friendly optimizations)
- Stripping of target binaries is disabled
- Debug packages (`*-dbg`) become buildable

### Menuconfig Path

```
make menuconfig
  → Build options
      → [*] build packages with debugging symbols   (BR2_ENABLE_DEBUG)
      → gcc debug level (3)                         (BR2_DEBUG_3)
      → [*] strip target binaries                   (uncheck this!)
      → optimization level (optimize for debugging) (BR2_OPTIMIZE_G)
```

### Relevant .config Entries

```ini
# .config excerpt — debug build
BR2_ENABLE_DEBUG=y
BR2_OPTIMIZE_G=y
BR2_DEBUG_3=y
# BR2_STRIP_strip is not set
```

### Effect on the Toolchain Wrapper

Buildroot's toolchain wrapper injects these flags into every compiler
invocation. Packages that hard-code their own `-O2` flags may override this;
inspect `package/<pkg>/<pkg>.mk` and override `<PKG>_CONF_OPTS` or patch the
build system as needed.

```makefile
# package/myapp/myapp.mk — forcing debug flags
MYAPP_CONF_OPTS += --enable-debug
MYAPP_CFLAGS    = $(TARGET_CFLAGS) -g3 -Og
```

### Separate Debug Symbols (Split Debug)

For production images that must be small but still debuggable from a host, use
split debug symbols:

```makefile
# Post-install hook in myapp.mk
define MYAPP_SPLIT_DEBUG
    $(TARGET_OBJCOPY) --only-keep-debug \
        $(TARGET_DIR)/usr/bin/myapp \
        $(BUILD_DIR)/myapp-$(MYAPP_VERSION)/myapp.debug
    $(TARGET_STRIP) --strip-debug \
        $(TARGET_DIR)/usr/bin/myapp
    $(TARGET_OBJCOPY) --add-gnu-debuglink=\
        $(BUILD_DIR)/myapp-$(MYAPP_VERSION)/myapp.debug \
        $(TARGET_DIR)/usr/bin/myapp
endef
MYAPP_POST_INSTALL_TARGET_HOOKS += MYAPP_SPLIT_DEBUG
```

The `.debug` files live on the host; GDB loads them automatically when you set
`debug-file-directory`.

---

## 2. Remote GDB Debugging with gdbserver

### Architecture Overview

```
  HOST                                    TARGET
  ─────────────────────────────────────── ─────────────────────────────
  arm-buildroot-linux-gnueabihf-gdb       gdbserver
       │                                       │
       │  ← RSP (Remote Serial Protocol) →     │
       │        TCP :2345  or  /dev/ttyUSB0    │
       │                                       │
  Symbols + source on host            Stripped binary on target
  $(STAGING_DIR)/usr/lib              /usr/bin/myapp
```

Remote GDB uses the GDB Remote Serial Protocol (RSP). The `gdbserver` stub runs
on the target and acts as a proxy, while the full GDB (with symbol files and
source) runs on the host.

### Enabling gdbserver in Buildroot

```
make menuconfig
  → Target packages
      → Debugging, profiling and benchmark
          → [*] gdb                  (BR2_PACKAGE_GDB)
          → [*]   gdb server         (BR2_PACKAGE_GDB_SERVER)
          → [*]   gdb debugger       (BR2_PACKAGE_GDB_DEBUGGER)  ← optional
```

Or directly in `.config`:

```ini
BR2_PACKAGE_GDB=y
BR2_PACKAGE_GDB_SERVER=y
BR2_PACKAGE_HOST_GDB=y        # builds cross-GDB on the host
```

### Starting a Debug Session

**On the target:**

```bash
# Attach to a running process
gdbserver :2345 --attach $(pidof myapp)

# Start a new process under gdbserver
gdbserver :2345 /usr/bin/myapp --arg1 --arg2

# Using a serial line instead of TCP
gdbserver /dev/ttyAMA0 /usr/bin/myapp
```

**On the host:**

```bash
# Use the cross-GDB built by Buildroot
output/host/bin/arm-buildroot-linux-gnueabihf-gdb \
    output/staging/usr/bin/myapp

(gdb) set sysroot output/staging
(gdb) set debug-file-directory output/build/myapp-1.0/
(gdb) target remote 192.168.1.100:2345
(gdb) break main
(gdb) continue
```

### GDB Init Script (.gdbinit)

```gdb
# .gdbinit — place in project root
set architecture arm
set sysroot /path/to/buildroot/output/staging
set solib-search-path /path/to/buildroot/output/staging/usr/lib
set debug-file-directory /path/to/buildroot/output/build

define connect
  target remote 192.168.1.100:2345
end

define reload
  file output/staging/usr/bin/myapp
  connect
end
```

### Python Pretty-Printers for Embedded Types

```python
# gdb_printers.py — register with: source gdb_printers.py
import gdb

class RingBufferPrinter:
    """Pretty-print a C ring buffer struct."""
    def __init__(self, val):
        self.val = val

    def to_string(self):
        head  = int(self.val['head'])
        tail  = int(self.val['tail'])
        size  = int(self.val['size'])
        used  = (head - tail) % size
        return (f"RingBuffer(head={head}, tail={tail}, "
                f"size={size}, used={used})")

def register_printers(objfile):
    gdb.pretty_printers.append(
        lambda val: RingBufferPrinter(val)
        if str(val.type) == 'struct ring_buf' else None
    )

register_printers(None)
```

---

## 3. System Call Tracing with strace

### Enabling strace

```
make menuconfig
  → Target packages
      → Debugging, profiling and benchmark
          → [*] strace    (BR2_PACKAGE_STRACE)
```

```ini
# .config
BR2_PACKAGE_STRACE=y
```

### Essential strace Invocations

```bash
# Trace all syscalls of a new process
strace /usr/bin/myapp

# Trace a running process by PID
strace -p $(pidof myapp)

# Filter to specific syscalls (open, read, write)
strace -e trace=openat,read,write /usr/bin/myapp

# Show timing between syscalls (latency analysis)
strace -T -e trace=read,write /usr/bin/myapp

# Count syscall frequency and time
strace -c /usr/bin/myapp

# Follow child processes (fork/exec)
strace -f /usr/bin/init

# Save output to file
strace -o /tmp/trace.log /usr/bin/myapp

# Trace file descriptor activity with string output
strace -e trace=read,write -s 256 /usr/bin/myapp
```

### Example strace Output Analysis

```
# Typical output — opening a SPI device
openat(AT_FDCWD, "/dev/spidev0.0", O_RDWR) = 3
ioctl(3, SPI_IOC_WR_MODE, [0])          = 0
ioctl(3, SPI_IOC_WR_MAX_SPEED_HZ, [1000000]) = 0
write(3, "\x01\x02\x03", 3)             = 3
read(3, "\x00\xAB\xCD", 3)              = 3
close(3)                                 = 0
```

### strace Summary Output (-c flag)

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.23    0.012340          82       150         3 read
 30.11    0.008210          54       152           write
 12.44    0.003392         339        10           openat
  7.89    0.002150          71        30           ioctl
  4.33    0.001180          58        20           mmap
------ ----------- ----------- --------- --------- ----------------
100.00    0.027272                   362         3 total
```

---

## 4. Performance Profiling with perf

### Cross-Building perf in Buildroot

`perf` is part of the Linux kernel source tree. Buildroot builds it from the
same kernel source used for the target image.

```
make menuconfig
  → Kernel
      → [*] Linux Kernel
  → Target packages
      → Debugging, profiling and benchmark
          → [*] perf   (BR2_PACKAGE_LINUX_TOOLS_PERF)
```

```ini
# .config
BR2_PACKAGE_LINUX_TOOLS=y
BR2_PACKAGE_LINUX_TOOLS_PERF=y
```

Buildroot automatically uses `ARCH` and `CROSS_COMPILE` from the selected
toolchain when building `perf`. The resulting binary is installed to
`/usr/bin/perf` on the target.

### Kernel Configuration for perf

Several kernel options must be enabled for full perf functionality:

```
# Linux kernel menuconfig
CONFIG_PERF_EVENTS=y
CONFIG_HW_PERF_EVENTS=y          # hardware counters
CONFIG_TRACEPOINTS=y
CONFIG_FTRACE=y
CONFIG_KPROBES=y
CONFIG_UPROBE_EVENTS=y
CONFIG_FRAME_POINTER=y            # call graph support
```

### Common perf Commands on Target

```bash
# CPU-wide sampling (1000 Hz, 10 seconds)
perf stat -a sleep 10

# Record a profile of myapp
perf record -g /usr/bin/myapp

# Show annotated report
perf report --stdio

# Count hardware events for one run
perf stat -e cycles,instructions,cache-misses /usr/bin/myapp

# Real-time top-like view
perf top -g

# Tracepoint — track scheduler events
perf record -e sched:sched_switch -a sleep 5
perf script

# Record with call graph (frame pointer method)
perf record -g fp /usr/bin/myapp
```

### perf stat Example Output

```
 Performance counter stats for '/usr/bin/myapp':

          1,234,567      cycles                    #    0.89 GHz
          2,345,678      instructions              #    1.90  insn per cycle
             12,345      cache-misses              #    5.26% of all cache refs
              1,234      branch-misses             #    0.89% of all branches
            234,567      context-switches

       0.001385691 seconds time elapsed
```

### Off-CPU Analysis with perf sched

```bash
# Record all scheduling events
perf sched record -- /usr/bin/myapp

# Show per-task latency statistics
perf sched latency

# Replay and visualize timing
perf sched timehist
```

---

## 5. JTAG Integration via OpenOCD

### What Is OpenOCD?

OpenOCD (Open On-Chip Debugger) is a free software tool that bridges JTAG/SWD
hardware adapters to GDB's RSP protocol. It provides:

- Flash programming
- CPU halt/resume/step
- Register and memory access
- OS-aware thread awareness (with RTOS plugins)
- Semihosting support

### Enabling OpenOCD in Buildroot

```
make menuconfig
  → Host utilities
      → [*] host openocd   (BR2_PACKAGE_HOST_OPENOCD)
```

```ini
BR2_PACKAGE_HOST_OPENOCD=y
```

OpenOCD is a host-side tool; it does not run on the target.

### Typical OpenOCD Configuration

```tcl
# openocd.cfg — example for STM32MP1 + FTDI adapter
source [find interface/ftdi/olimex-arm-usb-tiny-h.cfg]
source [find target/stm32mp15x.cfg]

adapter speed 4000

$_TARGETNAME configure -event reset-init {
    # Clock setup after reset
    mww 0x50000008 0x00000001
}
```

### Debug Session with OpenOCD + GDB

**Start OpenOCD (host terminal 1):**

```bash
output/host/bin/openocd \
    -f board/myboard/openocd.cfg \
    -c "init" \
    -c "reset halt"
```

**Connect GDB (host terminal 2):**

```bash
output/host/bin/arm-buildroot-linux-gnueabihf-gdb \
    output/staging/usr/bin/myapp

(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) break app_main
(gdb) continue
```

### JTAG + gdbserver Dual-Mode (Linux Running)

When embedded Linux is running, you can combine OpenOCD for hardware access and
gdbserver for application debugging simultaneously:

```
  HOST
  ┌──────────────────────────────────────────────────────┐
  │  GDB (application debug)    GDB (kernel/HW debug)    │
  │        │                          │                  │
  │        │ :2345 (TCP/RSP)          │ :3333 (OpenOCD)  │
  └────────┼──────────────────────────┼──────────────────┘
           │                          │
  TARGET   │                          │ (JTAG cable)
  ┌────────┼──────────────────────────┼──────────────────┐
  │  gdbserver ← myapp         OpenOCD stub / JTAG HW    │
  └──────────────────────────────────────────────────────┘
```

---

## 6. Code Examples — C

### 6.1 Signal Handler with GDB-Friendly Stack Frames

```c
/* crash_handler.c
 * Compile: arm-buildroot-linux-gnueabihf-gcc -g3 -Og -fno-omit-frame-pointer
 *          -o crash_handler crash_handler.c
 *
 * Demonstrates: async-signal-safe crash handler that preserves stack
 * for post-mortem GDB analysis.
 */
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <execinfo.h>  /* backtrace() */
#include <string.h>
#include <sys/ucontext.h>

#define BT_BUF_SIZE 32

/* Async-signal-safe integer to hex string */
static void uint_to_hex(unsigned long v, char *buf, size_t len)
{
    static const char hex[] = "0123456789abcdef";
    int i;
    memset(buf, '0', len);
    buf[len] = '\0';
    for (i = (int)len - 1; i >= 0 && v; --i, v >>= 4)
        buf[i] = hex[v & 0xF];
}

static void crash_handler(int sig, siginfo_t *si, void *ctx)
{
    void   *bt[BT_BUF_SIZE];
    int     nframes;
    char    hex[17];
    ucontext_t *uc = (ucontext_t *)ctx;

    /* Write crash banner — only async-signal-safe calls allowed here */
    const char banner[] = "\n=== CRASH DETECTED ===\n";
    write(STDERR_FILENO, banner, sizeof(banner) - 1);

    /* Print faulting address */
    uint_to_hex((unsigned long)si->si_addr, hex, 16);
    write(STDERR_FILENO, "Fault addr: 0x", 14);
    write(STDERR_FILENO, hex, 16);
    write(STDERR_FILENO, "\n", 1);

#if defined(__arm__)
    /* ARM: PC is in uc_mcontext.arm_pc */
    uint_to_hex((unsigned long)uc->uc_mcontext.arm_pc, hex, 16);
    write(STDERR_FILENO, "PC:         0x", 14);
    write(STDERR_FILENO, hex, 16);
    write(STDERR_FILENO, "\n", 1);
#endif

    /* Collect and dump backtrace */
    nframes = backtrace(bt, BT_BUF_SIZE);
    backtrace_symbols_fd(bt, nframes, STDERR_FILENO);

    /* Re-raise to get a core dump */
    signal(sig, SIG_DFL);
    raise(sig);
}

static void register_crash_handlers(void)
{
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_sigaction = crash_handler;
    sa.sa_flags     = SA_SIGINFO | SA_RESETHAND;
    sigemptyset(&sa.sa_mask);

    sigaction(SIGSEGV, &sa, NULL);
    sigaction(SIGBUS,  &sa, NULL);
    sigaction(SIGABRT, &sa, NULL);
    sigaction(SIGFPE,  &sa, NULL);
}

/* Deliberately buggy function for demonstration */
static void level3(void)
{
    volatile int *p = (int *)0xDEADBEEF;
    *p = 42;  /* SIGSEGV — caught by handler */
}

static void level2(void) { level3(); }
static void level1(void) { level2(); }

int main(void)
{
    register_crash_handlers();
    printf("Crash handler demo — triggering fault...\n");
    fflush(stdout);
    level1();
    return 0;  /* not reached */
}
```

### 6.2 strace-Friendly SPI Device Driver Interaction

```c
/* spi_trace_demo.c
 * Compile: arm-buildroot-linux-gnueabihf-gcc -g -o spi_demo spi_trace_demo.c
 *
 * Run with strace to observe every ioctl and read/write syscall:
 *   strace -e trace=openat,ioctl,read,write,close ./spi_demo
 */
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/spi/spidev.h>

#define SPI_DEVICE  "/dev/spidev0.0"
#define SPI_SPEED   1000000   /* 1 MHz */
#define SPI_BITS    8

static int spi_open(const char *dev)
{
    int fd = open(dev, O_RDWR);  /* strace: openat(...) = fd */
    if (fd < 0) {
        perror("open");
        return -1;
    }

    uint8_t mode  = SPI_MODE_0;
    uint8_t bits  = SPI_BITS;
    uint32_t speed = SPI_SPEED;

    /* strace will show each ioctl with decoded arguments */
    if (ioctl(fd, SPI_IOC_WR_MODE,         &mode)  < 0 ||
        ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits)  < 0 ||
        ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ,  &speed) < 0) {
        perror("ioctl");
        close(fd);
        return -1;
    }
    return fd;
}

static int spi_transfer(int fd, const uint8_t *tx, uint8_t *rx, size_t len)
{
    struct spi_ioc_transfer tr = {
        .tx_buf        = (unsigned long)tx,
        .rx_buf        = (unsigned long)rx,
        .len           = len,
        .speed_hz      = SPI_SPEED,
        .bits_per_word = SPI_BITS,
        .delay_usecs   = 0,
    };
    /* Full-duplex transfer — one ioctl, visible in strace */
    return ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
}

int main(void)
{
    int     fd;
    uint8_t tx[] = { 0x01, 0x02, 0x03, 0x04 };
    uint8_t rx[sizeof(tx)];

    fd = spi_open(SPI_DEVICE);
    if (fd < 0)
        return EXIT_FAILURE;

    memset(rx, 0, sizeof(rx));
    if (spi_transfer(fd, tx, rx, sizeof(tx)) < 0) {
        perror("transfer");
        close(fd);
        return EXIT_FAILURE;
    }

    printf("TX: %02x %02x %02x %02x\n", tx[0], tx[1], tx[2], tx[3]);
    printf("RX: %02x %02x %02x %02x\n", rx[0], rx[1], rx[2], rx[3]);

    close(fd);  /* strace: close(fd) = 0 */
    return EXIT_SUCCESS;
}
```

### 6.3 perf-Annotatable Hot Loop

```c
/* perf_hot_loop.c
 * Compile: arm-buildroot-linux-gnueabihf-gcc -g -O1 -fno-inline
 *          -o perf_demo perf_hot_loop.c
 *
 * Profile:
 *   perf record -g ./perf_demo
 *   perf report --stdio --no-children
 *   perf annotate matrix_multiply
 */
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N 128

static float A[N][N], B[N][N], C[N][N];

/* Naive matrix multiply — intentionally shows up hot in perf */
static void __attribute__((noinline)) matrix_multiply(void)
{
    int i, j, k;
    for (i = 0; i < N; i++)
        for (j = 0; j < N; j++) {
            C[i][j] = 0.0f;
            for (k = 0; k < N; k++)
                C[i][j] += A[i][k] * B[k][j];
        }
}

static void init_matrices(void)
{
    int i, j;
    srand(42);
    for (i = 0; i < N; i++)
        for (j = 0; j < N; j++) {
            A[i][j] = (float)rand() / RAND_MAX;
            B[i][j] = (float)rand() / RAND_MAX;
        }
}

int main(void)
{
    struct timespec t0, t1;
    int iterations = 100;

    init_matrices();
    clock_gettime(CLOCK_MONOTONIC, &t0);

    for (int i = 0; i < iterations; i++)
        matrix_multiply();

    clock_gettime(CLOCK_MONOTONIC, &t1);

    double elapsed = (t1.tv_sec - t0.tv_sec) +
                     (t1.tv_nsec - t0.tv_nsec) / 1e9;
    printf("%.3f s for %d iterations (%.3f ms each)\n",
           elapsed, iterations, elapsed * 1000.0 / iterations);

    /* Prevent dead-code elimination */
    printf("C[0][0] = %f\n", C[0][0]);
    return 0;
}
```

---

## 7. Code Examples — C++

### 7.1 RAII Debug Session Guard

```cpp
// debug_session.cpp
// Compile: arm-buildroot-linux-gnueabihf-g++ -std=c++17 -g3 -Og
//          -o debug_session debug_session.cpp
//
// Demonstrates RAII wrappers that interact cleanly with GDB:
//   - GDB can step into constructors/destructors
//   - Automatic cleanup on exception (visible via strace close() calls)

#include <iostream>
#include <stdexcept>
#include <string>
#include <cstring>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

// ── RAII file descriptor ──────────────────────────────────────────────────────
class FileDescriptor {
    int fd_;
public:
    explicit FileDescriptor(const std::string &path, int flags)
        : fd_(::open(path.c_str(), flags))
    {
        if (fd_ < 0)
            throw std::system_error(errno, std::generic_category(),
                                    "open(" + path + ")");
        std::clog << "[fd=" << fd_ << "] opened " << path << "\n";
    }

    ~FileDescriptor()
    {
        if (fd_ >= 0) {
            std::clog << "[fd=" << fd_ << "] closing\n";
            ::close(fd_);  // visible in strace as close(fd)
        }
    }

    // Non-copyable, movable
    FileDescriptor(const FileDescriptor &)            = delete;
    FileDescriptor &operator=(const FileDescriptor &) = delete;
    FileDescriptor(FileDescriptor &&o) noexcept : fd_(o.fd_) { o.fd_ = -1; }

    int get() const noexcept { return fd_; }

    template <typename Req, typename Arg>
    int ioctl(Req req, Arg &&arg)
    {
        int r = ::ioctl(fd_, req, &arg);
        if (r < 0)
            throw std::system_error(errno, std::generic_category(), "ioctl");
        return r;
    }
};

// ── Scoped GDB breakpoint marker ─────────────────────────────────────────────
// Place a nop that GDB can intercept without modifying source.
// Use:  break DebugBreakpoint::hit
struct DebugBreakpoint {
    [[gnu::noinline]] static void hit() { asm volatile("nop"); }
};

// ── Device abstraction ────────────────────────────────────────────────────────
class GpioDevice {
    FileDescriptor fd_;
    unsigned       line_;
public:
    GpioDevice(const std::string &chip, unsigned line)
        : fd_("/dev/" + chip, O_RDONLY), line_(line)
    {
        // GDB: 'break GpioDevice::GpioDevice' to inspect construction
    }

    void toggle()
    {
        // Annotated with a debug breakpoint the GDB user can catch
        DebugBreakpoint::hit();
        std::cout << "GPIO " << line_ << " toggled on fd=" << fd_.get() << "\n";
    }
};

int main()
{
    try {
        GpioDevice led("gpiochip0", 17);
        led.toggle();
        led.toggle();
        // Destructor runs here — close() visible in strace
    } catch (const std::system_error &e) {
        std::cerr << "Device error: " << e.what() << "\n";
        return 1;
    }
    return 0;
}
```

### 7.2 Exception-Safe perf_event Wrapper

```cpp
// perf_event_wrapper.cpp
// Compile: arm-buildroot-linux-gnueabihf-g++ -std=c++17 -g
//          -o perf_wrap perf_event_wrapper.cpp
//
// Wraps the perf_event_open(2) syscall in a C++ RAII type.
// Useful for in-process micro-benchmarking visible to 'perf stat'.

#include <iostream>
#include <stdexcept>
#include <system_error>
#include <cstring>
#include <cstdint>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/syscall.h>
#include <linux/perf_event.h>

// ── perf_event_open syscall wrapper ──────────────────────────────────────────
static long perf_event_open(struct perf_event_attr *attr,
                             pid_t pid, int cpu, int group_fd,
                             unsigned long flags)
{
    return syscall(SYS_perf_event_open, attr, pid, cpu, group_fd, flags);
}

// ── RAII Hardware Counter ─────────────────────────────────────────────────────
class HwCounter {
    int      fd_   = -1;
    uint64_t type_;
    uint64_t config_;
public:
    HwCounter(uint64_t type, uint64_t config)
        : type_(type), config_(config)
    {
        struct perf_event_attr attr{};
        attr.type           = type;
        attr.size           = sizeof(attr);
        attr.config         = config;
        attr.disabled       = 1;
        attr.exclude_kernel = 1;
        attr.exclude_hv     = 1;

        fd_ = static_cast<int>(
            perf_event_open(&attr, 0 /* self */, -1 /* any CPU */,
                            -1 /* no group */, 0));
        if (fd_ < 0)
            throw std::system_error(errno, std::generic_category(),
                                    "perf_event_open");
    }

    ~HwCounter() { if (fd_ >= 0) ::close(fd_); }

    void reset()  { ioctl(fd_, PERF_EVENT_IOC_RESET,  0); }
    void enable() { ioctl(fd_, PERF_EVENT_IOC_ENABLE, 0); }
    void disable(){ ioctl(fd_, PERF_EVENT_IOC_DISABLE,0); }

    uint64_t read_count() const
    {
        uint64_t count = 0;
        if (::read(fd_, &count, sizeof(count)) != sizeof(count))
            throw std::system_error(errno, std::generic_category(),
                                    "perf read");
        return count;
    }
};

// ── RAII measurement scope ────────────────────────────────────────────────────
class PerfScope {
    HwCounter &ctr_;
public:
    explicit PerfScope(HwCounter &c) : ctr_(c) { ctr_.reset(); ctr_.enable(); }
    ~PerfScope() { ctr_.disable(); }
};

// ── Example workload ──────────────────────────────────────────────────────────
static volatile uint64_t sink;

static void workload(unsigned n)
{
    uint64_t sum = 0;
    for (unsigned i = 0; i < n; ++i)
        sum += i * i;
    sink = sum;
}

int main()
{
    try {
        HwCounter cycles(PERF_TYPE_HARDWARE, PERF_COUNT_HW_CPU_CYCLES);
        HwCounter insns (PERF_TYPE_HARDWARE, PERF_COUNT_HW_INSTRUCTIONS);

        constexpr unsigned N = 1'000'000;

        {
            PerfScope sc(cycles);
            PerfScope si(insns);
            workload(N);
        }

        uint64_t c = cycles.read_count();
        uint64_t i = insns.read_count();

        std::cout << "Cycles:       " << c << "\n"
                  << "Instructions: " << i << "\n"
                  << "IPC:          " << (i ? (double)i / c : 0.0) << "\n";
    } catch (const std::system_error &e) {
        std::cerr << "perf error: " << e.what()
                  << " (need CONFIG_PERF_EVENTS=y)\n";
        return 1;
    }
    return 0;
}
```

---

## 8. Code Examples — Rust

### 8.1 Buildroot Cross-Compilation Setup for Rust

```toml
# .cargo/config.toml — place in project root

[target.armv7-unknown-linux-gnueabihf]
linker = "arm-buildroot-linux-gnueabihf-gcc"
ar     = "arm-buildroot-linux-gnueabihf-ar"

[env]
PKG_CONFIG_SYSROOT_DIR  = { value = "/path/to/buildroot/output/staging" }
PKG_CONFIG_ALLOW_CROSS  = "1"
```

```makefile
# package/myrust-app/myrust-app.mk
MYRUST_APP_VERSION = 1.0.0
MYRUST_APP_SITE    = $(TOPDIR)/package/myrust-app/src
MYRUST_APP_SITE_METHOD = local

define MYRUST_APP_BUILD_CMDS
    $(TARGET_MAKE_ENV) \
    CARGO_HOME=$(BUILD_DIR)/cargo-home \
    $(HOST_DIR)/bin/cargo build \
        --target $(RUSTC_TARGET_NAME) \
        --release \
        --manifest-path $(@D)/Cargo.toml
endef

define MYRUST_APP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 \
        $(@D)/target/$(RUSTC_TARGET_NAME)/release/myrust-app \
        $(TARGET_DIR)/usr/bin/myrust-app
endef

$(eval $(generic-package))
```

### 8.2 Rust Crash Reporter with Backtrace

```rust
// src/main.rs
// Cargo.toml dependencies:
//   [dependencies]
//   backtrace = "0.3"
//   libc      = "1"
//
// Build (cross): cargo build --target armv7-unknown-linux-gnueabihf
// Debug build:   RUST_BACKTRACE=full cargo build --target ...
//
// The panic hook installs a handler that mimics gdbserver crash reports
// and writes a file that OpenOCD scripts can detect.

use std::panic;
use std::fs;
use std::io::Write;
use backtrace::Backtrace;

fn install_panic_hook() {
    panic::set_hook(Box::new(|info| {
        let bt = Backtrace::new();

        // Write crash log — visible to strace as open/write/close
        let msg = format!(
            "=== RUST PANIC ===\n{info}\n\nBacktrace:\n{bt:?}\n"
        );

        // stderr for immediate visibility
        eprintln!("{msg}");

        // File for post-mortem retrieval via JTAG/gdb
        if let Ok(mut f) = fs::File::create("/tmp/rust_crash.log") {
            let _ = f.write_all(msg.as_bytes());
        }
    }));
}

// ── Ring buffer (GDB pretty-printer target) ───────────────────────────────────
struct RingBuffer<T, const N: usize> {
    buf:  [Option<T>; N],
    head: usize,
    tail: usize,
    len:  usize,
}

impl<T: Copy + Default, const N: usize> RingBuffer<T, N> {
    fn new() -> Self {
        Self {
            buf:  [None; N],
            head: 0,
            tail: 0,
            len:  0,
        }
    }

    fn push(&mut self, val: T) -> bool {
        if self.len == N {
            return false;  // full
        }
        self.buf[self.head] = Some(val);
        self.head = (self.head + 1) % N;
        self.len += 1;
        true
    }

    fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        let val = self.buf[self.tail].take();
        self.tail = (self.tail + 1) % N;
        self.len -= 1;
        val
    }

    fn len(&self)  -> usize { self.len  }
    fn is_full(&self) -> bool { self.len == N }
}

// ── SPI abstraction (strace-observable) ──────────────────────────────────────
mod spi {
    use std::fs::{File, OpenOptions};
    use std::io;
    use std::os::unix::io::AsRawFd;

    pub struct Device {
        file: File,
    }

    impl Device {
        pub fn open(path: &str) -> io::Result<Self> {
            // strace: openat(AT_FDCWD, path, O_RDWR) = fd
            let file = OpenOptions::new().read(true).write(true).open(path)?;
            Ok(Self { file })
        }

        pub fn fd(&self) -> i32 {
            self.file.as_raw_fd()
        }
    }

    // Drop calls close(fd) — visible in strace
    impl Drop for Device {
        fn drop(&mut self) {
            // File::drop() calls close() automatically
        }
    }
}

fn demonstrate_ring_buffer() {
    let mut rb: RingBuffer<u32, 8> = RingBuffer::new();

    for i in 0..8u32 {
        rb.push(i);
    }
    println!("Ring buffer full: {}", rb.is_full());

    // GDB: 'print rb' with a pretty-printer shows head/tail/len
    while let Some(v) = rb.pop() {
        print!("{v} ");
    }
    println!();
}

fn main() {
    install_panic_hook();

    println!("Rust embedded debug demo");

    demonstrate_ring_buffer();

    // Demonstrate SPI open (strace-observable)
    match spi::Device::open("/dev/spidev0.0") {
        Ok(dev) => println!("SPI opened on fd={}", dev.fd()),
        Err(e)  => eprintln!("SPI unavailable: {e}"),
    }
    // spi::Device::drop() called here — close() visible in strace

    // Trigger a controlled panic to exercise the panic hook
    // Comment out for normal operation
    // panic!("Intentional panic for demo");
}
```

### 8.3 Rust perf_event Bindings

```rust
// src/perf.rs — safe wrapper around perf_event_open(2)
//
// Cargo.toml:
//   [dependencies]
//   libc = "1"

use libc::{c_int, c_long, c_ulong, pid_t, syscall, SYS_perf_event_open};
use std::fs::File;
use std::io::{self, Read};
use std::os::unix::io::{FromRawFd, RawFd};
use std::mem;

// Minimal perf_event_attr subset (full struct is 128 bytes)
#[repr(C)]
struct PerfEventAttr {
    kind:           u32,
    size:           u32,
    config:         u64,
    sample_period:  u64,
    sample_type:    u64,
    read_format:    u64,
    flags:          u64,
    _reserved:      [u64; 16],
}

const PERF_TYPE_HARDWARE:       u32 = 0;
const PERF_COUNT_HW_CPU_CYCLES: u64 = 0;
const PERF_COUNT_HW_INSTRUCTIONS: u64 = 1;

const PERF_EVENT_IOC_RESET:   c_ulong = 0x2403;
const PERF_EVENT_IOC_ENABLE:  c_ulong = 0x2400;
const PERF_EVENT_IOC_DISABLE: c_ulong = 0x2401;

pub struct Counter {
    file: File,
}

impl Counter {
    pub fn new(kind: u32, config: u64) -> io::Result<Self> {
        let mut attr: PerfEventAttr = unsafe { mem::zeroed() };
        attr.kind   = kind;
        attr.size   = mem::size_of::<PerfEventAttr>() as u32;
        attr.config = config;
        // disabled=1, exclude_kernel=1, exclude_hv=1 packed into flags
        attr.flags  = (1 << 0) | (1 << 5) | (1 << 6);

        let fd: c_long = unsafe {
            syscall(
                SYS_perf_event_open,
                &attr as *const _ as c_long,
                0i32,   // pid=0: current process
                -1i32,  // cpu=-1: any CPU
                -1i32,  // group_fd=-1: no group
                0u64,   // flags
            )
        };

        if fd < 0 {
            return Err(io::Error::last_os_error());
        }

        Ok(Self {
            file: unsafe { File::from_raw_fd(fd as RawFd) },
        })
    }

    fn ioctl(&self, request: c_ulong) -> io::Result<()> {
        use std::os::unix::io::AsRawFd;
        let ret = unsafe { libc::ioctl(self.file.as_raw_fd(), request, 0) };
        if ret < 0 {
            Err(io::Error::last_os_error())
        } else {
            Ok(())
        }
    }

    pub fn reset(&self)   -> io::Result<()> { self.ioctl(PERF_EVENT_IOC_RESET)   }
    pub fn enable(&self)  -> io::Result<()> { self.ioctl(PERF_EVENT_IOC_ENABLE)  }
    pub fn disable(&self) -> io::Result<()> { self.ioctl(PERF_EVENT_IOC_DISABLE) }

    pub fn count(&mut self) -> io::Result<u64> {
        let mut buf = [0u8; 8];
        self.file.read_exact(&mut buf)?;
        Ok(u64::from_ne_bytes(buf))
    }
}

// Usage example
pub fn benchmark<F: Fn()>(label: &str, f: F) -> io::Result<()> {
    let mut cycles = Counter::new(PERF_TYPE_HARDWARE, PERF_COUNT_HW_CPU_CYCLES)?;
    let mut insns  = Counter::new(PERF_TYPE_HARDWARE, PERF_COUNT_HW_INSTRUCTIONS)?;

    cycles.reset()?; cycles.enable()?;
    insns.reset()?;  insns.enable()?;

    f();

    cycles.disable()?;
    insns.disable()?;

    let c = cycles.count()?;
    let i = insns.count()?;

    println!("[{label}] cycles={c}  instructions={i}  IPC={:.2}",
        if c > 0 { i as f64 / c as f64 } else { 0.0 });

    Ok(())
}
```

---

## 9. Debugging Workflow Diagrams

### 9.1 Remote GDB Session Flow

```
  HOST                                      TARGET (192.168.1.100)
  ═══════════════════════════════════       ═══════════════════════════

  $ make menuconfig                         [ Linux running ]
    [*] BR2_ENABLE_DEBUG                    $ gdbserver :2345 /usr/bin/app
    [*] BR2_PACKAGE_GDB_SERVER              Listening on port 2345...
    [*] BR2_PACKAGE_HOST_GDB
  $ make
             │ build
             ▼
  output/host/bin/
    arm-...-gdb ──────── TCP :2345 ──────► gdbserver
                    RSP packets                 │
  (gdb) set sysroot                             │ ptrace(2)
        output/staging    ◄────────────────────► myapp
  (gdb) break sensor_read                    [stopped at bp]
  (gdb) continue
  (gdb) print temperature                   [reads memory]
  (gdb) backtrace
  #0  sensor_read (dev=0x...)
  #1  main ()
```

### 9.2 strace System Call Visibility

```
  Application Layer
  ┌─────────────────────────────────────────────┐
  │  myapp                                      │
  │    open("/dev/i2c-1", O_RDWR)   ──────────► │──► strace intercepts here
  │    ioctl(fd, I2C_SLAVE, 0x48)   ──────────► │──► prints decoded syscall
  │    write(fd, "\x00", 1)         ──────────► │──► shows buffer content
  │    read(fd, buf, 2)             ──────────► │──► shows return value
  │    close(fd)                    ──────────► │
  └─────────────────────────────────────────────┘
            │ ptrace(PTRACE_SYSCALL)
            ▼
  ┌─────────────────────────────────────────────┐
  │  Linux Kernel                               │
  │    sys_openat → VFS → i2c-dev driver        │
  │    sys_ioctl  → i2c_check_functionality()   │
  │    sys_write  → i2c_master_send()           │
  │    sys_read   → i2c_master_recv()           │
  └─────────────────────────────────────────────┘

  strace output:
  openat(AT_FDCWD, "/dev/i2c-1", O_RDWR)  = 3     <0.000234>
  ioctl(3, I2C_SLAVE, 0x48)               = 0     <0.000019>
  write(3, "\x00", 1)                     = 1     <0.000215>
  read(3, "\x1a\x80", 2)                  = 2     <0.001342>
  close(3)                                = 0     <0.000018>
```

### 9.3 perf Profiling Architecture

```
  ┌──────────── HOST ─────────────────────────────────────────────┐
  │                                                               │
  │  perf report ◄── perf.data ◄── scp from target                │
  │      │                                                        │
  │  Annotated source / flame graph                               │
  └───────────────────────────────────────────────────────────────┘
                             │ scp perf.data
  ┌──────────── TARGET ───────────────────────────────────────────┐
  │                                                               │
  │  perf record -g ./myapp                                       │
  │       │                                                       │
  │       ▼                                                       │
  │  ┌─────────────────────────────────────────────┐              │
  │  │  Linux PMU (Performance Monitoring Unit)    │              │
  │  │                                             │              │
  │  │  Hardware Events          Software Events   │              │
  │  │  ─────────────            ───────────────   │              │
  │  │  CPU cycles               Page faults       │              │
  │  │  Instructions             Context switches  │              │
  │  │  Cache misses             Minor/Major faults│              │
  │  │  Branch mispredictions    CPU migrations    │              │
  │  └─────────────────────────────────────────────┘              │
  │       │ interrupt every N events                              │
  │       ▼                                                       │
  │  Sample: {PC, call-chain, timestamp, pid, cpu}                │
  │  Written to: perf.data                                        │
  └───────────────────────────────────────────────────────────────┘
```

### 9.4 OpenOCD + JTAG Chain

```
  HOST                 USB/FTDI              TARGET BOARD
  ─────────────        ─────────────         ──────────────────────
                                             ┌──────────────┐
  GDB client           JTAG Adapter          │  ARM SoC     │
  arm-...-gdb  ──RSP──► OpenOCD ──JTAG──────►│  CoreSight   │
  port :3333           (host daemon)    TCK  │  DAP         │
                                        TMS  │     │        │
                                        TDI  │     ▼        │
                                        TDO  │  CPU cores   │
                                        nTRST│  Cortex-A/M  │
                       OpenOCD               │     │        │
                       commands:             │  AHB/AXI bus │
                       - reset halt          │     │        │
                       - reg                 │  Peripherals │
                       - mdw 0x40000000      │  Flash ctrl  │
                       - flash write_image   └──────────────┘

  OpenOCD also exposes:
  - Telnet CLI  :4444
  - TCL server  :6666
  - GDB RSP     :3333
```

### 9.5 Buildroot Debug Build Decision Tree

```
  START: Need to debug embedded target?
              │
              ▼
  ┌───────────────────────┐
  │ Know the crash addr?  │
  └───────────────────────┘
       │ YES             │ NO
       ▼                 ▼
  ┌──────────┐      ┌─────────────────────────────┐
  │ addr2line│      │   Is Linux running?         │
  │ + map    │      └─────────────────────────────┘
  └──────────┘           │ YES           │ NO
                         ▼               ▼
                  ┌────────────┐   ┌──────────────┐
                  │ Behavior   │   │  OpenOCD +   │
                  │ correct?   │   │  JTAG debug  │
                  └────────────┘   └──────────────┘
                  │ YES  │ NO
                  ▼      ▼
             ┌──────┐ ┌──────────────────────┐
             │ perf │ │ gdbserver or strace? │
             │ stat │ └──────────────────────┘
             └──────┘    │ source known │ syscall issue
                         ▼              ▼
                    ┌─────────┐   ┌─────────┐
                    │ remote  │   │ strace  │
                    │   GDB   │   │ -e trace│
                    └─────────┘   └─────────┘
```

---

## 10. Summary

This chapter covered the complete debugging toolkit available in Buildroot for
embedded Linux targets.

**BR2_ENABLE_DEBUG** is the foundational switch. Enabling it preserves DWARF
debug information across all packages, disables binary stripping, and switches
the optimization level to `-Og`. Split debug symbols (`.debug` files on host,
stripped binary on target) offer a practical middle ground for production
images that still need to be debuggable.

**Remote GDB with gdbserver** provides source-level interactive debugging
without running full GDB on the resource-constrained target. The host-side
cross-GDB loads symbols from `output/staging`, the target runs the lightweight
`gdbserver` stub, and RSP packets flow over TCP (or serial). Python
pretty-printers and `.gdbinit` scripts greatly improve the session experience
for embedded data structures.

**strace** (`BR2_PACKAGE_STRACE`) is the quickest path to understanding
unexpected application behavior. It requires no recompilation; it attaches via
`ptrace` and decodes every system call with argument types, return values, and
timing. The `-c` flag produces statistical summaries that immediately identify
slow or frequently failing syscalls. It is especially valuable for debugging
device driver interactions (I²C, SPI, GPIO) where the kernel ABI is the
interface.

**perf** is cross-compiled from the kernel source tree
(`BR2_PACKAGE_LINUX_TOOLS_PERF`) and requires matching `CONFIG_PERF_EVENTS` in
the kernel. It provides both sampling-based CPU profiling (`perf record`) and
counting-based measurements (`perf stat`) using hardware PMU counters. Profile
data can be transferred to the host for visualization with `perf report`, flame
graphs, or `perf annotate`. In-process measurement is possible via the
`perf_event_open(2)` syscall, demonstrated in both C++ and Rust examples above.

**OpenOCD with JTAG** operates at a lower level than all the above tools,
accessing the CPU's CoreSight DAP directly. It is essential when Linux itself
will not boot, when debugging bare-metal early startup code, or when hardware
breakpoints (which do not require a running OS) are needed. OpenOCD exposes a
GDB RSP server on port 3333, so the same cross-GDB workflow applies. It can
coexist with gdbserver for simultaneous hardware and application debugging.

Together these tools form a layered debugging strategy:

| Layer | Tool | When to use |
|---|---|---|
| Hardware / bare metal | OpenOCD + JTAG | Boot failures, hardware faults, register inspection |
| Kernel / OS | perf, ftrace | Scheduler latency, IRQ storms, cache behavior |
| System calls | strace | Driver interactions, file I/O, IPC issues |
| Application logic | remote GDB | Logic bugs, memory corruption, race conditions |
| Performance | perf stat/record | Hotspots, IPC, branch mispredictions |

Effective embedded debugging combines all five layers, starting at the highest
level (strace, GDB) and moving toward hardware (perf, OpenOCD) as the root
cause demands it.

---

*End of Chapter 29 — Debugging Embedded Targets (GDB, strace, perf)*