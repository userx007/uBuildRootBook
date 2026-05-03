# 23. TLS & Cryptographic Libraries in Buildroot

**Structure & Content:**

- **Library comparison matrix** — OpenSSL vs. mbedTLS vs. WolfSSL across footprint, FIPS support, hardware engine API, Rust bindings, and license, plus an ASCII decision tree for choosing between them
- **Buildroot configuration** — `menuconfig` ASCII layout, config fragment snippets, and a minimal `openssl.cnf` for embedded targets
- **Certificate deployment** — filesystem layout diagram, factory provisioning shell script using `openssl ecparam` + `x509`
- **C code examples** — full TLS 1.3 client in OpenSSL, full TLS 1.3 client in mbedTLS (both with graceful error handling)
- **C++ RAII wrappers** — `tls::Session` and `tls::SslCtxPtr` with `unique_ptr` deleters, a `Config` struct, and a working demo
- **Hardware crypto engine** — ASCII stack diagram showing the software/kernel/hardware layers, OpenSSL cryptodev engine demo with graceful software fallback, mbedTLS ALT hook implementation for SoC AES accelerators
- **PKCS#11 in C++** — full RAII session class with EC key pair generation and ECDSA signing via `dlopen`/`CK_FUNCTION_LIST`
- **PKCS#11 in Rust** — `cryptoki` crate usage with RSA-2048 keygen and signing
- **mTLS** — sequence diagram (ASCII) of the full handshake, plus a complete OpenSSL mTLS echo server in C++ with CN extraction
- **Performance benchmark table** — AES and RSA numbers for common Cortex-A cores, SW vs. HW
- **Security hardening checklist** — structured ASCII checklist covering certs, TLS config, Buildroot packaging, and key storage
- **Summary** — narrative overview tying all sections together


