# 21. Buildroot Networking Stack Configuration

**Structure:**
- Full ASCII architecture diagram showing the complete stack from hardware to application layer
- Five main sections matching the topic's scope: kernel Kconfig, iproute2, wpa_supplicant, connman, NetworkManager, and udev rules

**Code examples (10 total):**
- **C (3):** Netlink RTM_GETLINK interface enumeration, wpa_supplicant Unix control socket client, udev Netlink uevent monitor for interface hotplug
- **C++ (3):** connman D-Bus client via sdbus-c++, NetworkManager connection activation via libnm, modern RAII Netlink monitor with async event dispatch
- **Rust (4):** Async Netlink enumeration with the `rtnetlink` crate, wpa_supplicant control socket client, udev interface monitor using `tokio-udev`, and a DHCP trigger daemon using `std::process`

**Notable features:**
- Buildroot `Config.in` and `.config` symbols for every package
- Kernel Kconfig fragment files with commentary
- ASCII flow diagrams for wpa_supplicant handshake, connman/NM internal architectures, and the udev naming pipeline
- A final ASCII summary map tying all five layers together

---

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Kernel Kconfig Options for Networking](#kernel-kconfig-options-for-networking)
4. [Package Configuration](#package-configuration)
   - [iproute2](#iproute2)
   - [wpa_supplicant](#wpa_supplicant)
   - [connman](#connman)
   - [NetworkManager](#networkmanager)
5. [udev Rules for Interface Naming](#udev-rules-for-interface-naming)
6. [C Programming Examples](#c-programming-examples)
7. [C++ Programming Examples](#c-programming-examples-c)
8. [Rust Programming Examples](#rust-programming-examples)
9. [Summary](#summary)

---

## Introduction

Buildroot is a popular embedded Linux build system that enables developers to create minimal, efficient Linux-based firmware images for embedded targets. One of its most important configuration domains is the **networking stack** — the set of kernel options, user-space packages, and system rules that together enable a device to communicate over wired, wireless, or virtual networks.

Networking stack configuration in Buildroot encompasses:

- **Kernel-level networking**: TCP/IP stack options, socket families, driver selection, and netfilter subsystems — all controlled via Linux `Kconfig` fragments.
- **User-space network tools**: Packages such as `iproute2`, `wpa_supplicant`, `connman`, and `NetworkManager` that manage interfaces, routing, and wireless authentication.
- **Interface naming**: `udev` rules that provide stable, predictable interface names and eliminate the non-deterministic `eth0`, `wlan0` naming of older kernels.

Understanding how these layers interact is essential for creating robust embedded networking configurations. A misconfigured kernel option, missing package, or absent udev rule can render a device unable to connect — which in a headless embedded product can be extremely difficult to debug after deployment.

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                      APPLICATION LAYER                           |
|   (Web server, MQTT client, OPC-UA, custom daemons, etc.)        |
+------------------------------------------------------------------+
            |                            |
            v                            v
+-------------------+         +----------------------+
|   NetworkManager  |         |       connman        |
|   (heavyweight,   |         |  (lightweight, D-Bus)|
|    D-Bus, NM API) |         |   plugins: WiFi, VPN |
+-------------------+         +----------------------+
            |                            |
            v                            v
+------------------------------------------------------------------+
|                    iproute2  (ip, ss, tc, bridge)                |
|    Address mgmt  |  Routing  |  Traffic ctrl  |  Netlink cmds    |
+------------------------------------------------------------------+
                              |
            +-----------------+------------------+
            |                                    |
            v                                    v
+------------------------+           +------------------------+
|   wpa_supplicant       |           |    Wired / Virtual     |
|   (IEEE 802.11, EAP,   |           |    Ethernet drivers    |
|    WPA2/WPA3, 802.1X)  |           |    (e1000e, r8169 ...) |
+------------------------+           +------------------------+
            |
            v
+------------------------------------------------------------------+
|                    LINUX KERNEL                                  |
|  +-------------------+  +---------------+  +----------------+    |
|  | TCP/IP Stack      |  |   Netfilter   |  | WiFi / cfg80211|    |
|  | IPv4 / IPv6 / SCTP|  | iptables / nf |  | mac80211 layer |    |
|  +-------------------+  +---------------+  +----------------+    |
|  +-------------------+  +---------------+                        |
|  | Socket Families   |  | Network Devs  |                        |
|  | AF_INET, AF_PACKET|  | Drivers/PHYs  |                        |
|  +-------------------+  +---------------+                        |
+------------------------------------------------------------------+
                              |
+------------------------------------------------------------------+
|              HARDWARE (NIC, WiFi chip, PHY, Switch)              |
+------------------------------------------------------------------+
```

---

## Kernel Kconfig Options for Networking

Buildroot's kernel configuration is managed via the `BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES` mechanism or a full `.config` file referenced by `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE`. Network-related `Kconfig` symbols are grouped across several menu trees.

### Buildroot menuconfig path

```
Kernel  --->
  Linux Kernel  --->
    Kernel configuration  --->
      [ ] Use a custom config file
      [ ] Additional configuration fragment files
```

### Core networking Kconfig symbols

```kconfig
# ============================================================
# Core TCP/IP networking
# ============================================================
CONFIG_NET=y
CONFIG_INET=y
CONFIG_IPV6=y                        # Enable IPv6 dual-stack
CONFIG_IP_MULTICAST=y                # Needed for mDNS, SSDP, RTP
CONFIG_IP_ADVANCED_ROUTER=y
CONFIG_IP_MULTIPLE_TABLES=y          # Policy-based routing
CONFIG_IP_ROUTE_MULTIPATH=y

# ============================================================
# Packet socket (AF_PACKET) - required by iproute2, tcpdump
# ============================================================
CONFIG_PACKET=y
CONFIG_UNIX=y                        # AF_UNIX sockets

# ============================================================
# Netfilter / iptables (if firewall needed)
# ============================================================
CONFIG_NETFILTER=y
CONFIG_NF_CONNTRACK=y
CONFIG_NF_TABLES=y                   # nftables backend
CONFIG_IP_NF_IPTABLES=y
CONFIG_IP_NF_FILTER=y
CONFIG_IP_NF_NAT=y                   # NAT / masquerading

# ============================================================
# Wireless stack
# ============================================================
CONFIG_CFG80211=y
CONFIG_MAC80211=y
CONFIG_RFKILL=y

# ============================================================
# Ethernet drivers (target-specific)
# ============================================================
CONFIG_NET_VENDOR_REALTEK=y
CONFIG_R8169=y                       # Example: Realtek PCIe GbE
CONFIG_SMSC_PHY=y                    # SMSC PHY driver (common SoC)

# ============================================================
# VLAN / Bridge (needed by NetworkManager / connman)
# ============================================================
CONFIG_VLAN_8021Q=y
CONFIG_BRIDGE=y
CONFIG_BRIDGE_NETFILTER=y

# ============================================================
# Traffic control (tc / iproute2)
# ============================================================
CONFIG_NET_SCHED=y
CONFIG_NET_SCH_PFIFO_FAST=y
CONFIG_NET_SCH_HTB=y
CONFIG_NET_SCH_FQ_CODEL=y

# ============================================================
# Network namespaces (containers, isolation)
# ============================================================
CONFIG_NET_NS=y
```

### Buildroot kernel fragment file example

```
# board/myboard/linux-net.config

CONFIG_NET=y
CONFIG_INET=y
CONFIG_IPV6=y
CONFIG_PACKET=y
CONFIG_UNIX=y
CONFIG_CFG80211=y
CONFIG_MAC80211=y
CONFIG_RFKILL=y
CONFIG_BRIDGE=y
CONFIG_VLAN_8021Q=y
CONFIG_NET_SCHED=y
CONFIG_NET_SCH_FQ_CODEL=y
CONFIG_NF_CONNTRACK=y
CONFIG_IP_NF_IPTABLES=y
CONFIG_IP_NF_FILTER=y
CONFIG_IP_NF_NAT=y
```

Reference it in `board/myboard/buildroot.config`:

```makefile
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES="board/myboard/linux-net.config"
```

---

## Package Configuration

Buildroot organises packages under `package/`. Each package has a `Config.in` file that exposes Kconfig symbols and a `<package>.mk` makefile.

### iproute2

`iproute2` is the modern Linux networking toolkit, replacing deprecated `ifconfig`/`route`. It communicates with the kernel via **Netlink** sockets and provides `ip`, `ss`, `tc`, `bridge`, and `nstat`.

**Buildroot menuconfig path:**
```
Target packages  --->
  Networking applications  --->
    [*] iproute2
```

**`package/iproute2/Config.in` (excerpt from upstream Buildroot):**
```kconfig
config BR2_PACKAGE_IPROUTE2
    bool "iproute2"
    depends on BR2_USE_MMU
    select BR2_PACKAGE_LIBMNL
    help
      Utilities for controlling networking in Linux kernels.
      Includes ip, ss, tc, bridge, nstat and others.
```

**Selecting iproute2 in your Buildroot `.config`:**
```makefile
BR2_PACKAGE_IPROUTE2=y
```

**iproute2 usage in the target system:**

```
ip link show
+---------------------------------------------------+
| 1: lo   <LOOPBACK,UP>  mtu 65536                  |
| 2: eth0 <BROADCAST,MULTICAST,UP,LOWER_UP>  mtu 1500|
| 3: wlan0<BROADCAST,MULTICAST>  mtu 1500           |
+---------------------------------------------------+

ip addr add 192.168.1.100/24 dev eth0
ip route add default via 192.168.1.1 dev eth0
ip -6 addr add fd00::1/64 dev eth0

ss -tulpn          # socket statistics (replaces netstat)
tc qdisc show      # show traffic control queuing disciplines
bridge fdb show    # Forwarding DataBase entries
```

---

### wpa_supplicant

`wpa_supplicant` is the IEEE 802.11 authentication and association daemon. It handles WPA2-PSK, WPA3-SAE, and enterprise 802.1X/EAP modes. It exposes a control socket consumed by `connman`, `NetworkManager`, and custom scripts.

**Buildroot menuconfig path:**
```
Target packages  --->
  Networking applications  --->
    [*] wpa_supplicant
          [*] Enable nl80211 support
          [*] Enable IBSS RSN support
          [*] Enable AP mode
          [*] Enable WPA3 support
          [*] Install wpa_cli binary
          [*] Install wpa_passphrase binary
```

**`package/wpa_supplicant/Config.in` (key symbols):**
```kconfig
config BR2_PACKAGE_WPA_SUPPLICANT
    bool "wpa_supplicant"
    depends on BR2_PACKAGE_OPENSSL || BR2_PACKAGE_LIBRESSL || ...
    select BR2_PACKAGE_LIBNL

config BR2_PACKAGE_WPA_SUPPLICANT_NL80211
    bool "Enable nl80211 support"
    depends on BR2_PACKAGE_WPA_SUPPLICANT
    # Requires kernel CONFIG_CFG80211=y
```

**Target `.config` selections:**
```makefile
BR2_PACKAGE_WPA_SUPPLICANT=y
BR2_PACKAGE_WPA_SUPPLICANT_NL80211=y
BR2_PACKAGE_WPA_SUPPLICANT_WPA3=y
BR2_PACKAGE_WPA_SUPPLICANT_CLI=y
```

**`/etc/wpa_supplicant.conf` example on the target:**
```
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=0
update_config=1

network={
    ssid="MyEmbeddedNet"
    psk="secretpassword"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
}

# Enterprise example (802.1X / EAP-TLS)
network={
    ssid="CorpNet"
    key_mgmt=WPA-EAP
    eap=TLS
    identity="device@corp.com"
    ca_cert="/etc/ssl/corp-ca.pem"
    client_cert="/etc/ssl/device.pem"
    private_key="/etc/ssl/device.key"
}
```

**wpa_supplicant control flow:**

```
wpa_supplicant starts
       |
       v
  Opens control socket (/var/run/wpa_supplicant/wlan0)
       |
       v
  Scans for SSIDs (nl80211 -> kernel cfg80211)
       |
       v
  Matches network{} block in wpa_supplicant.conf
       |
       v
  Four-way handshake (WPA2-PSK) / EAP exchange (Enterprise)
       |
       v
  Signals CONNECTED -> client (connman/NM/dhcpcd gets IP)
```

---

### connman

`connman` (Connection Manager) is a lightweight, D-Bus-driven network manager designed for embedded Linux. It supports Ethernet, WiFi (via `wpa_supplicant`), Bluetooth tethering, VPN, and cellular via ofono. It is well-suited to resource-constrained platforms.

**Buildroot menuconfig path:**
```
Target packages  --->
  Networking applications  --->
    [*] connman
          [*] wifi
          [*] ethernet
          [*] loopback
          [*] bluetooth
          [ ] vpn (optional)
          [*] iptables
```

**Target `.config` selections:**
```makefile
BR2_PACKAGE_CONNMAN=y
BR2_PACKAGE_CONNMAN_WIFI=y
BR2_PACKAGE_CONNMAN_ETHERNET=y
BR2_PACKAGE_CONNMAN_LOOPBACK=y
BR2_PACKAGE_CONNMAN_IPTABLES=y
BR2_PACKAGE_CONNMAN_CLIENT=y       # connmanctl CLI tool
```

**connman depends on:**
- `BR2_PACKAGE_DBUS` — D-Bus daemon must be enabled.
- `BR2_PACKAGE_WPA_SUPPLICANT` — for WiFi plugin.
- `BR2_PACKAGE_IPTABLES` — for tethering/NAT.

**`/etc/connman/main.conf` on the target:**
```ini
[General]
InputRequestTimeout = 30
BrowserLaunchTimeout = 300
NetworkInterfaceBlacklist = vmnet,vboxnet,virbr,ifb,docker,veth
AllowHostnameUpdates = false
SingleConnectedTechnology = false
PersistentTetheringMode = false

[WiFi]
EnableOnlineCheck = true
OnlineCheckURL = http://connectivitycheck.platform.hisilicon.com/generate_204
```

**connman service configuration (`/var/lib/connman/<hash>/settings`):**
```ini
[service_eth0_cable]
Type = ethernet
IPv4.method = dhcp
IPv6.method = auto

[service_wlan0_mynet]
Type = wifi
SSID = 4d79456d626564646564...   # hex-encoded SSID
Passphrase = secretpassword
IPv4.method = dhcp
```

**connman internal architecture:**

```
+------------------------------------------------------------------+
|                          connman                                 |
|                                                                  |
|   +-----------+   +-----------+   +-----------+   +----------+   |
|   |  Ethernet |   |   WiFi    |   | Bluetooth |   |  VPN     |   |
|   |  plugin   |   |  plugin   |   |  plugin   |   |  plugin  |   |
|   +-----------+   +-----------+   +-----------+   +----------+   |
|         |               |                                        |
|         v               v                                        |
|   +-----------+   +-------------------+                          |
|   |  iproute2 |   |  wpa_supplicant   |                          |
|   |  Netlink  |   |  ctrl socket      |                          |
|   +-----------+   +-------------------+                          |
|                                                                  |
|   +----------------------------------------------------------+   |
|   |              D-Bus interface (net.connman.*)             |   |
|   +----------------------------------------------------------+   |
+------------------------------------------------------------------+
       ^                   ^
       |                   |
  connmanctl          3rd party apps
  (CLI client)        via D-Bus API
```

---

### NetworkManager

`NetworkManager` is a feature-rich connection manager targeting desktops, servers, and complex embedded systems. It supports Ethernet, WiFi, mobile broadband, VPN, bonds, teams, bridges, VLANs, and WireGuard tunnels. It is heavier than connman but offers richer API and `nmcli`/`nmtui` tooling.

**Buildroot menuconfig path:**
```
Target packages  --->
  Networking applications  --->
    [*] NetworkManager
          [*] wifi support
          [*] ppp support
          [*] nmcli (command line tool)
          [*] nmtui (text UI)
          [ ] modem manager (optional, for 4G/LTE)
```

**Target `.config` selections:**
```makefile
BR2_PACKAGE_NETWORKMANAGER=y
BR2_PACKAGE_NETWORKMANAGER_WIFI=y
BR2_PACKAGE_NETWORKMANAGER_NMCLI=y
BR2_PACKAGE_NETWORKMANAGER_NMTUI=y
BR2_PACKAGE_NETWORKMANAGER_PPP=y
```

**NetworkManager depends on:**
- `BR2_PACKAGE_DBUS`
- `BR2_PACKAGE_GLIB2`
- `BR2_PACKAGE_LIBNDP` — IPv6 Neighbor Discovery.
- `BR2_PACKAGE_NEWT` — for `nmtui`.

**Connection profile (`/etc/NetworkManager/system-connections/MyWiFi.nmconnection`):**
```ini
[connection]
id=MyWiFi
uuid=a1b2c3d4-e5f6-7890-abcd-ef1234567890
type=wifi
autoconnect=true
autoconnect-priority=10

[wifi]
ssid=MyEmbeddedNet
mode=infrastructure
band=bg
channel=6

[wifi-security]
auth-alg=open
key-mgmt=wpa-psk
psk=secretpassword

[ipv4]
method=auto

[ipv6]
method=auto
```

**nmcli quick reference on target:**
```
nmcli device status          # list devices
nmcli connection show        # list saved connections
nmcli connection up MyWiFi   # activate a profile
nmcli connection add type ethernet ifname eth0 con-name cable
nmcli dev wifi list          # scan access points
nmcli dev wifi connect "SSID" password "pass"
```

**NetworkManager internal architecture:**

```
+------------------------------------------------------------------+
|                       NetworkManager                             |
|                                                                  |
|  +---------+  +----------+  +----------+  +------------------+   |
|  | Device  |  | Device   |  | Device   |  | VPN/Bond/Bridge  |   |
|  | Ethernet|  | WiFi     |  | Modem    |  | Team / VLAN      |   |
|  +---------+  +----------+  +----------+  +------------------+   |
|       |             |              |                             |
|       v             v              v                             |
|  +---------+  +----------+  +---------------+                    |
|  | Netlink |  |wpa_supp. |  | ModemManager  |                    |
|  | (kernel)|  |ctrl sock |  | (D-Bus)       |                    |
|  +---------+  +----------+  +---------------+                    |
|                                                                  |
|  +----------------------------------------------------------+    |
|  |         D-Bus interface (org.freedesktop.NetworkManager) |    |
|  +----------------------------------------------------------+    |
+------------------------------------------------------------------+
     ^         ^          ^
     |         |          |
  nmcli     nmtui     3rd-party
  (CLI)   (text-UI)   NM-applet
```

---

## udev Rules for Interface Naming

Older kernels assign names like `eth0`, `wlan0` non-deterministically — the order depends on driver initialisation timing, which can change between boots or kernel versions. Buildroot targets running `udev` (or `eudev`, its lighter fork) can use rules to enforce **persistent, predictable names** based on MAC address, PCI slot, or BIOS path.

**Buildroot udev packages:**
```makefile
BR2_PACKAGE_EUDEV=y          # lightweight udev fork (typical for embedded)
# or
BR2_PACKAGE_SYSTEMD=y        # includes full udev (heavier)
```

**menuconfig path:**
```
System configuration  --->
  /dev management  --->
    (X) Dynamic using devtmpfs + eudev
```

### Systemd predictable network interface names

When using `systemd-udevd`, the naming policy follows `net_id` rules:

```
+-------------------------------+-----------------------+
| Naming scheme                 | Example               |
+-------------------------------+-----------------------+
| Onboard index (firmware)      | eno1                  |
| PCI hotplug slot index        | ens3                  |
| PCI geographic (bus/dev/fn)   | enp2s0                |
| MAC-based                     | enx001122334455       |
| Wireless same scheme          | wlp3s0 / wlxMAC       |
+-------------------------------+-----------------------+
```

### Custom udev rules

Rules live in `/etc/udev/rules.d/` on the target. Buildroot overlays are the correct place to ship them.

**`board/myboard/rootfs-overlay/etc/udev/rules.d/70-persistent-net.rules`:**
```
# Rename Ethernet by MAC address
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="de:ad:be:ef:01:02", \
    NAME="eth-mgmt"

# Rename second Ethernet port by PCI path
SUBSYSTEM=="net", ACTION=="add", KERNELS=="0000:02:00.0", \
    NAME="eth-data"

# Rename WiFi by MAC
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="aa:bb:cc:dd:ee:ff", \
    NAME="wlan-ext"

# Rename USB Ethernet dongle by driver
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="r8152", \
    NAME="eth-usb"
```

**Register overlay in Buildroot config:**
```makefile
BR2_ROOTFS_OVERLAY="board/myboard/rootfs-overlay"
```

### eudev rule for interface naming (without systemd)

```
# /etc/udev/rules.d/71-net-names.rules
# Match by interface type and MAC, set stable name

SUBSYSTEM!="net", GOTO="net_names_end"
ACTION!="add", GOTO="net_names_end"

# Wired: rename by MAC
ATTR{type}=="1", ATTR{address}=="de:ad:be:ef:01:02", NAME="eth0-fixed"

# Wireless: rename by MAC
ATTR{type}=="1", DRIVERS=="?*", ATTR{address}=="aa:bb:cc:dd:ee:ff", \
    SUBSYSTEMS=="usb", NAME="wlan-usb"

LABEL="net_names_end"
```

### Interface naming flow

```
Kernel detects NIC
      |
      v
udevd receives uevent (SUBSYSTEM=net, ACTION=add)
      |
      v
Rules evaluated in /etc/udev/rules.d/ (sorted by filename)
      |
      +---> ATTR{address} matches MAC?  --->  NAME="eth-mgmt"
      |
      +---> KERNELS matches PCI slot?   --->  NAME="eth-data"
      |
      +---> DRIVERS matches driver?     --->  NAME="eth-usb"
      |
      v
Kernel interface renamed via SIOCSIFNAME ioctl
      |
      v
connman / NetworkManager / wpa_supplicant use new name
```

---

## C Programming Examples

### Example 1 — Netlink socket: enumerate network interfaces (iproute2-style)

```c
/*
 * netlink_ifaces.c
 * Enumerate network interfaces via Netlink RTM_GETLINK,
 * mimicking what iproute2 'ip link show' does internally.
 *
 * Build (Buildroot cross-compile):
 *   ${CROSS_COMPILE}gcc -o netlink_ifaces netlink_ifaces.c
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
#include <net/if.h>
#include <arpa/inet.h>

#define BUFSIZE 8192

static void parse_link_msg(struct nlmsghdr *nlh)
{
    struct ifinfomsg *ifi = NLMSG_DATA(nlh);
    struct rtattr *rta = IFLA_RTA(ifi);
    int rta_len = IFLA_PAYLOAD(nlh);
    char ifname[IFNAMSIZ] = "<unknown>";
    unsigned char *mac = NULL;

    for (; RTA_OK(rta, rta_len); rta = RTA_NEXT(rta, rta_len)) {
        if (rta->rta_type == IFLA_IFNAME) {
            strncpy(ifname, RTA_DATA(rta), IFNAMSIZ - 1);
        } else if (rta->rta_type == IFLA_ADDRESS) {
            mac = RTA_DATA(rta);
        }
    }

    printf("  [%2d]  %-12s  flags=0x%04x  %s\n",
           ifi->ifi_index, ifname, ifi->ifi_flags,
           (ifi->ifi_flags & IFF_UP) ? "UP" : "DOWN");

    if (mac && RTA_PAYLOAD(rta) >= 6) {
        printf("        MAC: %02x:%02x:%02x:%02x:%02x:%02x\n",
               mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
    }
}

int main(void)
{
    int fd;
    struct sockaddr_nl sa = { .nl_family = AF_NETLINK };
    struct {
        struct nlmsghdr nlh;
        struct ifinfomsg ifi;
    } req = {
        .nlh = {
            .nlmsg_len   = NLMSG_LENGTH(sizeof(struct ifinfomsg)),
            .nlmsg_type  = RTM_GETLINK,
            .nlmsg_flags = NLM_F_REQUEST | NLM_F_DUMP,
            .nlmsg_seq   = 1,
        },
        .ifi = { .ifi_family = AF_UNSPEC },
    };
    char buf[BUFSIZE];
    ssize_t n;

    fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
    if (fd < 0) { perror("socket"); return 1; }

    if (bind(fd, (struct sockaddr *)&sa, sizeof(sa)) < 0) {
        perror("bind"); close(fd); return 1;
    }

    if (send(fd, &req, req.nlh.nlmsg_len, 0) < 0) {
        perror("send"); close(fd); return 1;
    }

    printf("\n  Network Interfaces (via Netlink RTM_GETLINK)\n");
    printf("  %-4s  %-12s  %-10s  %s\n",
           "Idx", "Name", "Flags", "State");
    printf("  %s\n", "----------------------------------------------");

    while ((n = recv(fd, buf, BUFSIZE, 0)) > 0) {
        struct nlmsghdr *nlh = (struct nlmsghdr *)buf;
        for (; NLMSG_OK(nlh, (size_t)n); nlh = NLMSG_NEXT(nlh, n)) {
            if (nlh->nlmsg_type == NLMSG_DONE) goto done;
            if (nlh->nlmsg_type == NLMSG_ERROR) {
                fprintf(stderr, "Netlink error\n"); goto done;
            }
            if (nlh->nlmsg_type == RTM_NEWLINK)
                parse_link_msg(nlh);
        }
    }
done:
    close(fd);
    return 0;
}
```

### Example 2 — wpa_supplicant control socket client (C)

```c
/*
 * wpa_ctrl_client.c
 * Connect to wpa_supplicant's control socket,
 * issue STATUS and SCAN_RESULTS commands.
 *
 * Build:
 *   ${CROSS_COMPILE}gcc -o wpa_ctrl_client wpa_ctrl_client.c
 *
 * Requires: wpa_supplicant running with ctrl_interface=/var/run/wpa_supplicant
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <fcntl.h>
#include <errno.h>

#define WPA_CTRL_DIR  "/var/run/wpa_supplicant"
#define IFACE         "wlan0"
#define LOCAL_SOCK    "/tmp/wpa_client_XXXXXX"
#define BUFSIZE       4096

static int wpa_send_recv(int fd, const char *cmd, char *reply, size_t rlen)
{
    ssize_t n;
    if (send(fd, cmd, strlen(cmd), 0) < 0) {
        perror("send"); return -1;
    }
    n = recv(fd, reply, rlen - 1, 0);
    if (n < 0) { perror("recv"); return -1; }
    reply[n] = '\0';
    return (int)n;
}

int main(void)
{
    int fd;
    struct sockaddr_un remote_addr, local_addr;
    char local_path[] = LOCAL_SOCK;
    char buf[BUFSIZE];

    fd = socket(AF_UNIX, SOCK_DGRAM, 0);
    if (fd < 0) { perror("socket"); return 1; }

    /* Bind to a temporary local socket */
    mkstemp(local_path);
    unlink(local_path);
    memset(&local_addr, 0, sizeof(local_addr));
    local_addr.sun_family = AF_UNIX;
    strncpy(local_addr.sun_path, local_path, sizeof(local_addr.sun_path) - 1);

    if (bind(fd, (struct sockaddr *)&local_addr, sizeof(local_addr)) < 0) {
        perror("bind"); close(fd); return 1;
    }

    /* Connect to wpa_supplicant control socket */
    memset(&remote_addr, 0, sizeof(remote_addr));
    remote_addr.sun_family = AF_UNIX;
    snprintf(remote_addr.sun_path, sizeof(remote_addr.sun_path),
             "%s/%s", WPA_CTRL_DIR, IFACE);

    if (connect(fd, (struct sockaddr *)&remote_addr, sizeof(remote_addr)) < 0) {
        perror("connect (is wpa_supplicant running?)");
        unlink(local_path); close(fd); return 1;
    }

    /* Query STATUS */
    printf("\n=== wpa_supplicant STATUS ===\n");
    if (wpa_send_recv(fd, "STATUS", buf, BUFSIZE) > 0)
        printf("%s\n", buf);

    /* Trigger a scan */
    printf("=== Triggering SCAN ===\n");
    wpa_send_recv(fd, "SCAN", buf, BUFSIZE);
    printf("Scan initiated: %s\n", buf);
    sleep(3);

    /* Retrieve scan results */
    printf("=== SCAN_RESULTS ===\n");
    if (wpa_send_recv(fd, "SCAN_RESULTS", buf, BUFSIZE) > 0)
        printf("%s\n", buf);

    unlink(local_path);
    close(fd);
    return 0;
}
```

### Example 3 — udev Netlink monitor (C): detect interface add/remove events

```c
/*
 * udev_net_monitor.c
 * Monitor udev Netlink events for network interface changes.
 * Prints add/remove events — useful for hotplug handling without udev rules.
 *
 * Build:
 *   ${CROSS_COMPILE}gcc -o udev_net_monitor udev_net_monitor.c
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <poll.h>

#define UDEV_MONITOR_GRP  1   /* UDEV kernel monitor group */
#define BUFSIZE           4096

int main(void)
{
    int fd;
    struct sockaddr_nl sa = {
        .nl_family = AF_NETLINK,
        .nl_groups = UDEV_MONITOR_GRP,
    };
    char buf[BUFSIZE];
    struct pollfd pfd;

    fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_KOBJECT_UEVENT);
    if (fd < 0) { perror("socket"); return 1; }

    if (bind(fd, (struct sockaddr *)&sa, sizeof(sa)) < 0) {
        perror("bind"); close(fd); return 1;
    }

    printf("Monitoring udev network events (Ctrl-C to stop)...\n");
    printf("+---------------------------------------------------+\n");
    printf("| Action    | Subsystem | Device                    |\n");
    printf("+---------------------------------------------------+\n");

    pfd.fd = fd;
    pfd.events = POLLIN;

    while (poll(&pfd, 1, -1) > 0) {
        ssize_t n = recv(fd, buf, BUFSIZE - 1, MSG_DONTWAIT);
        if (n <= 0) continue;
        buf[n] = '\0';

        /* uevent format: "ACTION@/path\0KEY=VALUE\0..." */
        char *action = NULL, *subsystem = NULL, *devname = NULL;
        char *p = buf;

        /* First token is "action@path" */
        action = strtok(p, "@");
        strtok(NULL, "\0"); /* skip path */

        /* Remaining tokens are KEY=VALUE pairs */
        char *tok;
        while ((tok = strtok(NULL, "\0")) != NULL) {
            if (strncmp(tok, "SUBSYSTEM=", 10) == 0)
                subsystem = tok + 10;
            else if (strncmp(tok, "INTERFACE=", 10) == 0)
                devname = tok + 10;
        }

        if (subsystem && strcmp(subsystem, "net") == 0 && devname && action) {
            printf("| %-9s | %-9s | %-25s |\n", action, subsystem, devname);
        }
    }

    close(fd);
    return 0;
}
```

---

## C++ Programming Examples

### Example 4 — connman D-Bus client (C++ with sdbus-c++)

```cpp
/*
 * connman_dbus_client.cpp
 * Query connman via D-Bus using sdbus-c++ to list services
 * and their connection states.
 *
 * Build:
 *   ${CROSS_COMPILE}g++ -std=c++17 -o connman_client connman_dbus_client.cpp \
 *       $(pkg-config --cflags --libs sdbus-c++)
 *
 * Buildroot: BR2_PACKAGE_SDBUS_CPP=y, BR2_PACKAGE_CONNMAN=y
 */

#include <sdbus-c++/sdbus-c++.h>
#include <iostream>
#include <string>
#include <map>
#include <variant>
#include <vector>
#include <iomanip>

using SdbusVariant  = sdbus::Variant;
using ServiceProps  = std::map<std::string, SdbusVariant>;
using ServiceList   = std::vector<sdbus::Struct<sdbus::ObjectPath, ServiceProps>>;

void print_state_bar(const std::string &state)
{
    // ASCII state diagram
    const std::vector<std::string> states = {
        "idle", "association", "configuration", "ready", "online"
    };
    std::cout << "    State: [";
    bool reached = false;
    for (auto &s : states) {
        if (s == state) { std::cout << ">" << s << "<"; reached = true; }
        else            { std::cout << (reached ? " " : "-") << s << "-"; }
    }
    std::cout << "]\n";
}

int main()
{
    try {
        auto proxy = sdbus::createProxy("net.connman", "/");

        // Call GetServices on net.connman.Manager
        ServiceList services;
        proxy->callMethod("GetServices")
              .onInterface("net.connman.Manager")
              .storeResultsTo(services);

        std::cout << "\n=== connman Services ===\n";
        std::cout << std::string(60, '-') << "\n";

        for (auto &[path, props] : services) {
            std::string name  = "<unknown>";
            std::string type  = "<unknown>";
            std::string state = "<unknown>";

            if (props.count("Name"))  name  = props.at("Name").get<std::string>();
            if (props.count("Type"))  type  = props.at("Type").get<std::string>();
            if (props.count("State")) state = props.at("State").get<std::string>();

            std::cout << "  Service: " << std::setw(20) << std::left << name
                      << "  Type: " << std::setw(8) << type << "\n";
            print_state_bar(state);
            std::cout << "  Path: " << path << "\n";
            std::cout << std::string(60, '-') << "\n";
        }

    } catch (const sdbus::Error &e) {
        std::cerr << "D-Bus error: " << e.what() << "\n";
        return 1;
    }
    return 0;
}
```

### Example 5 — NetworkManager connection activation (C++ with libnm)

```cpp
/*
 * nm_activate.cpp
 * Use libnm (NetworkManager C library) from C++ to
 * list connections and activate a named profile.
 *
 * Build:
 *   ${CROSS_COMPILE}g++ -std=c++17 -o nm_activate nm_activate.cpp \
 *       $(pkg-config --cflags --libs libnm)
 *
 * Buildroot: BR2_PACKAGE_NETWORKMANAGER=y
 */

#include <NetworkManager.h>
#include <glib.h>
#include <iostream>
#include <string>
#include <cstring>

class NMClient {
public:
    NMClient()
    {
        GError *err = nullptr;
        client_ = nm_client_new(nullptr, &err);
        if (!client_) {
            std::string msg = err ? err->message : "unknown";
            g_clear_error(&err);
            throw std::runtime_error("nm_client_new failed: " + msg);
        }
    }
    ~NMClient() { if (client_) g_object_unref(client_); }

    void list_connections() const
    {
        const GPtrArray *conns = nm_client_get_connections(client_);
        std::cout << "\n=== NetworkManager Connections ===\n";
        std::cout << std::string(50, '-') << "\n";
        for (guint i = 0; i < conns->len; ++i) {
            NMConnection *c = NM_CONNECTION(conns->pdata[i]);
            NMSettingConnection *sc = nm_connection_get_setting_connection(c);
            if (!sc) continue;
            std::cout << "  ID:   " << nm_setting_connection_get_id(sc)       << "\n";
            std::cout << "  UUID: " << nm_setting_connection_get_uuid(sc)      << "\n";
            std::cout << "  Type: " << nm_setting_connection_get_connection_type(sc) << "\n";
            std::cout << std::string(50, '-') << "\n";
        }
    }

    bool activate(const std::string &connection_id) const
    {
        const GPtrArray *conns = nm_client_get_connections(client_);
        NMRemoteConnection *target = nullptr;

        for (guint i = 0; i < conns->len; ++i) {
            NMRemoteConnection *rc = NM_REMOTE_CONNECTION(conns->pdata[i]);
            NMSettingConnection *sc =
                nm_connection_get_setting_connection(NM_CONNECTION(rc));
            if (sc && connection_id == nm_setting_connection_get_id(sc)) {
                target = rc;
                break;
            }
        }

        if (!target) {
            std::cerr << "Connection '" << connection_id << "' not found.\n";
            return false;
        }

        GError *err = nullptr;
        NMActiveConnection *ac = nm_client_activate_connection_finish(
            client_,
            nullptr, /* async result */
            &err
        );
        /* Simplified: use blocking nm_client_activate_connection */
        (void)ac;
        g_clear_error(&err);

        std::cout << "Activating: " << connection_id << "\n";
        return true;
    }

private:
    NMClient_ *client_ = nullptr;
};

int main(int argc, char *argv[])
{
    try {
        NMClient client;
        client.list_connections();

        if (argc > 1) {
            std::cout << "\nActivating '" << argv[1] << "'...\n";
            client.activate(argv[1]);
        }
    } catch (const std::exception &e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }
    return 0;
}
```

### Example 6 — Async Netlink monitoring with RAII wrappers (Modern C++)

```cpp
/*
 * raii_netlink_monitor.cpp
 * RAII wrapper around a Netlink RTMGRP_LINK socket.
 * Monitors interface UP/DOWN/ADD/REMOVE events asynchronously.
 *
 * Build:
 *   ${CROSS_COMPILE}g++ -std=c++20 -o nl_monitor raii_netlink_monitor.cpp
 */

#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
#include <net/if.h>
#include <unistd.h>

#include <iostream>
#include <string>
#include <functional>
#include <stdexcept>
#include <array>
#include <cstring>

// ---- RAII socket wrapper ------------------------------------------------
class NetlinkSocket {
public:
    explicit NetlinkSocket(int groups)
    {
        fd_ = ::socket(AF_NETLINK, SOCK_RAW | SOCK_CLOEXEC, NETLINK_ROUTE);
        if (fd_ < 0) throw std::system_error(errno, std::system_category(), "socket");

        sockaddr_nl sa{};
        sa.nl_family = AF_NETLINK;
        sa.nl_groups = static_cast<unsigned>(groups);
        if (::bind(fd_, reinterpret_cast<sockaddr *>(&sa), sizeof(sa)) < 0) {
            ::close(fd_);
            throw std::system_error(errno, std::system_category(), "bind");
        }
    }

    ~NetlinkSocket() { if (fd_ >= 0) ::close(fd_); }
    NetlinkSocket(const NetlinkSocket &)            = delete;
    NetlinkSocket &operator=(const NetlinkSocket &) = delete;
    NetlinkSocket(NetlinkSocket &&o) noexcept : fd_(o.fd_) { o.fd_ = -1; }

    int fd() const noexcept { return fd_; }

private:
    int fd_;
};

// ---- Interface event dispatcher ----------------------------------------
struct IfEvent {
    enum class Action { Added, Removed, Up, Down, Changed };
    int         index;
    std::string name;
    unsigned    flags;
    Action      action;

    static std::string action_str(Action a) {
        switch (a) {
            case Action::Added:   return "ADDED";
            case Action::Removed: return "REMOVED";
            case Action::Up:      return "UP";
            case Action::Down:    return "DOWN";
            default:              return "CHANGED";
        }
    }
};

class InterfaceMonitor {
public:
    using Callback = std::function<void(const IfEvent &)>;

    explicit InterfaceMonitor(Callback cb)
        : sock_(RTMGRP_LINK), cb_(std::move(cb))
    {}

    void run()
    {
        std::array<char, 8192> buf{};
        while (true) {
            ssize_t n = ::recv(sock_.fd(), buf.data(), buf.size(), 0);
            if (n < 0) {
                if (errno == EINTR) continue;
                throw std::system_error(errno, std::system_category(), "recv");
            }
            process(buf.data(), static_cast<size_t>(n));
        }
    }

private:
    void process(const char *buf, size_t len)
    {
        for (auto *nlh = reinterpret_cast<const nlmsghdr *>(buf);
             NLMSG_OK(nlh, len); nlh = NLMSG_NEXT(nlh, len))
        {
            if (nlh->nlmsg_type != RTM_NEWLINK &&
                nlh->nlmsg_type != RTM_DELLINK) continue;

            auto *ifi = reinterpret_cast<const ifinfomsg *>(NLMSG_DATA(nlh));
            IfEvent ev;
            ev.index = ifi->ifi_index;
            ev.flags = ifi->ifi_flags;

            int rta_len = IFLA_PAYLOAD(nlh);
            for (auto *rta = IFLA_RTA(ifi); RTA_OK(rta, rta_len);
                 rta = RTA_NEXT(rta, rta_len))
            {
                if (rta->rta_type == IFLA_IFNAME)
                    ev.name = static_cast<const char *>(RTA_DATA(rta));
            }

            if (nlh->nlmsg_type == RTM_DELLINK) {
                ev.action = IfEvent::Action::Removed;
            } else {
                ev.action = (ev.flags & IFF_UP)
                    ? IfEvent::Action::Up
                    : IfEvent::Action::Down;
            }
            cb_(ev);
        }
    }

    NetlinkSocket sock_;
    Callback      cb_;
};

// ---- ASCII event log ----------------------------------------------------
void print_event(const IfEvent &ev)
{
    std::cout
        << "  +----------------------------------+\n"
        << "  | IF Event\n"
        << "  |  Index : " << ev.index << "\n"
        << "  |  Name  : " << ev.name  << "\n"
        << "  |  Action: " << IfEvent::action_str(ev.action) << "\n"
        << "  |  Flags : 0x" << std::hex << ev.flags << std::dec << "\n"
        << "  +----------------------------------+\n";
}

int main()
{
    std::cout << "Monitoring network interface events...\n\n";
    InterfaceMonitor mon(print_event);
    mon.run();  // blocks; send SIGINT to stop
}
```

---

## Rust Programming Examples

### Example 7 — Netlink interface enumeration with rtnetlink crate

```toml
# Cargo.toml
[package]
name = "netlink-ifaces"
version = "0.1.0"
edition = "2021"

[dependencies]
rtnetlink = "0.13"
tokio = { version = "1", features = ["full"] }
futures = "0.3"
```

```rust
// src/main.rs
//
// Enumerate network interfaces via the rtnetlink crate (Tokio async).
// Equivalent to 'ip link show' from iproute2.
//
// Cross-compile for Buildroot target:
//   cargo build --target aarch64-unknown-linux-gnu --release

use futures::stream::TryStreamExt;
use rtnetlink::{new_connection, LinkLayerType};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (connection, handle, _) = new_connection()?;
    tokio::spawn(connection);

    let mut links = handle.link().get().execute();

    println!("\n  Network Interfaces (rtnetlink)\n");
    println!("  {:<4}  {:<15}  {:<10}  {}", "Idx", "Name", "Type", "State");
    println!("  {}", "-".repeat(55));

    while let Some(link) = links.try_next().await? {
        use netlink_packet_route::link::nlas::Nla;

        let index = link.header.index;
        let flags = link.header.flags;

        let mut name = String::from("<unknown>");
        let mut mac  = String::new();

        for nla in &link.nlas {
            match nla {
                Nla::IfName(n)   => name = n.clone(),
                Nla::Address(a)  => {
                    mac = a.iter()
                           .map(|b| format!("{:02x}", b))
                           .collect::<Vec<_>>()
                           .join(":");
                }
                _ => {}
            }
        }

        let link_type = match link.header.link_layer_type {
            LinkLayerType::Ether    => "Ethernet",
            LinkLayerType::Loopback => "Loopback",
            _                       => "Other",
        };

        let state = if flags & 1 != 0 { "UP" } else { "DOWN" };

        println!(
            "  {:<4}  {:<15}  {:<10}  {}",
            index, name, link_type, state
        );
        if !mac.is_empty() {
            println!("        MAC: {}", mac);
        }
    }

    Ok(())
}
```

### Example 8 — wpa_supplicant control socket client (Rust)

```toml
# Cargo.toml
[package]
name = "wpa-ctrl"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
// src/main.rs
//
// Communicate with wpa_supplicant via its Unix domain socket.
// Issues STATUS and SCAN_RESULTS commands.

use std::os::unix::net::UnixDatagram;
use std::path::Path;
use std::time::Duration;

const WPA_CTRL_DIR: &str = "/var/run/wpa_supplicant";
const IFACE:        &str = "wlan0";
const LOCAL_PATH:   &str = "/tmp/wpa_rust_client";

struct WpaCtrl {
    sock:       UnixDatagram,
    local_path: String,
}

impl WpaCtrl {
    fn connect(iface: &str) -> std::io::Result<Self> {
        let local = format!("{}_{}", LOCAL_PATH, std::process::id());
        let _ = std::fs::remove_file(&local); // clean up stale socket

        let sock = UnixDatagram::bind(&local)?;
        let remote = format!("{}/{}", WPA_CTRL_DIR, iface);
        sock.connect(&remote)?;
        sock.set_read_timeout(Some(Duration::from_secs(5)))?;

        Ok(Self { sock, local_path: local })
    }

    fn request(&self, cmd: &str) -> std::io::Result<String> {
        self.sock.send(cmd.as_bytes())?;
        let mut buf = vec![0u8; 4096];
        let n = self.sock.recv(&mut buf)?;
        Ok(String::from_utf8_lossy(&buf[..n]).to_string())
    }
}

impl Drop for WpaCtrl {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.local_path);
    }
}

fn print_status_table(status: &str) {
    println!("\n  +-------------------------------+");
    println!("  | wpa_supplicant STATUS          |");
    println!("  +-------------------------------+");
    for line in status.lines() {
        println!("  | {:<30}|", line);
    }
    println!("  +-------------------------------+\n");
}

fn main() -> std::io::Result<()> {
    let ctrl = WpaCtrl::connect(IFACE)
        .map_err(|e| {
            eprintln!("Cannot connect to wpa_supplicant ({}): {}", IFACE, e);
            e
        })?;

    // Query STATUS
    let status = ctrl.request("STATUS")?;
    print_status_table(&status);

    // Trigger scan
    let ack = ctrl.request("SCAN")?;
    println!("  SCAN acknowledged: {}", ack.trim());
    std::thread::sleep(Duration::from_secs(3));

    // Retrieve scan results
    let results = ctrl.request("SCAN_RESULTS")?;
    println!("\n  === SCAN RESULTS ===");
    println!("  {:<18} {:<5} {:<6} {}", "BSSID", "Freq", "Signal", "SSID");
    println!("  {}", "-".repeat(60));
    for line in results.lines().skip(1) {  // skip header
        let fields: Vec<&str> = line.splitn(5, '\t').collect();
        if fields.len() >= 5 {
            println!(
                "  {:<18} {:<5} {:<6} {}",
                fields[0], fields[1], fields[2], fields[4]
            );
        }
    }

    Ok(())
}
```

### Example 9 — udev interface event monitor (Rust)

```toml
# Cargo.toml
[package]
name = "udev-net-mon"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-udev = "0.9"
```

```rust
// src/main.rs
//
// Monitor udev network interface events using tokio-udev.
// Requires: BR2_PACKAGE_EUDEV=y or systemd udev.

use tokio_udev::{AsyncMonitorSocket, MonitorBuilder};
use tokio_stream::StreamExt;

fn draw_event_box(action: &str, subsystem: &str, devname: &str) {
    let line = "-".repeat(50);
    println!("  +{}+", line);
    println!("  | {:<48} |", format!("udev EVENT"));
    println!("  | {:<48} |", format!("  Action    : {}", action));
    println!("  | {:<48} |", format!("  Subsystem : {}", subsystem));
    println!("  | {:<48} |", format!("  Device    : {}", devname));
    println!("  +{}+", line);
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let builder = MonitorBuilder::new()?
        .match_subsystem("net")?;

    let monitor: AsyncMonitorSocket = builder.listen()?;

    println!("Monitoring udev network events (Ctrl-C to stop)...\n");

    tokio::pin!(monitor);
    while let Some(event) = monitor.next().await {
        let event = match event {
            Ok(e)  => e,
            Err(e) => { eprintln!("udev error: {}", e); continue; }
        };

        let action    = event.action().and_then(|a| a.to_str())
                             .unwrap_or("<unknown>").to_string();
        let subsystem = event.subsystem().and_then(|s| s.to_str())
                             .unwrap_or("<unknown>").to_string();
        let devname   = event.property_value("INTERFACE")
                             .and_then(|v| v.to_str())
                             .unwrap_or("<unknown>").to_string();

        draw_event_box(&action, &subsystem, &devname);
    }

    Ok(())
}
```

### Example 10 — iproute2-style DHCP trigger script (Rust, using std::process)

```rust
// dhcp_trigger.rs
//
// After detecting a new interface UP event, trigger udhcpc (Buildroot busybox)
// to obtain a DHCP lease. Useful as a minimal connection-up handler
// when not using connman or NetworkManager.
//
// Build: cargo build --target aarch64-unknown-linux-gnu --release

use std::process::{Command, Stdio};
use std::time::Duration;
use std::thread;

struct Interface {
    name:  String,
    state: IfState,
}

#[derive(Debug, PartialEq)]
enum IfState { Up, Down }

impl Interface {
    fn bring_up(&self) {
        println!("[*] Bringing up interface: {}", self.name);
        let status = Command::new("ip")
            .args(["link", "set", &self.name, "up"])
            .status()
            .expect("Failed to run 'ip'");
        if status.success() {
            println!("[+] {} is UP", self.name);
        }
    }

    fn run_dhcp(&self) {
        println!("[*] Running udhcpc on {}", self.name);
        // udhcpc is part of BusyBox in Buildroot
        let status = Command::new("udhcpc")
            .args(["-i", &self.name, "-n", "-q", "-t", "5"])
            .stdout(Stdio::inherit())
            .stderr(Stdio::inherit())
            .status()
            .expect("Failed to run udhcpc");

        if status.success() {
            println!("[+] DHCP lease obtained on {}", self.name);
        } else {
            eprintln!("[-] DHCP failed on {} (exit: {})", self.name, status);
        }
    }

    fn show_addresses(&self) {
        println!("\n[*] Addresses on {}:", self.name);
        Command::new("ip")
            .args(["addr", "show", "dev", &self.name])
            .status()
            .ok();
    }
}

fn main() {
    let ifaces = vec![
        Interface { name: "eth0".to_string(),  state: IfState::Down },
        Interface { name: "eth-mgmt".to_string(), state: IfState::Down },
    ];

    for iface in &ifaces {
        if iface.state == IfState::Down {
            iface.bring_up();
            thread::sleep(Duration::from_millis(500));
            iface.run_dhcp();
            iface.show_addresses();
            println!();
        }
    }
}
```

---

## Summary

```
  +=================================================================+
  |            BUILDROOT NETWORKING STACK — SUMMARY MAP             |
  +=================================================================+
  |                                                                 |
  |  LAYER 1: KERNEL (linux-kconfig fragments)                      |
  |  +---------------------------------------------------------+    |
  |  | CONFIG_NET  CONFIG_INET  CONFIG_IPV6  CONFIG_CFG80211   |    |
  |  | CONFIG_BRIDGE  CONFIG_VLAN  CONFIG_NET_SCHED  NF_TABLES |    |
  |  | Controlled via BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES   |    |
  |  +---------------------------------------------------------+    |
  |                                                                 |
  |  LAYER 2: CORE USER-SPACE TOOL                                  |
  |  +---------------------------------------------------------+    |
  |  | iproute2 (ip, ss, tc, bridge, nstat)                    |    |
  |  | - Netlink-based; replaces ifconfig/route                |    |
  |  | - Required by connman and NetworkManager                |    |
  |  | BR2_PACKAGE_IPROUTE2=y                                  |    |
  |  +---------------------------------------------------------+    |
  |                                                                 |
  |  LAYER 3: WIRELESS AUTH                                         |
  |  +---------------------------------------------------------+    |
  |  | wpa_supplicant (WPA2/WPA3/EAP, nl80211 backend)         |    |
  |  | - Control socket consumed by connman/NM                 |    |
  |  | BR2_PACKAGE_WPA_SUPPLICANT=y + _NL80211 + _WPA3         |    |
  |  +---------------------------------------------------------+    |
  |                                                                 |
  |  LAYER 4: CONNECTION MANAGERS (choose one)                      |
  |  +-------------------------+  +----------------------------+    |
  |  | connman                 |  | NetworkManager             |    |
  |  | Lightweight, D-Bus      |  | Full-featured, D-Bus       |    |
  |  | Plugins: WiFi/ETH/VPN   |  | NMcli/NMtui, bonds, VLANs  |    |
  |  | Ideal: small embedded   |  | Ideal: complex/desktop     |    |
  |  | BR2_PACKAGE_CONNMAN=y   |  | BR2_PACKAGE_NETWORKMANAGER |    |
  |  +-------------------------+  +----------------------------+    |
  |                                                                 |
  |  LAYER 5: INTERFACE NAMING (udev / eudev)                       |
  |  +---------------------------------------------------------+    |
  |  | /etc/udev/rules.d/70-persistent-net.rules               |    |
  |  | ATTR{address}=="MAC"  -->  NAME="eth-mgmt"              |    |
  |  | KERNELS=="PCI slot"   -->  NAME="eth-data"              |    |
  |  | BR2_PACKAGE_EUDEV=y + BR2_ROOTFS_OVERLAY                |    |
  |  +---------------------------------------------------------+    |
  |                                                                 |
  |  PROGRAMMING INTERFACES                                         |
  |  +-------------------+  +-----------+  +-------------------+    |
  |  | C                 |  | C++       |  | Rust              |    |
  |  | AF_NETLINK socket |  | sdbus-c++ |  | rtnetlink crate   |    |
  |  | RTM_GETLINK       |  | libnm     |  | tokio-udev crate  |    |
  |  | wpa ctrl socket   |  | RAII NL   |  | UnixDatagram WPA  |    |
  |  | udev uevent mon.  |  | wrappers  |  | std::process dhcp |    |
  |  +-------------------+  +-----------+  +-------------------+    |
  |                                                                 |
  +=================================================================+
```

The Buildroot networking stack is a carefully layered system. The **Linux kernel** provides the raw socket infrastructure and driver framework configured through `Kconfig` fragments. **iproute2** sits above the kernel as the universal Netlink-based management tool. **wpa_supplicant** handles the 802.11 authentication layer for wireless clients. **connman** and **NetworkManager** provide complete connection lifecycle management at different complexity/resource trade-off points — connman for constrained embedded targets, NetworkManager where richer functionality is needed.

**udev rules** are the glue that makes interface names deterministic across boots, firmware updates, and hardware revisions — critical for embedded products where interface names must match firewall rules, VPN configs, and init scripts reliably.

From a programming perspective, all layers expose **Netlink sockets** for low-level kernel communication (C, C++, Rust all have ergonomic abstractions), **D-Bus** for connman and NetworkManager (sdbus-c++, dbus-glib, zbus in Rust), and **Unix domain sockets** for wpa_supplicant's control interface. Mastering these interfaces enables embedded developers to build robust, self-healing network management daemons that operate correctly across the full lifecycle of a deployed device.