> **Buildroot Embedded Systems Series — Chapter 23**
> OpenSSL vs. mbedTLS vs. WolfSSL selection, certificate deployment, hardware crypto engine integration, and PKCS#11 in C++/Rust.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Library Comparison: OpenSSL vs. mbedTLS vs. WolfSSL](#library-comparison)
3. [Buildroot Configuration](#buildroot-configuration)
4. [Certificate Deployment](#certificate-deployment)
5. [TLS Client & Server in C](#tls-client--server-in-c)
6. [TLS in C++ — RAII Wrappers](#tls-in-c--raii-wrappers)
7. [Hardware Crypto Engine Integration](#hardware-crypto-engine-integration)
8. [PKCS#11 — C++ and Rust](#pkcs11--c-and-rust)
9. [Rust TLS with rustls and tokio-rustls](#rust-tls-with-rustls-and-tokio-rustls)
10. [Mutual TLS (mTLS)](#mutual-tls-mtls)
11. [Performance Benchmarking](#performance-benchmarking)
12. [Security Hardening Checklist](#security-hardening-checklist)
13. [Summary](#summary)

---

## Introduction

Embedded Linux systems built with Buildroot increasingly require secure communication channels. Whether you are sending sensor data to a cloud backend, downloading OTA firmware updates, or authenticating an industrial device to a server, TLS (Transport Layer Security) and strong cryptography are non-negotiable.

Buildroot provides three primary TLS/crypto library choices:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │              TLS/Crypto Library Ecosystem in Buildroot          │
  │                                                                 │
  │   ┌───────────┐     ┌───────────┐     ┌───────────────────┐     │
  │   │  OpenSSL  │     │  mbedTLS  │     │     WolfSSL       │     │
  │   │  3.x/1.1  │     │  (Arm)    │     │  (wolfCrypt)      │     │
  │   └─────┬─────┘     └─────┬─────┘     └─────────┬─────────┘     │
  │         │                 │                     │               │
  │   Feature-rich      Lightweight           FIPS-certified        │
  │   Large footprint   ~100KB RAM            Dual-license          │
  │   Hardware engines  mbedTLS engines       wolfEngine (OEM)      │
  │         │                 │                     │               │
  │   ┌─────┴─────────────────┴─────────────────────┴───────────┐   │
  │   │              PKCS#11 Abstraction Layer                  │   │
  │   │         (p11-kit / SoftHSM / Hardware HSM)              │   │
  │   └─────────────────────────────────────────────────────────┘   │
  │                           │                                     │
  │            ┌──────────────┴───────────────┐                     │
  │            │ Applications (C / C++ / Rust)│                     │
  │            └──────────────────────────────┘                     │
  └─────────────────────────────────────────────────────────────────┘
```

This chapter covers: library selection criteria, Buildroot `menuconfig` settings, certificate deployment strategies, C/C++ TLS programming patterns, hardware crypto engine integration, PKCS#11 usage, and Rust-based TLS with `rustls`.

---

## Library Comparison

### Feature Matrix

```
  ┌──────────────────────┬────────────────┬────────────────┬────────────────┐
  │ Feature              │   OpenSSL 3.x  │   mbedTLS 3.x  │   WolfSSL 5.x  │
  ├──────────────────────┼────────────────┼────────────────┼────────────────┤
  │ TLS 1.3 support      │     YES        │     YES        │     YES        │
  │ TLS 1.2 support      │     YES        │     YES        │     YES        │
  │ DTLS support         │     YES        │     YES        │     YES        │
  │ Footprint (shared)   │  ~3-5 MB       │  ~300 KB       │  ~400-600 KB   │
  │ Footprint (static)   │  ~2-4 MB       │  ~60-200 KB    │  ~150-500 KB   │
  │ RAM usage (idle)     │  ~2 MB         │  ~50-100 KB    │  ~100-200 KB   │
  │ FIPS 140-2/3         │  YES (FIPS mod)│     NO         │     YES        │
  │ Hardware accel API   │  OpenSSL Eng.  │  mbedTLS HAL   │  wolfEngine    │
  │ PKCS#11              │  p11-kit/eng.  │  PKCS#11 mod   │  wolfPKCS11    │
  │ Buildroot package    │BR2_PACKAGE_    │BR2_PACKAGE_    │BR2_PACKAGE_    │
  │                      │OPENSSL         │MBEDTLS         │WOLFSSL         │
  │ Rust bindings        │ openssl crate  │ mbedtls crate  │ wolfssl crate  │
  │ License              │ Apache 2.0     │ Apache 2.0     │ GPL2/Commercial│
  │ Common use case      │ General Linux  │ MCU/constrained│ Certified IoT  │
  └──────────────────────┴────────────────┴────────────────┴────────────────┘
```

### When to Choose Which Library

```
  Decision Tree:
  
  START
    │
    ├─► Does your target have < 512 KB RAM?
    │         YES ──► mbedTLS   (smallest footprint, no external deps)
    │         NO  ──► continue
    │
    ├─► Do you need FIPS 140-2/3 certification?
    │         YES ──► WolfSSL (FIPS edition) or OpenSSL (FIPS module)
    │         NO  ──► continue
    │
    ├─► Are you targeting a standard Linux system (glibc)?
    │         YES ──► OpenSSL  (widest software ecosystem, most compatible)
    │         NO  ──► continue
    │
    ├─► Minimal musl/uClibc system with strict size budget?
    │         YES ──► mbedTLS or WolfSSL (can disable unused algorithms)
    │         NO  ──► OpenSSL
    │
    └─► END
```

---

## Buildroot Configuration

### menuconfig Selections

```
  Buildroot menuconfig navigation:
  
  Target packages
    └── Libraries
          └── Security (crypto & TLS)
  
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                  Security/crypto library options                         │
  │                                                                          │
  │  [ ] botan                                                               │
  │  [*] openssl                                                             │
  │        openssl variant (openssl)  --->                                   │
  │        [ ] openssl additional engines                                    │
  │        [ ] openssl force soft abi                                        │
  │  [ ] mbedtls                                                             │
  │        [ ] mbedtls benchmark program                                     │
  │        [ ] mbedtls test programs                                         │
  │  [ ] wolfssl                                                             │
  │        wolfssl features  --->                                            │
  │  [ ] p11-kit                   (PKCS#11 abstraction)                     │
  │  [ ] softhsm2                  (software HSM for testing)                │
  │  [ ] tpm2-tss                  (TPM 2.0 software stack)                  │
  └──────────────────────────────────────────────────────────────────────────┘
```

### Buildroot Config Fragments

```
# configs/my_device_crypto_defconfig

# Choose ONE primary TLS library:
BR2_PACKAGE_OPENSSL=y
# BR2_PACKAGE_MBEDTLS=y
# BR2_PACKAGE_WOLFSSL=y

# OpenSSL specific — hardware engine support
BR2_PACKAGE_OPENSSL_ENGINES=y

# PKCS#11 support layer
BR2_PACKAGE_P11_KIT=y

# If using TPM
BR2_PACKAGE_TPM2_TSS=y
BR2_PACKAGE_TPM2_TOOLS=y

# CA certificates bundle
BR2_PACKAGE_CA_CERTIFICATES=y

# For Rust TLS applications
BR2_PACKAGE_HOST_RUSTC=y
BR2_PACKAGE_RUSTFMT=y
```

### OpenSSL Configuration File — Embedded Minimal

```ini
# /etc/ssl/openssl.cnf (trimmed for embedded)
openssl_conf = openssl_init

[openssl_init]
providers = provider_sect
engines   = engine_section

[provider_sect]
default = default_sect

[default_sect]
activate = 1

[engine_section]
# Uncomment for hardware engine, e.g., cryptodev
# cryptodev = cryptodev_section

# [cryptodev_section]
# engine_id  = cryptodev
# dynamic_path = /usr/lib/engines-3/cryptodev.so
# CIPHERS    = AES-128-CBC:AES-256-CBC
# DIGESTS    = SHA1:SHA256
# init       = 1

[req]
distinguished_name = req_distinguished_name
x509_extensions    = v3_ca
prompt             = no

[req_distinguished_name]
CN = MyEmbeddedDevice
O  = MyOrg
C  = DE
```

---

## Certificate Deployment

### Certificate Storage Layout

```
  /etc/ssl/ layout on target:
  
  /etc/ssl/
  ├── certs/
  │   ├── ca-certificates.crt    ← bundle (ca-certificates package)
  │   ├── my-root-ca.crt         ← custom root CA
  │   └── device-cert.crt        ← this device's certificate
  ├── private/
  │   └── device-key.pem         ← private key (chmod 600!)
  └── openssl.cnf

  Buildroot overlay method:
  board/mydevice/rootfs_overlay/
  └── etc/
      └── ssl/
          ├── certs/
          │   └── my-root-ca.crt
          └── private/
              └── device-key.pem   ← provisioned at factory
```

### Generating Device Certificates

```bash
#!/bin/sh
# scripts/gen_device_certs.sh
# Run during factory provisioning or first-boot init

DEVICE_ID=$(cat /proc/cpuinfo | grep Serial | awk '{print $3}')
KEY_FILE=/etc/ssl/private/device-key.pem
CERT_FILE=/etc/ssl/certs/device-cert.crt
CSR_FILE=/tmp/device.csr
CA_CERT=/etc/ssl/certs/my-root-ca.crt
CA_KEY=/etc/ssl/private/my-root-ca-key.pem  # kept on signing server

# Generate EC key (preferred over RSA for embedded)
openssl ecparam -name prime256v1 -genkey -noout -out $KEY_FILE
chmod 600 $KEY_FILE

# Create CSR
openssl req -new -key $KEY_FILE \
  -subj "/CN=device-${DEVICE_ID}/O=MyOrg/C=DE" \
  -out $CSR_FILE

# Sign with CA (on production server — shown here for demo)
openssl x509 -req -in $CSR_FILE \
  -CA $CA_CERT -CAkey $CA_KEY \
  -CAcreateserial \
  -days 3650 \
  -sha256 \
  -out $CERT_FILE

echo "Certificate deployed: $CERT_FILE"
```

---

## TLS Client & Server in C

### TLS Client with OpenSSL (C)

```c
/* tls_client_openssl.c
 * Simple TLS 1.3 client using OpenSSL
 * Build: gcc tls_client_openssl.c -o tls_client -lssl -lcrypto
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netdb.h>
#include <openssl/ssl.h>
#include <openssl/err.h>

#define HOST        "api.example.com"
#define PORT        "443"
#define CA_BUNDLE   "/etc/ssl/certs/ca-certificates.crt"

static int tcp_connect(const char *host, const char *port)
{
    struct addrinfo hints = {0}, *res, *rp;
    int fd = -1;

    hints.ai_family   = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    if (getaddrinfo(host, port, &hints, &res) != 0)
        return -1;

    for (rp = res; rp; rp = rp->ai_next) {
        fd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
        if (fd < 0) continue;
        if (connect(fd, rp->ai_addr, rp->ai_addrlen) == 0) break;
        close(fd);
        fd = -1;
    }
    freeaddrinfo(res);
    return fd;
}

int main(void)
{
    SSL_CTX *ctx = NULL;
    SSL     *ssl = NULL;
    int      fd  = -1;
    int      ret = EXIT_FAILURE;

    /* ── Initialize OpenSSL ─────────────────────────────────────── */
    OPENSSL_init_ssl(0, NULL);
    ERR_load_crypto_strings();

    /* ── Create TLS context ─────────────────────────────────────── */
    ctx = SSL_CTX_new(TLS_client_method());
    if (!ctx) { ERR_print_errors_fp(stderr); goto cleanup; }

    /* ── Enforce TLS 1.2 minimum ────────────────────────────────── */
    SSL_CTX_set_min_proto_version(ctx, TLS1_2_VERSION);

    /* ── Load trusted CA bundle ─────────────────────────────────── */
    if (!SSL_CTX_load_verify_locations(ctx, CA_BUNDLE, NULL)) {
        fprintf(stderr, "Cannot load CA bundle: %s\n", CA_BUNDLE);
        goto cleanup;
    }
    SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);

    /* ── Optional: load client certificate (mTLS) ───────────────── */
    /* SSL_CTX_use_certificate_file(ctx, "/etc/ssl/certs/device-cert.crt",
                                      SSL_FILETYPE_PEM);
       SSL_CTX_use_PrivateKey_file(ctx, "/etc/ssl/private/device-key.pem",
                                      SSL_FILETYPE_PEM); */

    /* ── TCP connection ─────────────────────────────────────────── */
    fd = tcp_connect(HOST, PORT);
    if (fd < 0) { perror("connect"); goto cleanup; }

    /* ── TLS handshake ──────────────────────────────────────────── */
    ssl = SSL_new(ctx);
    SSL_set_fd(ssl, fd);
    SSL_set_tlsext_host_name(ssl, HOST);   /* SNI */

    if (SSL_connect(ssl) != 1) {
        ERR_print_errors_fp(stderr);
        goto cleanup;
    }

    printf("Connected: %s  Cipher: %s\n",
           SSL_get_version(ssl),
           SSL_get_cipher(ssl));

    /* ── HTTP GET ───────────────────────────────────────────────── */
    const char *req =
        "GET /api/status HTTP/1.1\r\n"
        "Host: " HOST "\r\n"
        "Connection: close\r\n\r\n";

    SSL_write(ssl, req, (int)strlen(req));

    char buf[4096];
    int  n;
    while ((n = SSL_read(ssl, buf, sizeof(buf) - 1)) > 0) {
        buf[n] = '\0';
        printf("%s", buf);
    }

    ret = EXIT_SUCCESS;

cleanup:
    if (ssl) { SSL_shutdown(ssl); SSL_free(ssl); }
    if (fd >= 0) close(fd);
    SSL_CTX_free(ctx);
    return ret;
}
```

### TLS Client with mbedTLS (C)

```c
/* tls_client_mbedtls.c
 * TLS 1.3 client using mbedTLS — suitable for constrained devices
 * Build: gcc tls_client_mbedtls.c -o tls_client_mbed -lmbedtls -lmbedcrypto -lmbedx509
 */

#include <stdio.h>
#include <string.h>
#include "mbedtls/net_sockets.h"
#include "mbedtls/ssl.h"
#include "mbedtls/entropy.h"
#include "mbedtls/ctr_drbg.h"
#include "mbedtls/error.h"

#define HOST     "api.example.com"
#define PORT     "443"
#define CA_FILE  "/etc/ssl/certs/my-root-ca.crt"

int main(void)
{
    mbedtls_net_context       server_fd;
    mbedtls_entropy_context   entropy;
    mbedtls_ctr_drbg_context  ctr_drbg;
    mbedtls_ssl_context       ssl;
    mbedtls_ssl_config        conf;
    mbedtls_x509_crt          cacert;
    int ret;
    char err_buf[256];

    /* ── Initialize structures ──────────────────────────────────── */
    mbedtls_net_init(&server_fd);
    mbedtls_ssl_init(&ssl);
    mbedtls_ssl_config_init(&conf);
    mbedtls_x509_crt_init(&cacert);
    mbedtls_ctr_drbg_init(&ctr_drbg);
    mbedtls_entropy_init(&entropy);

    /* ── Seed the RNG ───────────────────────────────────────────── */
    const char *pers = "tls_client";
    ret = mbedtls_ctr_drbg_seed(&ctr_drbg, mbedtls_entropy_func,
                                 &entropy,
                                 (const unsigned char *)pers, strlen(pers));
    if (ret != 0) goto exit;

    /* ── Load CA certificate ────────────────────────────────────── */
    ret = mbedtls_x509_crt_parse_file(&cacert, CA_FILE);
    if (ret != 0) { fprintf(stderr, "Cannot load CA: %s\n", CA_FILE); goto exit; }

    /* ── Connect ────────────────────────────────────────────────── */
    ret = mbedtls_net_connect(&server_fd, HOST, PORT, MBEDTLS_NET_PROTO_TCP);
    if (ret != 0) goto exit;

    /* ── Configure TLS ──────────────────────────────────────────── */
    ret = mbedtls_ssl_config_defaults(&conf,
                                      MBEDTLS_SSL_IS_CLIENT,
                                      MBEDTLS_SSL_TRANSPORT_STREAM,
                                      MBEDTLS_SSL_PRESET_DEFAULT);
    if (ret != 0) goto exit;

    mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
    mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
    mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);

    /* ── Set minimum TLS version ────────────────────────────────── */
    mbedtls_ssl_conf_min_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_2);

    ret = mbedtls_ssl_setup(&ssl, &conf);
    if (ret != 0) goto exit;

    mbedtls_ssl_set_hostname(&ssl, HOST);  /* SNI + peer CN check */
    mbedtls_ssl_set_bio(&ssl, &server_fd,
                        mbedtls_net_send,
                        mbedtls_net_recv, NULL);

    /* ── Handshake ──────────────────────────────────────────────── */
    while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
        if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
            ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
            mbedtls_strerror(ret, err_buf, sizeof(err_buf));
            fprintf(stderr, "Handshake failed: %s\n", err_buf);
            goto exit;
        }
    }

    printf("Connected via %s\n", mbedtls_ssl_get_version(&ssl));

    /* ── Write HTTP GET ─────────────────────────────────────────── */
    const char *req = "GET /api/status HTTP/1.1\r\nHost: " HOST "\r\n"
                      "Connection: close\r\n\r\n";
    size_t written = 0, len = strlen(req);
    while (written < len) {
        ret = mbedtls_ssl_write(&ssl,
                                (const unsigned char *)req + written,
                                len - written);
        if (ret < 0 && ret != MBEDTLS_ERR_SSL_WANT_WRITE) goto exit;
        if (ret > 0) written += (size_t)ret;
    }

    /* ── Read response ──────────────────────────────────────────── */
    unsigned char buf[1024];
    do {
        ret = mbedtls_ssl_read(&ssl, buf, sizeof(buf) - 1);
        if (ret > 0) { buf[ret] = 0; printf("%s", buf); }
    } while (ret > 0 ||
             ret == MBEDTLS_ERR_SSL_WANT_READ);

    mbedtls_ssl_close_notify(&ssl);

exit:
    mbedtls_net_free(&server_fd);
    mbedtls_x509_crt_free(&cacert);
    mbedtls_ssl_free(&ssl);
    mbedtls_ssl_config_free(&conf);
    mbedtls_ctr_drbg_free(&ctr_drbg);
    mbedtls_entropy_free(&entropy);
    return ret != 0 ? 1 : 0;
}
```

---

## TLS in C++ — RAII Wrappers

C++ RAII (Resource Acquisition Is Initialization) wrappers eliminate the common mistake of forgetting to free OpenSSL objects, and model TLS sessions as proper value types.

```cpp
// tls_raii.hpp
// C++17 RAII wrappers for OpenSSL
// Build: g++ -std=c++17 tls_raii.cpp -o tls_demo -lssl -lcrypto

#pragma once
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <memory>
#include <stdexcept>
#include <string>
#include <string_view>
#include <vector>

namespace tls {

// ── Deleter types ──────────────────────────────────────────────────
struct SslCtxDeleter  { void operator()(SSL_CTX *p) const noexcept { SSL_CTX_free(p); } };
struct SslDeleter     { void operator()(SSL     *p) const noexcept { SSL_free(p);     } };

using SslCtxPtr = std::unique_ptr<SSL_CTX, SslCtxDeleter>;
using SslPtr    = std::unique_ptr<SSL,     SslDeleter>;

// ── TLS configuration ──────────────────────────────────────────────
struct Config {
    std::string ca_bundle    = "/etc/ssl/certs/ca-certificates.crt";
    std::string client_cert;       // empty = no mTLS
    std::string client_key;
    int         min_tls_version = TLS1_2_VERSION;
    bool        verify_peer     = true;
};

// ── Error helpers ──────────────────────────────────────────────────
[[noreturn]] inline void throw_ssl_error(std::string_view msg)
{
    char buf[256];
    ERR_error_string_n(ERR_get_error(), buf, sizeof(buf));
    throw std::runtime_error(std::string(msg) + ": " + buf);
}

// ── Context builder ────────────────────────────────────────────────
inline SslCtxPtr make_client_context(const Config &cfg)
{
    SslCtxPtr ctx{ SSL_CTX_new(TLS_client_method()) };
    if (!ctx) throw_ssl_error("SSL_CTX_new");

    SSL_CTX_set_min_proto_version(ctx.get(), cfg.min_tls_version);

    if (cfg.verify_peer) {
        SSL_CTX_set_verify(ctx.get(), SSL_VERIFY_PEER, nullptr);
        if (!SSL_CTX_load_verify_locations(ctx.get(),
                                           cfg.ca_bundle.c_str(), nullptr))
            throw_ssl_error("load CA bundle");
    } else {
        SSL_CTX_set_verify(ctx.get(), SSL_VERIFY_NONE, nullptr);
    }

    if (!cfg.client_cert.empty()) {
        if (!SSL_CTX_use_certificate_file(ctx.get(),
                cfg.client_cert.c_str(), SSL_FILETYPE_PEM))
            throw_ssl_error("load client cert");
        if (!SSL_CTX_use_PrivateKey_file(ctx.get(),
                cfg.client_key.c_str(), SSL_FILETYPE_PEM))
            throw_ssl_error("load client key");
        if (!SSL_CTX_check_private_key(ctx.get()))
            throw_ssl_error("private key mismatch");
    }
    return ctx;
}

// ── TLS session class ──────────────────────────────────────────────
class Session {
public:
    explicit Session(SSL_CTX *ctx, int fd, std::string_view hostname)
        : ssl_{ SSL_new(ctx) }
    {
        if (!ssl_) throw_ssl_error("SSL_new");
        SSL_set_fd(ssl_.get(), fd);
        SSL_set_tlsext_host_name(ssl_.get(), hostname.data());

        if (SSL_connect(ssl_.get()) != 1)
            throw_ssl_error("SSL_connect");
    }

    ~Session() noexcept
    {
        if (ssl_) SSL_shutdown(ssl_.get());
    }

    // Non-copyable, movable
    Session(const Session &) = delete;
    Session &operator=(const Session &) = delete;
    Session(Session &&)                 = default;

    std::string version()  const { return SSL_get_version(ssl_.get()); }
    std::string cipher()   const { return SSL_get_cipher(ssl_.get()); }

    void write(std::string_view data)
    {
        size_t off = 0;
        while (off < data.size()) {
            int n = SSL_write(ssl_.get(),
                              data.data() + off,
                              static_cast<int>(data.size() - off));
            if (n <= 0) throw_ssl_error("SSL_write");
            off += static_cast<size_t>(n);
        }
    }

    std::string read_all()
    {
        std::string result;
        char buf[4096];
        int  n;
        while ((n = SSL_read(ssl_.get(), buf, sizeof(buf))) > 0)
            result.append(buf, static_cast<size_t>(n));
        return result;
    }

private:
    SslPtr ssl_;
};

} // namespace tls


// ── Usage example ──────────────────────────────────────────────────
// tls_raii_demo.cpp

#include "tls_raii.hpp"
#include <iostream>
#include <netdb.h>
#include <unistd.h>
#include <sys/socket.h>

static int tcp_connect(const char *host, const char *port)
{
    addrinfo hints{}, *res;
    hints.ai_family   = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    if (getaddrinfo(host, port, &hints, &res) != 0) return -1;
    int fd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
    connect(fd, res->ai_addr, res->ai_addrlen);
    freeaddrinfo(res);
    return fd;
}

int main()
{
    constexpr auto HOST = "api.example.com";
    constexpr auto PORT = "443";

    try {
        tls::Config cfg;
        cfg.ca_bundle        = "/etc/ssl/certs/ca-certificates.crt";
        cfg.min_tls_version  = TLS1_3_VERSION;  // TLS 1.3 only

        auto ctx = tls::make_client_context(cfg);

        int fd = tcp_connect(HOST, PORT);
        if (fd < 0) throw std::runtime_error("TCP connect failed");

        tls::Session session{ ctx.get(), fd, HOST };

        std::cout << "TLS version : " << session.version() << "\n"
                  << "Cipher suite: " << session.cipher()  << "\n";

        session.write("GET /api/data HTTP/1.1\r\nHost: "
                      + std::string(HOST) + "\r\nConnection: close\r\n\r\n");

        auto resp = session.read_all();
        std::cout << resp << "\n";

        close(fd);
    }
    catch (const std::exception &ex) {
        std::cerr << "Error: " << ex.what() << "\n";
        return 1;
    }
    return 0;
}
```

---

## Hardware Crypto Engine Integration

Many embedded SoCs (NXP i.MX, STM32MP1, Broadcom BCM, etc.) include hardware crypto accelerators. Buildroot supports these via kernel drivers and OpenSSL engine plugins.

### Architecture

```
  Software stack with hardware crypto acceleration:

  ┌─────────────────────────────────────────────────────────────────┐
  │  Application (C / C++ / Rust)                                   │
  ├─────────────────────────────────────────────────────────────────┤
  │  OpenSSL 3.x  ─── EVP API (cipher/digest/asymmetric)            │
  ├────────────────────┬────────────────────────────────────────────┤
  │  Software provider │  Hardware provider (Engine / Provider)     │
  │  (default)         │  e.g. cryptodev-linux, devcrypto, pkcs11   │
  ├────────────────────┴────────────────────────────────────────────┤
  │  Kernel crypto API  (/dev/crypto via cryptodev-linux module)    │
  ├─────────────────────────────────────────────────────────────────┤
  │  Hardware: AES-NI / DMA crypto engine / SE050 / TPM             │
  └─────────────────────────────────────────────────────────────────┘

  Buildroot kernel config for cryptodev:
  CONFIG_CRYPTO_USER_API_HASH=y
  CONFIG_CRYPTO_USER_API_SKCIPHER=y
  CONFIG_CRYPTO_DEV_ALLWINNER_SUN8I_CE=y  (example: Allwinner SoC)
```

### OpenSSL Hardware Engine via cryptodev-linux (C)

```c
/* hw_engine_demo.c
 * Use AES-256-CBC via cryptodev hardware engine
 * Requires: kernel cryptodev module + openssl-engines
 * Build: gcc hw_engine_demo.c -o hw_engine -lssl -lcrypto -ldl
 */

#include <stdio.h>
#include <string.h>
#include <openssl/engine.h>
#include <openssl/evp.h>
#include <openssl/rand.h>
#include <openssl/err.h>

#define CRYPTODEV_ENGINE "cryptodev"

static void print_hex(const char *label, const unsigned char *data, size_t len)
{
    printf("%s: ", label);
    for (size_t i = 0; i < len; i++) printf("%02x", data[i]);
    printf("\n");
}

int main(void)
{
    ENGINE *eng = NULL;
    EVP_CIPHER_CTX *ctx = NULL;
    int ret = 1;

    /* ── Load dynamic engine ─────────────────────────────────────── */
    ENGINE_load_dynamic();
    eng = ENGINE_by_id(CRYPTODEV_ENGINE);
    if (!eng) {
        fprintf(stderr, "Engine '%s' not available — falling back to software\n",
                CRYPTODEV_ENGINE);
        /* Graceful fallback: continue without hardware acceleration */
        eng = NULL;
    } else {
        if (!ENGINE_init(eng)) {
            ERR_print_errors_fp(stderr);
            goto cleanup;
        }
        ENGINE_set_default(eng, ENGINE_METHOD_ALL);
        printf("Hardware engine loaded: %s (%s)\n",
               ENGINE_get_id(eng), ENGINE_get_name(eng));
    }

    /* ── Generate random key and IV ─────────────────────────────── */
    unsigned char key[32], iv[16];
    RAND_bytes(key, sizeof(key));
    RAND_bytes(iv,  sizeof(iv));
    print_hex("Key", key, sizeof(key));
    print_hex("IV ", iv,  sizeof(iv));

    /* ── Encrypt ─────────────────────────────────────────────────── */
    const unsigned char plaintext[] = "Hello hardware crypto engine!   ";
    unsigned char ciphertext[64];
    int len, ciphertext_len;

    ctx = EVP_CIPHER_CTX_new();
    EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), eng, key, iv);
    EVP_EncryptUpdate(ctx, ciphertext, &len, plaintext, sizeof(plaintext));
    ciphertext_len = len;
    EVP_EncryptFinal_ex(ctx, ciphertext + len, &len);
    ciphertext_len += len;
    print_hex("Ciphertext", ciphertext, ciphertext_len);

    /* ── Decrypt ─────────────────────────────────────────────────── */
    unsigned char decrypted[64] = {0};
    int decrypted_len;
    EVP_CIPHER_CTX_reset(ctx);
    EVP_DecryptInit_ex(ctx, EVP_aes_256_cbc(), eng, key, iv);
    EVP_DecryptUpdate(ctx, decrypted, &len, ciphertext, ciphertext_len);
    decrypted_len = len;
    EVP_DecryptFinal_ex(ctx, decrypted + len, &len);
    decrypted_len += len;

    printf("Decrypted: %.*s\n", decrypted_len, decrypted);
    ret = 0;

cleanup:
    EVP_CIPHER_CTX_free(ctx);
    if (eng) { ENGINE_finish(eng); ENGINE_free(eng); }
    return ret;
}
```

### mbedTLS Hardware Acceleration (C) — Custom HAL

```c
/* mbedtls_hw_alt.c
 * mbedTLS hardware alternate implementation via ALT hooks
 * Define MBEDTLS_AES_ALT in mbedtls/config.h to activate
 */

#include "mbedtls/aes.h"
#include <string.h>

/* Simulated hardware registers (replace with real SoC MMIO) */
#define HW_CRYPTO_BASE  0x40020000UL   /* example SoC address */

typedef struct {
    volatile uint32_t CTRL;
    volatile uint32_t STATUS;
    volatile uint32_t KEY[8];   /* 256-bit key */
    volatile uint32_t DATA_IN[4];
    volatile uint32_t DATA_OUT[4];
} HwCryptoRegs;

static HwCryptoRegs *hw = (HwCryptoRegs *)HW_CRYPTO_BASE;

/* mbedTLS alternate context — matches mbedtls_aes_context layout */
typedef struct mbedtls_aes_context {
    int      nr;           /* number of rounds */
    uint32_t key[68];      /* opaque storage   */
} mbedtls_aes_context;

void mbedtls_aes_init(mbedtls_aes_context *ctx)
{
    memset(ctx, 0, sizeof(*ctx));
}

void mbedtls_aes_free(mbedtls_aes_context *ctx)
{
    /* Zeroize key material before freeing */
    memset(ctx, 0, sizeof(*ctx));
}

int mbedtls_aes_setkey_enc(mbedtls_aes_context *ctx,
                            const unsigned char *key, unsigned int keybits)
{
    if (keybits != 128 && keybits != 192 && keybits != 256)
        return MBEDTLS_ERR_AES_INVALID_KEY_LENGTH;

    ctx->nr = (keybits / 32) + 6;

    /* Load key into hardware registers */
    const uint32_t *kw = (const uint32_t *)key;
    for (int i = 0; i < (int)(keybits / 32); i++)
        hw->KEY[i] = kw[i];

    /* Also save for software fallback */
    memcpy(ctx->key, key, keybits / 8);
    return 0;
}

int mbedtls_aes_crypt_ecb(mbedtls_aes_context *ctx, int mode,
                           const unsigned char input[16],
                           unsigned char output[16])
{
    const uint32_t *in  = (const uint32_t *)input;
    uint32_t       *out = (uint32_t *)output;

    /* Load input data */
    hw->DATA_IN[0] = in[0];
    hw->DATA_IN[1] = in[1];
    hw->DATA_IN[2] = in[2];
    hw->DATA_IN[3] = in[3];

    /* Trigger: ENCRYPT=1, DECRYPT=0 */
    hw->CTRL = (mode == MBEDTLS_AES_ENCRYPT) ? 0x01 : 0x03;

    /* Poll completion (real code: use interrupt/DMA) */
    while (hw->STATUS & 0x01)
        __asm__ volatile("nop");

    out[0] = hw->DATA_OUT[0];
    out[1] = hw->DATA_OUT[1];
    out[2] = hw->DATA_OUT[2];
    out[3] = hw->DATA_OUT[3];

    return 0;
}
```

---

## PKCS#11 — C++ and Rust

PKCS#11 is the standard API for hardware security modules (HSMs), smart cards, TPMs, and SE (Secure Element) devices. It decouples the application from the specific hardware.

### PKCS#11 Architecture

```
  PKCS#11 ecosystem:

  ┌─────────────────────────────────────────────────────────────────┐
  │  Application: key generation, signing, encryption               │
  │  (C++/Rust — never touches raw key bytes!)                      │
  ├─────────────────────────────────────────────────────────────────┤
  │  PKCS#11 API  (CK_FUNCTION_LIST, C_SignInit, C_Sign, ...)       │
  ├──────────────┬──────────────────┬───────────────────────────────┤
  │  SoftHSM2    │  TPM2-PKCS11     │  NXP SE050 PKCS11 lib         │
  │  (software,  │  (tpm2-pkcs11,   │  (Secure Element,             │
  │   testing)   │   hardware TPM)  │   hardware key storage)       │
  ├──────────────┴──────────────────┴───────────────────────────────┤
  │          p11-kit  (module discovery, trust anchor mgmt)         │
  └─────────────────────────────────────────────────────────────────┘

  p11-kit configuration on Buildroot target:
  /etc/pkcs11/modules/softhsm2.module
  /etc/pkcs11/modules/tpm2.module

  Module file example (softhsm2.module):
  module: /usr/lib/softhsm/libsofthsm2.so
  managed: yes
```

### PKCS#11 in C++ — Key Generation and Signing

```cpp
// pkcs11_demo.cpp
// RAII wrapper around PKCS#11 for embedded use
// Build: g++ -std=c++17 pkcs11_demo.cpp -o pkcs11_demo -ldl

#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>
#include <memory>
#include <dlfcn.h>

#define CRYPTOKI_GNU
#include <pkcs11.h>    // from p11-kit or system PKCS#11 headers

// ── Library path — override via environment or compile-time flag ───
#ifndef PKCS11_MODULE
#define PKCS11_MODULE "/usr/lib/softhsm/libsofthsm2.so"
#endif

// ── RAII PKCS#11 session ──────────────────────────────────────────
class Pkcs11Session {
public:
    explicit Pkcs11Session(CK_SLOT_ID slot, const std::string &pin)
    {
        // Dynamically load PKCS#11 library
        lib_ = dlopen(PKCS11_MODULE, RTLD_NOW | RTLD_LOCAL);
        if (!lib_) throw std::runtime_error(dlerror());

        auto getList = reinterpret_cast<CK_C_GetFunctionList>(
                           dlsym(lib_, "C_GetFunctionList"));
        if (!getList) throw std::runtime_error("C_GetFunctionList missing");

        CK_RV rv = getList(&fn_);
        check(rv, "C_GetFunctionList");

        rv = fn_->C_Initialize(nullptr);
        check(rv, "C_Initialize");

        rv = fn_->C_OpenSession(slot,
                                CKF_SERIAL_SESSION | CKF_RW_SESSION,
                                nullptr, nullptr, &session_);
        check(rv, "C_OpenSession");

        rv = fn_->C_Login(session_, CKU_USER,
                          reinterpret_cast<CK_UTF8CHAR_PTR>(
                              const_cast<char *>(pin.c_str())),
                          static_cast<CK_ULONG>(pin.size()));
        check(rv, "C_Login");
    }

    ~Pkcs11Session()
    {
        if (fn_) {
            fn_->C_Logout(session_);
            fn_->C_CloseSession(session_);
            fn_->C_Finalize(nullptr);
        }
        if (lib_) dlclose(lib_);
    }

    // ── Generate EC key pair on token ─────────────────────────────
    std::pair<CK_OBJECT_HANDLE, CK_OBJECT_HANDLE> generate_ec_keypair(
            const std::string &label)
    {
        // OID for prime256v1 (secp256r1)
        static const CK_BYTE ec_params[] = {
            0x06, 0x08, 0x2a, 0x86, 0x48, 0xce, 0x3d, 0x03, 0x01, 0x07
        };

        CK_BBOOL yes = CK_TRUE, no = CK_FALSE;
        CK_OBJECT_CLASS pub_class  = CKO_PUBLIC_KEY;
        CK_OBJECT_CLASS priv_class = CKO_PRIVATE_KEY;
        CK_KEY_TYPE key_type       = CKK_EC;

        CK_ATTRIBUTE pub_tmpl[] = {
            { CKA_CLASS,          &pub_class,  sizeof(pub_class)  },
            { CKA_KEY_TYPE,       &key_type,   sizeof(key_type)   },
            { CKA_TOKEN,          &yes,        sizeof(yes)        },
            { CKA_VERIFY,         &yes,        sizeof(yes)        },
            { CKA_EC_PARAMS,      (void*)ec_params, sizeof(ec_params) },
            { CKA_LABEL,          (void*)label.c_str(), label.size()  },
        };
        CK_ATTRIBUTE priv_tmpl[] = {
            { CKA_CLASS,          &priv_class, sizeof(priv_class) },
            { CKA_KEY_TYPE,       &key_type,   sizeof(key_type)   },
            { CKA_TOKEN,          &yes,        sizeof(yes)        },
            { CKA_SENSITIVE,      &yes,        sizeof(yes)        },
            { CKA_EXTRACTABLE,    &no,         sizeof(no)         },
            { CKA_SIGN,           &yes,        sizeof(yes)        },
            { CKA_LABEL,          (void*)label.c_str(), label.size()  },
        };

        CK_MECHANISM mech = { CKM_EC_KEY_PAIR_GEN, nullptr, 0 };
        CK_OBJECT_HANDLE hPub, hPriv;

        CK_RV rv = fn_->C_GenerateKeyPair(
                session_, &mech,
                pub_tmpl,  sizeof(pub_tmpl)  / sizeof(pub_tmpl[0]),
                priv_tmpl, sizeof(priv_tmpl) / sizeof(priv_tmpl[0]),
                &hPub, &hPriv);
        check(rv, "C_GenerateKeyPair");

        std::cout << "EC key pair generated. Pub=" << hPub
                  << "  Priv=" << hPriv << "\n";
        return {hPub, hPriv};
    }

    // ── Sign data with ECDSA ──────────────────────────────────────
    std::vector<uint8_t> sign_ecdsa(CK_OBJECT_HANDLE hPriv,
                                    const std::vector<uint8_t> &data)
    {
        CK_MECHANISM mech = { CKM_ECDSA_SHA256, nullptr, 0 };
        CK_RV rv = fn_->C_SignInit(session_, &mech, hPriv);
        check(rv, "C_SignInit");

        CK_ULONG sig_len = 0;
        // First call: get signature length
        rv = fn_->C_Sign(session_,
                         const_cast<CK_BYTE_PTR>(data.data()),
                         data.size(), nullptr, &sig_len);
        check(rv, "C_Sign (length query)");

        std::vector<uint8_t> sig(sig_len);
        rv = fn_->C_Sign(session_,
                         const_cast<CK_BYTE_PTR>(data.data()),
                         data.size(), sig.data(), &sig_len);
        check(rv, "C_Sign");
        sig.resize(sig_len);
        return sig;
    }

    CK_SESSION_HANDLE handle() const { return session_; }

private:
    void check(CK_RV rv, const char *op)
    {
        if (rv != CKR_OK) {
            throw std::runtime_error(std::string(op) +
                                     " failed: RV=0x" +
                                     std::to_string(rv));
        }
    }

    void                *lib_     = nullptr;
    CK_FUNCTION_LIST    *fn_      = nullptr;
    CK_SESSION_HANDLE    session_ = CK_INVALID_HANDLE;
};

int main()
{
    try {
        // Slot 0, user PIN — adjust for your HSM
        Pkcs11Session sess(0, "1234");

        auto [hPub, hPriv] = sess.generate_ec_keypair("device-signing-key");

        std::vector<uint8_t> payload = { 0xDE, 0xAD, 0xBE, 0xEF,
                                         0x01, 0x02, 0x03, 0x04 };
        auto signature = sess.sign_ecdsa(hPriv, payload);

        std::cout << "ECDSA-SHA256 signature (" << signature.size()
                  << " bytes): ";
        for (auto b : signature) printf("%02x", b);
        printf("\n");
    }
    catch (const std::exception &ex) {
        std::cerr << "Error: " << ex.what() << "\n";
        return 1;
    }
    return 0;
}
```

---

## Rust TLS with rustls and tokio-rustls

Rust's TLS ecosystem centers around `rustls` — a modern, memory-safe TLS library written entirely in Rust, with no C dependencies. For Buildroot Rust cross-compilation, use `BR2_PACKAGE_HOST_RUSTC=y` and a cross-toolchain cargo config.

### Cargo.toml

```toml
[package]
name    = "embedded-tls-client"
version = "0.1.0"
edition = "2021"

[dependencies]
rustls          = { version = "0.23", features = ["ring"] }
rustls-pemfile  = "2"
tokio           = { version = "1",  features = ["full"] }
tokio-rustls    = "0.26"
webpki-roots    = "0.26"   # Mozilla root CAs
anyhow          = "1"
```

### Async TLS Client in Rust

```rust
// src/main.rs — Async TLS 1.3 client with tokio + rustls

use std::sync::Arc;
use std::io::BufReader;
use std::fs::File;

use anyhow::{Context, Result};
use rustls::{ClientConfig, RootCertStore};
use rustls_pemfile::certs;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio_rustls::TlsConnector;

const HOST: &str = "api.example.com";
const PORT: u16  = 443;

/// Load a PEM file as a certificate list
fn load_certs(path: &str) -> Result<Vec<rustls::pki_types::CertificateDer<'static>>> {
    let f = File::open(path).with_context(|| format!("open {path}"))?;
    let mut rd = BufReader::new(f);
    Ok(certs(&mut rd).collect::<Result<_, _>>()
        .context("parse PEM certs")?)
}

/// Build a rustls client config with a custom CA bundle
fn make_tls_config(ca_path: Option<&str>) -> Result<ClientConfig> {
    let mut root_store = RootCertStore::empty();

    if let Some(path) = ca_path {
        // Custom CA (embedded device with private PKI)
        for cert in load_certs(path)? {
            root_store.add(cert).context("add CA cert")?;
        }
    } else {
        // Use Mozilla WebPKI roots (internet-facing)
        root_store.extend(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
    }

    let config = ClientConfig::builder()
        .with_root_certificates(root_store)
        .with_no_client_auth();   // No mTLS here

    Ok(config)
}

#[tokio::main]
async fn main() -> Result<()> {
    // Use custom CA for internal PKI, or None for public roots
    let tls_config = make_tls_config(Some("/etc/ssl/certs/my-root-ca.crt"))?;
    let connector  = TlsConnector::from(Arc::new(tls_config));

    // TCP connect
    let addr = format!("{HOST}:{PORT}");
    let stream = TcpStream::connect(&addr)
        .await
        .with_context(|| format!("TCP connect to {addr}"))?;

    // TLS handshake
    let server_name = HOST.try_into().expect("invalid hostname");
    let mut tls_stream = connector.connect(server_name, stream)
        .await
        .context("TLS handshake")?;

    println!("Connected to {HOST}:{PORT}");

    // Send HTTP GET
    let request = format!(
        "GET /api/status HTTP/1.1\r\nHost: {HOST}\r\nConnection: close\r\n\r\n"
    );
    tls_stream.write_all(request.as_bytes()).await.context("write")?;

    // Read response
    let mut response = Vec::new();
    tls_stream.read_to_end(&mut response).await.context("read")?;
    println!("{}", String::from_utf8_lossy(&response));

    Ok(())
}
```

### PKCS#11 in Rust — cryptoki crate

```rust
// pkcs11_rust.rs — PKCS#11 key operations in Rust
// Cargo.toml: cryptoki = "0.6"

use cryptoki::{
    context::{CInitializeArgs, Pkcs11},
    mechanism::Mechanism,
    object::{Attribute, AttributeType, KeyType, ObjectClass},
    session::UserType,
    types::AuthPin,
};
use std::path::PathBuf;

const PKCS11_LIB: &str = "/usr/lib/softhsm/libsofthsm2.so";
const USER_PIN:   &str = "1234";

fn main() -> anyhow::Result<()> {
    // ── Initialize PKCS#11 library ────────────────────────────────
    let pkcs11 = Pkcs11::new(PathBuf::from(PKCS11_LIB))
        .context("load PKCS#11 module")?;
    pkcs11.initialize(CInitializeArgs::OsThreads)?;

    // ── Get first available slot ──────────────────────────────────
    let slots = pkcs11.get_slots_with_token()?;
    let slot = *slots.first().expect("no PKCS#11 slots found");

    // ── Open session ──────────────────────────────────────────────
    let session = pkcs11.open_rw_session(slot)?;
    session.login(UserType::User, Some(&AuthPin::new(USER_PIN.into())))?;
    println!("PKCS#11 session opened on slot {slot:?}");

    // ── Search for existing key by label ─────────────────────────
    let label = "rust-device-key";
    let search = vec![
        Attribute::Class(ObjectClass::PRIVATE_KEY),
        Attribute::Label(label.as_bytes().to_vec()),
    ];
    let found = session.find_objects(&search)?;

    let priv_key = if let Some(&handle) = found.first() {
        println!("Found existing key: {handle:?}");
        handle
    } else {
        // ── Generate new RSA-2048 key pair ────────────────────────
        println!("Generating new RSA-2048 key pair...");

        let pub_attrs = vec![
            Attribute::Token(true),
            Attribute::Verify(true),
            Attribute::ModulusBits(2048.into()),
            Attribute::PublicExponent(vec![0x01, 0x00, 0x01]),
            Attribute::Label(label.as_bytes().to_vec()),
        ];
        let priv_attrs = vec![
            Attribute::Token(true),
            Attribute::Sensitive(true),
            Attribute::Extractable(false),
            Attribute::Sign(true),
            Attribute::Label(label.as_bytes().to_vec()),
        ];

        let (_, priv_handle) = session.generate_key_pair(
            &Mechanism::RsaPkcsKeyPairGen,
            &pub_attrs,
            &priv_attrs,
        )?;
        println!("Key pair generated: priv={priv_handle:?}");
        priv_handle
    };

    // ── Sign a message ────────────────────────────────────────────
    let message = b"Embedded Rust PKCS11 signing demo";
    let signature = session.sign(
        &Mechanism::RsaPkcsSha256,
        priv_key,
        message,
    )?;

    print!("RSA-PKCS#1-SHA256 signature ({} bytes): ", signature.len());
    for b in &signature[..8] { print!("{b:02x}"); }
    println!("...");

    // ── Logout and cleanup (handled by Drop) ──────────────────────
    session.logout()?;
    pkcs11.finalize()?;
    Ok(())
}
```

---

## Mutual TLS (mTLS)

mTLS requires both the server and the client to present valid certificates. This is the standard for device-to-cloud authentication in industrial IoT.

### mTLS Flow Diagram

```
  mTLS handshake sequence:

  Device (Client)                      Server (Backend)
       │                                     │
       │──── 1. ClientHello ────────────────►│
       │                                     │
       │◄─── 2. ServerHello ─────────────────│
       │◄─── 3. Server Certificate ──────────│
       │◄─── 4. CertificateRequest ──────────│  ← asks for client cert
       │◄─── 5. ServerHelloDone ─────────────│
       │                                     │
       │  [Verify server cert against CA]    │
       │                                     │
       │──── 6. Client Certificate ─────────►│  ← device cert
       │──── 7. ClientKeyExchange ──────────►│
       │──── 8. CertificateVerify ──────────►│  ← signed by device key
       │──── 9. ChangeCipherSpec ───────────►│
       │──── 10. Finished ──────────────────►│
       │                                     │
       │  [Verify client cert against CA]    │
       │                                     │
       │◄─── 11. ChangeCipherSpec ───────────│
       │◄─── 12. Finished ───────────────────│
       │                                     │
       │════════ Encrypted channel ══════════│
       │      (device authenticated!)        │
```

### mTLS Server in C++ with OpenSSL

```cpp
// mtls_server.cpp
// Simple mTLS echo server requiring client certificate
// Build: g++ -std=c++17 mtls_server.cpp -o mtls_server -lssl -lcrypto

#include <iostream>
#include <stdexcept>
#include <cstring>
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/x509v3.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

#define SERVER_CERT  "/etc/ssl/certs/server-cert.pem"
#define SERVER_KEY   "/etc/ssl/private/server-key.pem"
#define CA_CERT      "/etc/ssl/certs/my-root-ca.crt"
#define PORT         8443

static SSL_CTX *create_server_ctx()
{
    SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());
    if (!ctx) throw std::runtime_error("SSL_CTX_new failed");

    // Server certificate and key
    if (SSL_CTX_use_certificate_file(ctx, SERVER_CERT, SSL_FILETYPE_PEM) != 1 ||
        SSL_CTX_use_PrivateKey_file(ctx, SERVER_KEY,  SSL_FILETYPE_PEM) != 1 ||
        SSL_CTX_check_private_key(ctx) != 1)
        throw std::runtime_error("Server cert/key error");

    // Require client certificate signed by our CA
    if (!SSL_CTX_load_verify_locations(ctx, CA_CERT, nullptr))
        throw std::runtime_error("Cannot load CA cert");

    SSL_CTX_set_verify(ctx,
        SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT,
        nullptr);

    SSL_CTX_set_min_proto_version(ctx, TLS1_2_VERSION);
    return ctx;
}

static void handle_client(SSL_CTX *ctx, int client_fd)
{
    SSL *ssl = SSL_new(ctx);
    SSL_set_fd(ssl, client_fd);

    if (SSL_accept(ssl) != 1) {
        ERR_print_errors_fp(stderr);
        SSL_free(ssl);
        return;
    }

    // Extract and print client CN
    X509 *peer = SSL_get_peer_certificate(ssl);
    if (peer) {
        char cn[256];
        X509_NAME_get_text_by_NID(X509_get_subject_name(peer),
                                   NID_commonName, cn, sizeof(cn));
        std::cout << "Client authenticated: CN=" << cn << "\n";
        X509_free(peer);
    }

    char buf[1024];
    int  n = SSL_read(ssl, buf, sizeof(buf) - 1);
    if (n > 0) {
        buf[n] = '\0';
        std::cout << "Received: " << buf << "\n";
        SSL_write(ssl, buf, n);  // echo back
    }

    SSL_shutdown(ssl);
    SSL_free(ssl);
}

int main()
{
    SSL_CTX *ctx = create_server_ctx();

    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(server_fd, reinterpret_cast<sockaddr *>(&addr), sizeof(addr));
    listen(server_fd, 5);

    std::cout << "mTLS server listening on port " << PORT << "\n";

    while (true) {
        int client_fd = accept(server_fd, nullptr, nullptr);
        if (client_fd >= 0) {
            handle_client(ctx, client_fd);
            close(client_fd);
        }
    }

    SSL_CTX_free(ctx);
    close(server_fd);
    return 0;
}
```

---

## Performance Benchmarking

Comparing software vs. hardware-accelerated crypto on typical embedded SoCs:

```
  Benchmark results — AES-256-CBC, 1 MB payload
  (Illustrative values; actual results vary by SoC)

  ┌────────────────────────────┬──────────────────┬──────────────────┐
  │ Platform                   │ Software (MB/s)  │ Hardware (MB/s)  │
  ├────────────────────────────┼──────────────────┼──────────────────┤
  │ Cortex-A7 @ 800 MHz        │      42          │      350         │
  │ Cortex-A53 @ 1.2 GHz       │      95          │      800         │
  │ Cortex-A55 @ 1.8 GHz       │     190          │     1200+        │
  │ Cortex-A72 @ 1.5 GHz       │     280          │     2000+        │
  └────────────────────────────┴──────────────────┴──────────────────┘

  RSA-2048 sign operations/second:
  ┌────────────────────────────┬──────────────────┬──────────────────┐
  │ Platform                   │ Software (ops/s) │ HW/PKCS11 (ops/s)│
  ├────────────────────────────┼──────────────────┼──────────────────┤
  │ Cortex-A7 @ 800 MHz        │       55         │      140         │
  │ Cortex-A53 @ 1.2 GHz       │      120         │      400         │
  └────────────────────────────┴──────────────────┴──────────────────┘

  Run your own benchmarks:
  openssl speed -evp aes-256-cbc
  openssl speed -engine cryptodev -evp aes-256-cbc   # with HW engine
  openssl speed rsa2048
  mbedtls_benchmark (mbedTLS benchmark app)
```

---

## Security Hardening Checklist

```
  TLS/Crypto Security Hardening for Buildroot Embedded:

  Certificate Management
  ├── [x] Use EC keys (prime256v1) instead of RSA where possible
  ├── [x] Device key generated on-device (never exported)
  ├── [x] Private keys at chmod 600, owned by service user
  ├── [x] Certificate rotation mechanism in place
  └── [x] Short-lived device certs (1-3 years max)

  TLS Configuration
  ├── [x] Minimum TLS 1.2 enforced (TLS 1.3 preferred)
  ├── [x] Disable SSLv3, TLSv1.0, TLSv1.1 explicitly
  ├── [x] Disable NULL, EXPORT, RC4, MD5 cipher suites
  ├── [x] Enable perfect forward secrecy (ECDHE key exchange)
  ├── [x] Certificate validation: verify peer, check hostname
  ├── [x] OCSP stapling or CRL checking configured
  └── [x] SNI configured for multi-tenant servers

  Buildroot Integration
  ├── [x] Remove openssl CLI from release builds (BR2_PACKAGE_OPENSSL_BIN=n)
  ├── [x] Disable test/demo programs from mbedTLS and wolfSSL
  ├── [x] Strip debug symbols from TLS library in production
  ├── [x] Enable stack protector: -fstack-protector-strong
  ├── [x] Enable ASLR: /proc/sys/kernel/randomize_va_space = 2
  └── [x] Use read-only overlay for /etc/ssl

  Key Storage
  ├── [x] Store private keys in TPM or Secure Element if available
  ├── [x] Use PKCS#11 API to abstract key storage backend
  ├── [x] Never log, print, or transmit key material
  └── [x] Zeroize key material after use (memset_s / explicit_bzero)
```

---

## Summary

This chapter covered the full TLS and cryptographic library landscape for Buildroot-based embedded Linux systems.

**Library Selection** is driven by three axes: footprint (mbedTLS < WolfSSL < OpenSSL), ecosystem compatibility (OpenSSL wins for standard Linux), and certification requirements (WolfSSL or OpenSSL FIPS module for FIPS 140). For heavily constrained devices under 512 KB RAM, mbedTLS is the clear choice. For internet-facing general Linux targets, OpenSSL offers the widest software ecosystem. WolfSSL is the specialist choice when certification matters.

**Buildroot integration** is straightforward: select the library via `menuconfig`, add `BR2_PACKAGE_CA_CERTIFICATES=y` for the root CA bundle, `BR2_PACKAGE_P11_KIT=y` for PKCS#11 support, and deploy device certificates via the `rootfs_overlay` mechanism.

**C programming** with OpenSSL follows the EVP and SSL/BIO pattern. mbedTLS uses a flatter API with explicit context initialization — well-suited to bare-metal or musl targets. Both support hardware acceleration: OpenSSL via the Engine (legacy) or Provider (OpenSSL 3.x) API, mbedTLS via compile-time ALT implementation hooks.

**C++ RAII wrappers** are essential for safe OpenSSL usage — they prevent the pervasive resource leaks that plague raw C OpenSSL code and provide a clean session abstraction for application developers.

**Hardware crypto** integration significantly improves performance and power efficiency. The cryptodev-linux kernel module provides a POSIX `/dev/crypto` interface that the OpenSSL cryptodev engine bridges to user space, enabling AES, SHA, and asymmetric acceleration on a wide range of SoCs.

**PKCS#11** is the industry standard for hardware security modules, TPMs, and secure elements. The C++ `cryptoki` bindings and the Rust `cryptoki` crate both provide ergonomic access. P11-kit handles module discovery and trust policy in a Buildroot-deployed system. Keys never leave the HSM boundary: the application only passes data in and receives signatures or ciphertext out.

**Rust TLS** via `rustls` and `tokio-rustls` delivers memory-safe async TLS with no C dependencies, making it ideal for new embedded services where the Rust toolchain is available in Buildroot. The `cryptoki` crate covers PKCS#11 interactions from Rust with strong type safety.

**Mutual TLS** (mTLS) should be the default authentication model for device-to-cloud communication: it replaces password-based or token-based authentication with cryptographic identity anchored in the device's hardware-protected key.

The security hardening checklist ensures that neither Buildroot packaging defaults nor developer convenience settings create vulnerabilities in production — always enforce TLS 1.2+, validate peer certificates, store keys in hardware where available, and strip unnecessary TLS tooling from release images.

---

*End of Chapter 23 — TLS & Cryptographic Libraries*

*Next: Chapter 24 — Secure Boot & Verified Root of Trust*