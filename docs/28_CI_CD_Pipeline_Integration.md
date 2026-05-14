# 28. CI/CD Pipeline Integration (GitLab / GitHub Actions)

The document follows the five core topics from the chapter spec, each treated in depth:

1. **Caching `dl/` and `ccache`** — explains the directory layouts, the critical `hash_dir=false` ccache setting for portable cache keys across runners, and full YAML cache configuration for both GitLab and GitHub Actions.

2. **Parallelism (`-j$(nproc)`)** — covers how Buildroot's parallelism model works (sequential at package level, parallel within each package), optimal job count formulas, and how to wire `$(nproc)` into CI YAML.

3. **Artifact promotion** — the dev → staging → production flow, version naming conventions with git SHA and build number, and GitLab Package Registry upload via the API.

4. **Docker-based build containers** — a production-ready `Dockerfile` with all Buildroot host dependencies from the manual, ccache pre-configuration, and non-root `builder` user (Buildroot refuses to run as root).

5. **`make legal-info` gating** — the full `manifest.csv` format, the allow/deny/unknown classification logic shown as an ASCII decision tree, and YAML stages that archive legal artifacts for 90 days for audit.

**Code Examples**

| Language | Files | What they do |
|----------|-------|--------------|
| C | `license_check.c`, `build_stats.c` | CSV parser license gate; build-time log aggregator |
| C++ | `artifact_verify.cpp`, `pipeline_config.cpp` | SHA256 artifact integrity verifier; INI board config parser with ASCII table |
| Rust | `ci_monitor.rs`, `cache_manager.rs`, `legal_analyzer.rs` | Pipeline status monitor; `dl/` cache pruner with size chart; license family analyzer |

All graphics are in ASCII art as requested.


## Buildroot Embedded Systems Series — Chapter 28

---

## Table of Contents

1. [Overview](#overview)
2. [Pipeline Architecture](#pipeline-architecture)
3. [Caching `dl/` and `ccache`](#caching-dl-and-ccache)
4. [Parallelism with `-j$(nproc)`](#parallelism-with--jnproc)
5. [Artifact Promotion](#artifact-promotion)
6. [Docker-Based Build Containers](#docker-based-build-containers)
7. [`make legal-info` Gating](#make-legal-info-gating)
8. [GitLab CI Configuration](#gitlab-ci-configuration)
9. [GitHub Actions Configuration](#github-actions-configuration)
10. [C Code Examples](#c-code-examples)
11. [C++ Code Examples](#c-code-examples-1)
12. [Rust Code Examples](#rust-code-examples)
13. [Summary](#summary)

---

## Overview

Integrating Buildroot into a Continuous Integration / Continuous Delivery (CI/CD) pipeline
transforms the embedded Linux build process from a manual, error-prone operation into an
automated, reproducible, and auditable workflow. Buildroot, while conceptually simple (it
is essentially a sophisticated Makefile system), presents unique challenges in CI/CD due
to its long build times, large download footprints, and the need to enforce license
compliance.

The key pillars of a production-grade Buildroot CI/CD pipeline are:

- **Caching** — Avoiding redundant downloads (`dl/`) and recompilation (`ccache`)
- **Parallelism** — Maximising hardware utilisation with `-j$(nproc)`
- **Artifact promotion** — Moving validated build outputs through staging environments
- **Containerisation** — Ensuring build reproducibility via Docker-based environments
- **Legal gating** — Enforcing open-source license compliance with `make legal-info`

```
+-------------------------------------------------------------+
|                  Developer Workstation                      |
|  git push / merge request / pull request                    |
+-----------------------------+-------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|              CI/CD Platform (GitLab / GitHub)               |
|                                                             |
|  +----------+   +----------+   +----------+   +----------+  |
|  | Trigger  |-->|  Build   |-->|  Test    |-->| Promote  |  |
|  | Stage    |   |  Stage   |   |  Stage   |   |  Stage   |  |
|  +----------+   +----------+   +----------+   +----------+  |
|        |              |              |              |       |
|        v              v              v              v       |
|   Checkout       dl/ cache      Run tests      Push to      |
|   defconfig      ccache         legal-info     registry     |
|   Docker pull    make -j N      artifact       artifact     |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|              Target Hardware / QEMU / Simulator             |
+-------------------------------------------------------------+
```

---

## Pipeline Architecture

A well-structured Buildroot CI pipeline is divided into discrete, sequential stages, each
with a clearly defined responsibility and failure domain.

```
STAGE FLOW
==========

[1. Prepare]          [2. Build]          [3. Validate]       [4. Promote]
     |                     |                    |                    |
  Pull Docker           Restore             Run legal-info       Tag image
  image             dl/ from cache         check                Upload to
  Checkout repo     Restore ccache         Run smoke tests      artifact store
  Select defconfig  make -j$(nproc)        Check image size     Notify teams
     |                     |                    |                    |
     +---------------------+--------------------+--------------------+
                           |
                    On any failure:
                    - Save logs as artifact
                    - Notify developer
                    - Block merge/promotion
```

The pipeline enforces a strict left-to-right dependency: no stage runs unless all prior
stages have succeeded. This prevents partially-built or license-non-compliant firmware from
ever reaching a downstream environment.

---

## Caching `dl/` and `ccache`

### Why Caching Matters

Buildroot downloads all package source tarballs into the `dl/` directory. A full build for
a moderately complex target (e.g., an i.MX8-based system with Qt, OpenSSL, Python, and a
custom application stack) can involve downloading several gigabytes. Without caching, every
CI run pays this cost in full — both in time and in bandwidth charges.

`ccache` (compiler cache) is equally critical: it intercepts compiler invocations and
replaces them with cache lookups when source files and flags are unchanged. On repeated or
incremental builds, this can reduce compilation time from 45 minutes to under 5 minutes.

### `dl/` Cache Strategy

```
dl/ directory structure inside CI runner:
==========================================

/cache/buildroot-dl/               <-- persistent across runs (NFS / S3-backed)
    ├── busybox-1.36.1.tar.bz2
    ├── linux-6.6.21.tar.xz
    ├── openssl-3.2.1.tar.gz
    ├── qt-everywhere-src-6.6.3.tar.xz
    ├── python3-3.12.2.tar.xz
    └── ...

Build mount:
    BR2_DL_DIR=/cache/buildroot-dl  <-- passed to make, or set in .config
```

The `dl/` directory is mounted read-write into the build container. Buildroot checks for
the presence of a file (and its hash) before downloading. If the file is present and valid,
the download is skipped entirely.

### `ccache` Cache Strategy

```
ccache directory layout:
=========================

/cache/ccache/                     <-- persistent across runs
    ├── a/
    │   └── b/
    │       └── <hash>             <-- cached compiler output
    ├── tmp/
    └── ccache.conf

Key settings in ccache.conf:
    max_size = 10G
    compression = true
    compression_level = 6
    hash_dir = false               <-- IMPORTANT for reproducibility
```

`hash_dir = false` is critical in CI: it prevents ccache from including the build directory
path in the cache key, allowing cache hits even when the runner uses a different working
directory on different machines.

### GitLab Cache Configuration (YAML)

```yaml
# .gitlab-ci.yml — cache configuration section
variables:
  BR2_DL_DIR: "/cache/buildroot-dl"
  CCACHE_DIR: "/cache/ccache"
  CCACHE_MAXSIZE: "10G"

cache:
  key: "buildroot-dl-${CI_COMMIT_REF_SLUG}"
  paths:
    - dl/
  policy: pull-push            # pull at start, push at end if changed

.build_cache: &build_cache
  cache:
    - key: "buildroot-dl-${CI_PROJECT_ID}"
      paths:
        - dl/
      policy: pull-push
    - key: "ccache-${CI_COMMIT_REF_SLUG}-${TARGET_DEFCONFIG}"
      paths:
        - .ccache/
      policy: pull-push
```

---

## Parallelism with `-j$(nproc)`

### Understanding Buildroot's Parallelism Model

Buildroot itself is single-threaded at the package level — packages are built sequentially
according to their dependency graph. However, within each package's build, the underlying
build system (Make, CMake, Autotools, Meson, etc.) can use multiple parallel jobs.

```
Buildroot package dependency graph (simplified):
=================================================

        [linux-headers]
              |
        [glibc]
         /         \
   [openssl]    [zlib]
        \         /
        [libcurl]
             |
        [your-app]          <-- all dependencies must complete before this builds

Within each package: make -j$(nproc)
Between packages:    sequential (dependency-ordered)
```

The `-j$(nproc)` flag is passed as `BR2_JLEVEL` in the Buildroot configuration or via the
`MAKEFLAGS` environment variable:

```makefile
# In Buildroot's perspective:
# BR2_JLEVEL=8 means: make -j8 inside each package
# The top-level make is NOT parallelised (intentionally)
```

### Optimal Job Count Formula

```
Recommended -j value:
======================

  Physical cores      -->  Good baseline (e.g., -j8 on 8-core machine)
  Logical cores       -->  Often better for mixed I/O + CPU workloads
  Logical cores + 2   -->  Can help when some jobs are I/O-bound
  nproc               -->  $(nproc) reads logical cores from /proc/cpuinfo

For a 16-core / 32-thread Xeon:
  Conservative: -j16
  Aggressive:   -j32
  Recommended:  -j$(nproc)   -- auto-adapts to the runner's hardware
```

### CI Job Configuration

```yaml
# GitHub Actions — using nproc for parallelism
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - name: Build Buildroot
        run: |
          NPROC=$(nproc)
          echo "Building with ${NPROC} parallel jobs"
          make -C buildroot \
            BR2_EXTERNAL=$PWD/external \
            O=$PWD/output \
            myboard_defconfig
          make -C buildroot \
            O=$PWD/output \
            BR2_JLEVEL=${NPROC} \
            2>&1 | tee build.log
```

---

## Artifact Promotion

### The Promotion Model

Artifact promotion is the practice of taking a validated build output and moving it through
a defined sequence of environments, each with increasing stability requirements. A build
artifact is promoted (never rebuilt) — this guarantees that what is tested is exactly what
is deployed.

```
ARTIFACT PROMOTION PIPELINE
=============================

  [Build]          [Dev]          [Staging]        [Production]
     |                |                |                  |
  sdcard.img       Deploy to       Deploy to          Release to
  rootfs.tar.gz    QEMU / dev      pre-prod HW        production
  zImage           boards          Run integration    Update OTA
  dtb files        Run unit        tests              server
     |              tests               |                  |
     |                |           Pass? Gate          Pass? Gate
     +----------------+----------------+------------------+
                      |
             Artifact Registry
             (GitLab Package Registry /
              GitHub Releases /
              S3 bucket / Artifactory)
```

### Version Tagging Strategy

```
Artifact naming convention:
============================

  firmware-<board>-<version>-<git-sha>-<build-num>.<ext>

Examples:
  firmware-imx8mp-1.4.2-a3f9c12-142.sdcard.img.gz
  firmware-rpi4-1.4.2-a3f9c12-142.rootfs.tar.gz
  firmware-stm32mp1-1.4.2-a3f9c12-142.rootfs.ext4.gz

Promotion tags:
  :dev        -- passed unit tests, deployed to dev boards
  :staging    -- passed integration tests
  :release    -- signed off by QA, ready for production
  :v1.4.2     -- final semantic version tag
```

### GitLab Artifact Upload

```yaml
# .gitlab-ci.yml — artifact publication stage
publish:artifacts:
  stage: promote
  needs:
    - job: validate:legal
    - job: test:smoke
  script:
    - |
      VERSION="${CI_COMMIT_TAG:-${CI_COMMIT_SHORT_SHA}}"
      ARTIFACT_NAME="firmware-${TARGET_BOARD}-${VERSION}-${CI_PIPELINE_IID}"

      # Compress artifacts
      gzip -9 output/images/sdcard.img
      gzip -9 output/images/rootfs.tar.gz

      # Upload to GitLab Package Registry
      curl --header "JOB-TOKEN: ${CI_JOB_TOKEN}" \
           --upload-file output/images/sdcard.img.gz \
           "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/firmware/${VERSION}/${ARTIFACT_NAME}.sdcard.img.gz"
  artifacts:
    name: "firmware-${CI_COMMIT_SHORT_SHA}"
    paths:
      - output/images/
    expire_in: 30 days
    when: on_success
```

---

## Docker-Based Build Containers

### Why Docker for Buildroot?

Buildroot's build output is highly sensitive to the host environment. Different versions of
GCC, glibc, make, python, or even bash can produce different outputs or outright build
failures. Docker containers provide:

- **Reproducibility**: Every runner uses the identical build environment
- **Isolation**: Host system libraries cannot interfere with the build
- **Portability**: The same container runs on developer laptops and CI runners
- **Auditability**: The Dockerfile is version-controlled alongside the firmware

```
Docker container internals for Buildroot builds:
=================================================

+----------------------------------------------------------+
|  Docker Container (buildroot-builder:2024.02)            |
|                                                          |
|  Base: ubuntu:22.04                                      |
|                                                          |
|  Installed tools:                                        |
|  - build-essential, gcc, g++, make, cmake                |
|  - git, wget, curl, rsync, cpio                          |
|  - python3, python3-pip                                  |
|  - ccache, file, bc, bison, flex, libssl-dev             |
|  - libncurses-dev, unzip, bzr, mercurial                 |
|  - graphviz, imagemagick (for legal-info)                |
|                                                          |
|  Volumes mounted at runtime:                             |
|  - /workspace        (source repo, read-write)           |
|  - /cache/dl         (download cache, read-write)        |
|  - /cache/ccache     (compiler cache, read-write)        |
|  - /output           (build output, read-write)          |
+----------------------------------------------------------+
```

### Dockerfile for Buildroot Builds

```dockerfile
# Dockerfile.buildroot-builder
FROM ubuntu:22.04

LABEL maintainer="firmware-team@example.com"
LABEL description="Buildroot CI build environment"
LABEL version="2024.02.1"

# Prevent interactive prompts
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=UTC

# Install Buildroot host dependencies (from Buildroot manual, Chapter 2)
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Essential build tools
    build-essential \
    gcc \
    g++ \
    make \
    cmake \
    ninja-build \
    pkg-config \
    # Version control
    git \
    mercurial \
    subversion \
    bzr \
    # Download utilities
    wget \
    curl \
    rsync \
    # Compression tools
    gzip \
    bzip2 \
    xz-utils \
    zip \
    unzip \
    # Build support tools
    bison \
    flex \
    bc \
    cpio \
    file \
    # Python (required by many packages)
    python3 \
    python3-pip \
    python3-setuptools \
    # SSL/TLS development
    libssl-dev \
    # NCurses (for menuconfig)
    libncurses-dev \
    # ccache for compiler caching
    ccache \
    # Graphviz (for make graph-depends)
    graphviz \
    # Legal tools
    licensecheck \
    # Additional utilities
    locales \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Set locale
RUN locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8

# Configure ccache
RUN ccache --set-config=hash_dir=false \
    && ccache --set-config=compression=true \
    && ccache --set-config=compression_level=6 \
    && ccache --set-config=max_size=10G

# Create non-root build user (Buildroot refuses to run as root)
RUN useradd -m -u 1000 -s /bin/bash builder
USER builder
WORKDIR /workspace

# Verify build environment
RUN make --version && gcc --version && python3 --version

CMD ["/bin/bash"]
```

### Docker in GitLab CI

```yaml
# .gitlab-ci.yml — using custom Docker image
default:
  image: registry.example.com/firmware/buildroot-builder:2024.02.1
  before_script:
    - export PATH="/usr/lib/ccache:$PATH"
    - ccache --zero-stats
```

---

## `make legal-info` Gating

### What `make legal-info` Produces

The `make legal-info` target scans every package included in the Buildroot configuration
and generates a comprehensive legal compliance report. This is a mandatory step for any
commercial product shipping embedded Linux.

```
legal-info output directory structure:
========================================

output/legal-info/
    ├── manifest.csv              <-- machine-readable package list with licenses
    ├── licenses/                 <-- full text of all licenses
    │   ├── linux/
    │   │   └── COPYING
    │   ├── busybox/
    │   │   └── LICENSE
    │   ├── openssl/
    │   │   └── LICENSE.txt
    │   └── ...
    ├── sources/                  <-- source archives for GPL compliance
    │   ├── busybox-1.36.1.tar.bz2
    │   ├── linux-6.6.21.tar.xz
    │   └── ...
    ├── host-manifest.csv         <-- host-side tools (usually not shipped)
    └── README
```

### manifest.csv Format

```
manifest.csv columns:
======================

PACKAGE | VERSION | LICENSE | LICENSE FILES | SOURCE ARCHIVE | PATCHES

Example rows:
busybox,1.36.1,GPL-2.0+,COPYING,busybox-1.36.1.tar.bz2,
openssl,3.2.1,Apache-2.0,LICENSE.txt,,
zlib,1.3.1,Zlib,README,,
libpng,1.6.43,libpng-2.0,LICENSE,,
```

### License Compliance Gate

The gate works by parsing `manifest.csv` and failing the pipeline if any unapproved license
is detected. Teams maintain an allowlist of approved licenses and an explicit denylist of
incompatible licenses (e.g., GPL-3.0 in certain commercial contexts).

```
License gating logic:
======================

  Parse manifest.csv
         |
         v
  For each package:
  +---------------------+
  | License in ALLOW?   |--YES--> PASS (continue)
  +---------------------+
         |
         NO
         v
  +---------------------+
  | License in DENY?    |--YES--> FAIL (block pipeline)
  +---------------------+
         |
         NO
         v
  +---------------------+
  | License UNKNOWN?    |--YES--> WARN (notify legal team)
  +---------------------+
         |
         NO
         v
  Escalate to legal review
```

### `make legal-info` in CI

```yaml
# .gitlab-ci.yml — legal gating stage
validate:legal:
  stage: validate
  needs:
    - job: build:firmware
      artifacts: true
  script:
    - make -C buildroot O=$PWD/output legal-info
    - python3 ci/scripts/check_licenses.py \
        --manifest output/legal-info/manifest.csv \
        --allowlist ci/config/allowed_licenses.txt \
        --denylist ci/config/denied_licenses.txt \
        --report output/legal-info/compliance-report.txt
  artifacts:
    name: "legal-info-${CI_COMMIT_SHORT_SHA}"
    paths:
      - output/legal-info/
    expire_in: 90 days    # Keep for audit purposes
    when: always          # Keep even on failure for debugging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    - if: '$CI_MERGE_REQUEST_ID'
      when: always
```

---

## GitLab CI Configuration

### Complete `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml
# Buildroot CI/CD Pipeline — Full Configuration

variables:
  # Build configuration
  TARGET_BOARD: "myboard"
  TARGET_DEFCONFIG: "myboard_defconfig"
  BUILDROOT_VERSION: "2024.02.3"

  # Cache paths (backed by runner NFS or S3)
  BR2_DL_DIR: "/cache/buildroot-dl"
  CCACHE_DIR: "/cache/ccache/${CI_PROJECT_ID}"
  CCACHE_MAXSIZE: "10G"
  CCACHE_COMPRESS: "true"

  # Output configuration
  OUTPUT_DIR: "$CI_PROJECT_DIR/output"

  # Docker registry
  BUILD_IMAGE: "registry.example.com/firmware/buildroot-builder:2024.02.1"

stages:
  - prepare
  - build
  - validate
  - test
  - promote

# ─────────────────────────────────────────────
# STAGE 1: PREPARE
# ─────────────────────────────────────────────
prepare:environment:
  stage: prepare
  image: $BUILD_IMAGE
  script:
    - echo "Runner: $(hostname), CPUs: $(nproc), RAM: $(free -h | awk '/^Mem/{print $2}')"
    - make --version
    - gcc --version
    - ccache --version
    - ccache --show-config
  rules:
    - when: always

# ─────────────────────────────────────────────
# STAGE 2: BUILD
# ─────────────────────────────────────────────
build:firmware:
  stage: build
  image: $BUILD_IMAGE
  timeout: 4 hours
  variables:
    GIT_STRATEGY: fetch
    GIT_DEPTH: 10
  cache:
    - key: "dl-${CI_PROJECT_ID}-${BUILDROOT_VERSION}"
      paths:
        - dl/
      policy: pull-push
    - key: "ccache-${CI_PROJECT_ID}-${TARGET_DEFCONFIG}-${CI_COMMIT_REF_SLUG}"
      paths:
        - .ccache/
      policy: pull-push
  before_script:
    - export PATH="/usr/lib/ccache:$PATH"
    - export CCACHE_DIR="$CI_PROJECT_DIR/.ccache"
    - ccache --zero-stats
    - mkdir -p "$OUTPUT_DIR"
  script:
    # Configure
    - make -C buildroot \
        BR2_EXTERNAL="$CI_PROJECT_DIR/external" \
        O="$OUTPUT_DIR" \
        "${TARGET_DEFCONFIG}"

    # Build with parallel jobs
    - |
      NPROC=$(nproc)
      echo "Building with ${NPROC} parallel jobs"
      make -C buildroot \
        O="$OUTPUT_DIR" \
        BR2_DL_DIR="$BR2_DL_DIR" \
        BR2_CCACHE=y \
        BR2_CCACHE_DIR="$CCACHE_DIR" \
        BR2_JLEVEL="${NPROC}" \
        2>&1 | tee "$OUTPUT_DIR/build.log"

    # Show ccache statistics
    - ccache --show-stats

    # Record artifact checksums
    - sha256sum output/images/* > output/images/SHA256SUMS
  artifacts:
    name: "firmware-${TARGET_BOARD}-${CI_COMMIT_SHORT_SHA}"
    paths:
      - output/images/
      - output/build/build-time.log
    expire_in: 7 days
    when: on_success
  after_script:
    - "[ -f output/build.log ] && tail -50 output/build.log || true"

# ─────────────────────────────────────────────
# STAGE 3: VALIDATE
# ─────────────────────────────────────────────
validate:legal:
  stage: validate
  image: $BUILD_IMAGE
  needs:
    - job: build:firmware
      artifacts: true
  script:
    - make -C buildroot O="$OUTPUT_DIR" legal-info
    - python3 ci/scripts/check_licenses.py \
        --manifest output/legal-info/manifest.csv \
        --allowlist ci/config/allowed_licenses.txt \
        --denylist ci/config/denied_licenses.txt
  artifacts:
    paths:
      - output/legal-info/
    expire_in: 90 days
    when: always

validate:image-size:
  stage: validate
  needs:
    - job: build:firmware
      artifacts: true
  script:
    - python3 ci/scripts/check_image_size.py \
        --image output/images/sdcard.img \
        --max-size-mb 512

# ─────────────────────────────────────────────
# STAGE 4: TEST
# ─────────────────────────────────────────────
test:qemu-smoke:
  stage: test
  image: $BUILD_IMAGE
  needs:
    - job: build:firmware
      artifacts: true
    - job: validate:legal
  script:
    - python3 ci/scripts/qemu_smoke_test.py \
        --kernel output/images/zImage \
        --dtb output/images/myboard.dtb \
        --rootfs output/images/rootfs.ext4 \
        --timeout 120
  artifacts:
    paths:
      - output/test-results/
    reports:
      junit: output/test-results/junit.xml

# ─────────────────────────────────────────────
# STAGE 5: PROMOTE
# ─────────────────────────────────────────────
promote:release:
  stage: promote
  image: $BUILD_IMAGE
  needs:
    - job: test:qemu-smoke
    - job: validate:legal
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/'
      when: on_success
  script:
    - VERSION="${CI_COMMIT_TAG}"
    - |
      for artifact in output/images/*.img output/images/*.tar.gz; do
        gzip -9 "$artifact" || true
        fname=$(basename "${artifact}.gz")
        curl --fail --header "JOB-TOKEN: ${CI_JOB_TOKEN}" \
             --upload-file "${artifact}.gz" \
             "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/firmware/${VERSION}/${fname}"
      done
```

---

## GitHub Actions Configuration

### Complete `build.yml`

```yaml
# .github/workflows/build.yml
name: Buildroot Firmware Build

on:
  push:
    branches: [main, develop]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

env:
  BUILDROOT_VERSION: "2024.02.3"
  TARGET_BOARD: "myboard"
  TARGET_DEFCONFIG: "myboard_defconfig"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/buildroot-builder

jobs:
  # ─────────────────────────────────────────────
  # JOB 1: BUILD
  # ─────────────────────────────────────────────
  build:
    name: Build Firmware (${{ matrix.board }})
    runs-on: ubuntu-24.04
    container:
      image: ghcr.io/${{ github.repository }}/buildroot-builder:2024.02.1
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    timeout-minutes: 240

    strategy:
      matrix:
        board: [myboard, myboard-debug]
      fail-fast: false

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 10
          submodules: recursive

      - name: Restore dl/ cache
        uses: actions/cache@v4
        with:
          path: dl/
          key: buildroot-dl-${{ env.BUILDROOT_VERSION }}-${{ hashFiles('configs/**') }}
          restore-keys: |
            buildroot-dl-${{ env.BUILDROOT_VERSION }}-
            buildroot-dl-

      - name: Restore ccache
        uses: actions/cache@v4
        with:
          path: .ccache/
          key: ccache-${{ matrix.board }}-${{ github.ref_name }}-${{ github.sha }}
          restore-keys: |
            ccache-${{ matrix.board }}-${{ github.ref_name }}-
            ccache-${{ matrix.board }}-

      - name: Configure ccache
        run: |
          export CCACHE_DIR=$GITHUB_WORKSPACE/.ccache
          ccache --set-config=hash_dir=false
          ccache --set-config=compression=true
          ccache --set-config=max_size=10G
          ccache --zero-stats

      - name: Configure Buildroot
        run: |
          make -C buildroot \
            BR2_EXTERNAL=$GITHUB_WORKSPACE/external \
            O=$GITHUB_WORKSPACE/output \
            ${{ matrix.board }}_defconfig

      - name: Build firmware
        run: |
          NPROC=$(nproc)
          echo "::notice::Building with ${NPROC} parallel jobs"
          make -C buildroot \
            O=$GITHUB_WORKSPACE/output \
            BR2_DL_DIR=$GITHUB_WORKSPACE/dl \
            BR2_CCACHE=y \
            BR2_CCACHE_DIR=$GITHUB_WORKSPACE/.ccache \
            BR2_JLEVEL=${NPROC} \
            2>&1 | tee output/build.log
          echo "Build exit code: $?"

      - name: Display ccache statistics
        if: always()
        run: ccache --show-stats

      - name: Generate checksums
        run: sha256sum output/images/* > output/images/SHA256SUMS

      - name: Upload firmware artifacts
        uses: actions/upload-artifact@v4
        with:
          name: firmware-${{ matrix.board }}-${{ github.sha }}
          path: |
            output/images/
            output/build/build-time.log
          retention-days: 30

  # ─────────────────────────────────────────────
  # JOB 2: LEGAL VALIDATION
  # ─────────────────────────────────────────────
  legal-check:
    name: License Compliance Check
    runs-on: ubuntu-24.04
    needs: build
    container:
      image: ghcr.io/${{ github.repository }}/buildroot-builder:2024.02.1
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    steps:
      - uses: actions/checkout@v4

      - name: Download firmware artifacts
        uses: actions/download-artifact@v4
        with:
          name: firmware-myboard-${{ github.sha }}
          path: output/

      - name: Run make legal-info
        run: |
          make -C buildroot \
            O=$GITHUB_WORKSPACE/output \
            legal-info

      - name: Check license compliance
        run: |
          python3 ci/scripts/check_licenses.py \
            --manifest output/legal-info/manifest.csv \
            --allowlist ci/config/allowed_licenses.txt \
            --denylist ci/config/denied_licenses.txt \
            --report output/legal-info/compliance-report.txt

      - name: Upload legal-info
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: legal-info-${{ github.sha }}
          path: output/legal-info/
          retention-days: 90

  # ─────────────────────────────────────────────
  # JOB 3: PROMOTE ON TAG
  # ─────────────────────────────────────────────
  release:
    name: Publish Release
    runs-on: ubuntu-24.04
    needs: [build, legal-check]
    if: startsWith(github.ref, 'refs/tags/v')
    permissions:
      contents: write

    steps:
      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          pattern: firmware-*-${{ github.sha }}
          merge-multiple: true
          path: release-artifacts/

      - name: Compress artifacts
        run: |
          cd release-artifacts/images
          gzip -9 *.img *.ext4 || true
          gzip -9 *.tar.gz || true

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: release-artifacts/images/*
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref_name, '-rc') }}
```

---

## C Code Examples

### License Compliance Checker (C)

This utility parses the `manifest.csv` from `make legal-info` and exits with a non-zero
status if any disallowed license is found. It is suitable for use in shell-scripted CI
gates.

```c
/*
 * license_check.c
 * Buildroot CI/CD license compliance gate.
 *
 * Usage: ./license_check manifest.csv allowlist.txt denylist.txt
 * Exit:  0 = compliant, 1 = violation found, 2 = error
 *
 * Compile: gcc -O2 -Wall -o license_check license_check.c
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define MAX_LINE        1024
#define MAX_LICENSES    256
#define MAX_PKG_NAME    128
#define MAX_LICENSE_STR 256

typedef struct {
    char package[MAX_PKG_NAME];
    char version[64];
    char license[MAX_LICENSE_STR];
} PackageEntry;

/* Load newline-separated license names from a file into an array */
static int load_list(const char *path, char list[][MAX_LICENSE_STR], int max)
{
    FILE *f = fopen(path, "r");
    if (!f) {
        fprintf(stderr, "Error opening %s: %s\n", path, strerror(errno));
        return -1;
    }

    int count = 0;
    char line[MAX_LINE];
    while (count < max && fgets(line, sizeof(line), f)) {
        /* Strip trailing newline and whitespace */
        size_t len = strlen(line);
        while (len > 0 && (line[len-1] == '\n' || line[len-1] == '\r'
                           || line[len-1] == ' ' || line[len-1] == '\t'))
            line[--len] = '\0';

        /* Skip blank lines and comments */
        if (len == 0 || line[0] == '#')
            continue;

        strncpy(list[count++], line, MAX_LICENSE_STR - 1);
    }

    fclose(f);
    return count;
}

static int list_contains(char list[][MAX_LICENSE_STR], int count, const char *license)
{
    for (int i = 0; i < count; i++) {
        if (strcmp(list[i], license) == 0)
            return 1;
    }
    return 0;
}

/*
 * Parse a single CSV field, handling quoted strings.
 * Returns pointer to next field or NULL at end-of-line.
 */
static char *parse_csv_field(char *src, char *dst, size_t dstlen)
{
    size_t i = 0;
    if (*src == '"') {
        src++;
        while (*src && *src != '"' && i < dstlen - 1)
            dst[i++] = *src++;
        if (*src == '"') src++;
        if (*src == ',') src++;
    } else {
        while (*src && *src != ',' && *src != '\n' && *src != '\r' && i < dstlen - 1)
            dst[i++] = *src++;
        if (*src == ',') src++;
    }
    dst[i] = '\0';
    return *src ? src : NULL;
}

int main(int argc, char *argv[])
{
    if (argc < 4) {
        fprintf(stderr, "Usage: %s manifest.csv allowlist.txt denylist.txt\n", argv[0]);
        return 2;
    }

    /* Load license lists */
    static char allowlist[MAX_LICENSES][MAX_LICENSE_STR];
    static char denylist[MAX_LICENSES][MAX_LICENSE_STR];

    int allow_count = load_list(argv[2], allowlist, MAX_LICENSES);
    int deny_count  = load_list(argv[3], denylist,  MAX_LICENSES);

    if (allow_count < 0 || deny_count < 0)
        return 2;

    printf("Loaded %d allowed licenses, %d denied licenses\n",
           allow_count, deny_count);

    /* Parse manifest.csv */
    FILE *f = fopen(argv[1], "r");
    if (!f) {
        fprintf(stderr, "Error opening manifest: %s\n", strerror(errno));
        return 2;
    }

    int violations = 0;
    int warnings   = 0;
    int checked    = 0;
    char line[MAX_LINE];

    /* Skip header row */
    fgets(line, sizeof(line), f);

    while (fgets(line, sizeof(line), f)) {
        PackageEntry pkg = {0};
        char *p = line;

        p = parse_csv_field(p, pkg.package, sizeof(pkg.package));
        if (!p) continue;
        p = parse_csv_field(p, pkg.version, sizeof(pkg.version));
        if (!p) continue;
        parse_csv_field(p, pkg.license, sizeof(pkg.license));

        checked++;

        if (list_contains(denylist, deny_count, pkg.license)) {
            printf("[DENIED]  %-40s %-20s  %s\n",
                   pkg.package, pkg.version, pkg.license);
            violations++;
        } else if (!list_contains(allowlist, allow_count, pkg.license)) {
            printf("[UNKNOWN] %-40s %-20s  %s\n",
                   pkg.package, pkg.version, pkg.license);
            warnings++;
        } else {
            printf("[OK]      %-40s %-20s  %s\n",
                   pkg.package, pkg.version, pkg.license);
        }
    }

    fclose(f);

    printf("\n");
    printf("Packages checked : %d\n", checked);
    printf("Violations (DENY): %d\n", violations);
    printf("Warnings (unknown): %d\n", warnings);

    if (violations > 0) {
        printf("\nResult: FAIL — license violations detected\n");
        return 1;
    }

    printf("\nResult: PASS — all licenses compliant\n");
    return warnings > 0 ? 1 : 0;
}
```

### Build Statistics Aggregator (C)

```c
/*
 * build_stats.c
 * Parses Buildroot's build-time.log and reports per-package build times.
 * Useful for identifying bottlenecks in the CI pipeline.
 *
 * Compile: gcc -O2 -Wall -o build_stats build_stats.c -lm
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>

#define MAX_PACKAGES 512
#define MAX_NAME     128

typedef struct {
    char  name[MAX_NAME];
    float duration_seconds;
} BuildEntry;

static int cmp_duration(const void *a, const void *b)
{
    const BuildEntry *ea = (const BuildEntry *)a;
    const BuildEntry *eb = (const BuildEntry *)b;
    if (eb->duration_seconds > ea->duration_seconds) return  1;
    if (eb->duration_seconds < ea->duration_seconds) return -1;
    return 0;
}

int main(int argc, char *argv[])
{
    const char *logfile = argc > 1 ? argv[1] : "output/build/build-time.log";

    FILE *f = fopen(logfile, "r");
    if (!f) {
        perror("fopen");
        return 1;
    }

    static BuildEntry entries[MAX_PACKAGES];
    int count = 0;
    float total = 0.0f;

    char line[512];
    while (count < MAX_PACKAGES && fgets(line, sizeof(line), f)) {
        float secs;
        char name[MAX_NAME];
        /* Format: "build/package-1.2.3   45.2" */
        if (sscanf(line, "%127s %f", name, &secs) == 2) {
            strncpy(entries[count].name, name, MAX_NAME - 1);
            entries[count].duration_seconds = secs;
            total += secs;
            count++;
        }
    }
    fclose(f);

    qsort(entries, count, sizeof(BuildEntry), cmp_duration);

    printf("%-60s %8s %6s\n", "Package", "Seconds", "% tot");
    printf("%-60s %8s %6s\n",
           "------------------------------------------------------------",
           "--------", "------");

    for (int i = 0; i < count; i++) {
        float pct = total > 0.0f ? (entries[i].duration_seconds / total) * 100.0f : 0.0f;
        printf("%-60s %8.1f %5.1f%%\n",
               entries[i].name, entries[i].duration_seconds, pct);
    }

    printf("\n");
    printf("Total packages : %d\n", count);
    printf("Total build time: %.0f seconds (%.1f minutes)\n", total, total / 60.0f);

    return 0;
}
```

---

## C++ Code Examples

### Artifact Integrity Verifier (C++)

```cpp
/*
 * artifact_verify.cpp
 * Verifies Buildroot firmware artifacts against a SHA256SUMS file.
 * Used in the CI promotion stage to ensure artifact integrity.
 *
 * Compile: g++ -std=c++17 -O2 -Wall -o artifact_verify artifact_verify.cpp
 */

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <vector>
#include <filesystem>
#include <iomanip>
#include <stdexcept>

namespace fs = std::filesystem;

// Simple SHA256 interface — in production, link against OpenSSL or libsodium
// Here we shell out to sha256sum for portability in CI containers
static std::string compute_sha256(const fs::path &file)
{
    std::string cmd = "sha256sum \"" + file.string() + "\" 2>/dev/null";
    FILE *pipe = popen(cmd.c_str(), "r");
    if (!pipe)
        throw std::runtime_error("popen failed for sha256sum");

    char buffer[256] = {0};
    std::string result;
    while (fgets(buffer, sizeof(buffer), pipe))
        result += buffer;
    pclose(pipe);

    // sha256sum output: "<hash>  <filename>"
    std::istringstream iss(result);
    std::string hash;
    iss >> hash;
    return hash;
}

struct VerificationResult {
    std::string filename;
    std::string expected;
    std::string actual;
    bool        match;
};

class ArtifactVerifier {
public:
    explicit ArtifactVerifier(const fs::path &checksums_file)
        : checksums_path_(checksums_file)
    {
        load_checksums();
    }

    std::vector<VerificationResult> verify(const fs::path &dir)
    {
        std::vector<VerificationResult> results;

        for (const auto &[filename, expected_hash] : expected_) {
            fs::path artifact = dir / filename;
            VerificationResult r;
            r.filename = filename;
            r.expected = expected_hash;

            if (!fs::exists(artifact)) {
                r.actual = "<MISSING>";
                r.match  = false;
            } else {
                r.actual = compute_sha256(artifact);
                r.match  = (r.actual == r.expected);
            }

            results.push_back(r);
        }

        return results;
    }

    void print_report(const std::vector<VerificationResult> &results) const
    {
        int pass = 0, fail = 0;
        const int w = 45;

        std::cout << std::string(90, '=') << "\n";
        std::cout << "ARTIFACT INTEGRITY REPORT\n";
        std::cout << std::string(90, '=') << "\n\n";

        for (const auto &r : results) {
            std::string status = r.match ? "[PASS]" : "[FAIL]";
            std::cout << status << "  "
                      << std::left << std::setw(w) << r.filename << "\n";
            if (!r.match) {
                std::cout << "       Expected: " << r.expected << "\n";
                std::cout << "       Actual:   " << r.actual   << "\n";
                fail++;
            } else {
                pass++;
            }
        }

        std::cout << "\n";
        std::cout << "Passed: " << pass << "\n";
        std::cout << "Failed: " << fail << "\n";
        std::cout << "\nResult: " << (fail == 0 ? "PASS" : "FAIL") << "\n";
    }

    bool all_pass(const std::vector<VerificationResult> &results) const
    {
        for (const auto &r : results)
            if (!r.match) return false;
        return true;
    }

private:
    void load_checksums()
    {
        std::ifstream f(checksums_path_);
        if (!f.is_open())
            throw std::runtime_error("Cannot open checksums file: " + checksums_path_.string());

        std::string line;
        while (std::getline(f, line)) {
            if (line.empty() || line[0] == '#') continue;
            std::istringstream iss(line);
            std::string hash, filename;
            if (iss >> hash >> filename)
                expected_[filename] = hash;
        }
    }

    fs::path                               checksums_path_;
    std::unordered_map<std::string, std::string> expected_;
};

int main(int argc, char *argv[])
{
    if (argc < 3) {
        std::cerr << "Usage: " << argv[0] << " <SHA256SUMS> <artifacts-dir>\n";
        return 2;
    }

    try {
        ArtifactVerifier verifier(argv[1]);
        auto results = verifier.verify(argv[2]);
        verifier.print_report(results);
        return verifier.all_pass(results) ? 0 : 1;
    } catch (const std::exception &e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 2;
    }
}
```

### Pipeline Configuration Parser (C++)

```cpp
/*
 * pipeline_config.cpp
 * Reads a simple INI-style pipeline configuration file used to drive
 * multi-board Buildroot CI builds.
 *
 * Compile: g++ -std=c++17 -O2 -Wall -o pipeline_config pipeline_config.cpp
 */

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <vector>
#include <unordered_map>
#include <algorithm>
#include <cctype>

struct BoardConfig {
    std::string name;
    std::string defconfig;
    std::string arch;
    int         parallel_jobs;
    bool        run_legal_check;
    std::vector<std::string> extra_make_flags;
};

class PipelineConfig {
public:
    bool load(const std::string &path)
    {
        std::ifstream f(path);
        if (!f.is_open()) {
            std::cerr << "Cannot open config: " << path << "\n";
            return false;
        }

        std::string line;
        BoardConfig *current = nullptr;

        while (std::getline(f, line)) {
            trim(line);
            if (line.empty() || line[0] == ';' || line[0] == '#')
                continue;

            if (line.front() == '[' && line.back() == ']') {
                std::string section = line.substr(1, line.size() - 2);
                boards_.emplace_back();
                current = &boards_.back();
                current->name = section;
                current->parallel_jobs = 0;  // 0 = auto (nproc)
                current->run_legal_check = true;
            } else if (current) {
                auto eq = line.find('=');
                if (eq == std::string::npos) continue;
                std::string key = line.substr(0, eq);
                std::string val = line.substr(eq + 1);
                trim(key); trim(val);
                apply_key_value(*current, key, val);
            }
        }

        return true;
    }

    void print_summary() const
    {
        std::cout << "\n";
        std::cout << "+-------------------------------+------------------+--------+---------+--------+\n";
        std::cout << "| Board                         | Defconfig        | Arch   | Jobs    | Legal  |\n";
        std::cout << "+-------------------------------+------------------+--------+---------+--------+\n";

        for (const auto &b : boards_) {
            std::string jobs = b.parallel_jobs == 0 ? "auto" : std::to_string(b.parallel_jobs);
            std::string legal = b.run_legal_check ? "yes" : "no";
            std::cout << "| "
                      << padRight(b.name,      30) << "| "
                      << padRight(b.defconfig, 17) << "| "
                      << padRight(b.arch,       7) << "| "
                      << padRight(jobs,         8) << "| "
                      << padRight(legal,        7) << "|\n";
        }

        std::cout << "+-------------------------------+------------------+--------+---------+--------+\n";
        std::cout << "Total boards configured: " << boards_.size() << "\n\n";
    }

    const std::vector<BoardConfig> &boards() const { return boards_; }

private:
    static void trim(std::string &s)
    {
        auto f = [](unsigned char c){ return std::isspace(c); };
        s.erase(s.begin(), std::find_if_not(s.begin(), s.end(), f));
        s.erase(std::find_if_not(s.rbegin(), s.rend(), f).base(), s.end());
    }

    static std::string padRight(const std::string &s, size_t width)
    {
        if (s.size() >= width) return s.substr(0, width);
        return s + std::string(width - s.size(), ' ');
    }

    void apply_key_value(BoardConfig &b, const std::string &key, const std::string &val)
    {
        if      (key == "defconfig")       b.defconfig      = val;
        else if (key == "arch")            b.arch           = val;
        else if (key == "parallel_jobs")   b.parallel_jobs  = std::stoi(val);
        else if (key == "run_legal_check") b.run_legal_check= (val == "true" || val == "1");
        else if (key == "extra_flags")     b.extra_make_flags.push_back(val);
    }

    std::vector<BoardConfig> boards_;
};

int main(int argc, char *argv[])
{
    const char *cfg = argc > 1 ? argv[1] : "ci/config/boards.ini";

    PipelineConfig config;
    if (!config.load(cfg))
        return 1;

    config.print_summary();

    // Generate make commands for each board
    for (const auto &board : config.boards()) {
        int jobs = board.parallel_jobs > 0 ? board.parallel_jobs : 0;
        std::ostringstream cmd;
        cmd << "make -C buildroot O=output/" << board.name
            << " " << board.defconfig
            << (jobs > 0 ? " BR2_JLEVEL=" + std::to_string(jobs) : " BR2_JLEVEL=$(nproc)");
        for (const auto &flag : board.extra_make_flags)
            cmd << " " << flag;
        std::cout << "Build command [" << board.name << "]: " << cmd.str() << "\n";
    }

    return 0;
}
```

---

## Rust Code Examples

### CI Pipeline Health Monitor (Rust)

```rust
//! ci_monitor.rs
//! Monitors a Buildroot CI pipeline by polling the GitLab/GitHub API
//! and reporting job status, build times, and cache hit rates.
//!
//! [dependencies]
//! serde = { version = "1", features = ["derive"] }
//! serde_json = "1"
//! reqwest = { version = "0.12", features = ["blocking", "json"] }
//! chrono = { version = "0.4", features = ["serde"] }

use std::collections::HashMap;
use std::env;
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
enum JobStatus {
    Pending,
    Running,
    Passed,
    Failed,
    Cancelled,
    Unknown(String),
}

impl fmt::Display for JobStatus {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            JobStatus::Pending          => write!(f, "PENDING  "),
            JobStatus::Running          => write!(f, "RUNNING  "),
            JobStatus::Passed           => write!(f, "PASSED   "),
            JobStatus::Failed           => write!(f, "FAILED   "),
            JobStatus::Cancelled        => write!(f, "CANCELLED"),
            JobStatus::Unknown(s)       => write!(f, "UNKNOWN({s})"),
        }
    }
}

impl From<&str> for JobStatus {
    fn from(s: &str) -> Self {
        match s {
            "pending"  | "waiting_for_resource" | "scheduled" => JobStatus::Pending,
            "running"  | "preparing"                           => JobStatus::Running,
            "success"  | "passed"                              => JobStatus::Passed,
            "failed"                                           => JobStatus::Failed,
            "canceled" | "cancelled" | "skipped"               => JobStatus::Cancelled,
            other                                              => JobStatus::Unknown(other.to_string()),
        }
    }
}

#[derive(Debug)]
struct JobReport {
    name:           String,
    stage:          String,
    status:         JobStatus,
    duration_secs:  Option<f64>,
    runner:         Option<String>,
}

#[derive(Debug)]
struct PipelineReport {
    pipeline_id:    u64,
    project:        String,
    branch:         String,
    commit_sha:     String,
    jobs:           Vec<JobReport>,
    total_duration: f64,
}

impl PipelineReport {
    fn print_ascii_report(&self) {
        let sep = "=".repeat(78);
        let dash = "-".repeat(78);

        println!("{sep}");
        println!("  BUILDROOT CI PIPELINE REPORT");
        println!("  Pipeline #{} | {} | {}",
                 self.pipeline_id, self.branch,
                 &self.commit_sha[..8.min(self.commit_sha.len())]);
        println!("{sep}");

        // Stage grouping
        let mut by_stage: HashMap<&str, Vec<&JobReport>> = HashMap::new();
        for job in &self.jobs {
            by_stage.entry(job.stage.as_str()).or_default().push(job);
        }

        let stages = ["prepare", "build", "validate", "test", "promote"];
        for stage in &stages {
            if let Some(jobs) = by_stage.get(stage) {
                println!("\n  STAGE: {}", stage.to_uppercase());
                println!("  {dash}");
                for job in jobs {
                    let dur = job.duration_secs
                        .map(|d| format!("{:6.1}s", d))
                        .unwrap_or_else(|| "    --".to_string());
                    let runner = job.runner.as_deref().unwrap_or("unknown");
                    println!("  [{status}]  {name:<38} {dur}  [{runner}]",
                             status = job.status,
                             name = job.name,
                             dur = dur,
                             runner = runner);
                }
            }
        }

        println!("\n{sep}");
        println!("  Total pipeline duration: {:.1}s ({:.1} min)",
                 self.total_duration,
                 self.total_duration / 60.0);

        // Overall status
        let all_passed = self.jobs.iter().all(|j| j.status == JobStatus::Passed);
        let any_failed = self.jobs.iter().any(|j| j.status == JobStatus::Failed);

        let verdict = if all_passed {
            "PIPELINE: PASSED"
        } else if any_failed {
            "PIPELINE: FAILED"
        } else {
            "PIPELINE: IN PROGRESS"
        };

        println!("  {verdict}");
        println!("{sep}");
    }

    fn print_ascii_bar_chart(&self) {
        println!("\n  BUILD TIME BREAKDOWN (by job)");
        println!("  {}", "-".repeat(60));

        let max_dur = self.jobs.iter()
            .filter_map(|j| j.duration_secs)
            .fold(0.0_f64, f64::max);

        if max_dur < 1.0 { return; }

        for job in &self.jobs {
            if let Some(dur) = job.duration_secs {
                let bar_len = ((dur / max_dur) * 40.0) as usize;
                let bar = "#".repeat(bar_len);
                println!("  {name:<35} |{bar:<40}| {dur:.1}s",
                         name = &job.name[..35.min(job.name.len())],
                         bar = bar,
                         dur = dur);
            }
        }
        println!();
    }
}

fn make_demo_report() -> PipelineReport {
    PipelineReport {
        pipeline_id:   142857,
        project:       "firmware/myboard".to_string(),
        branch:        "main".to_string(),
        commit_sha:    "a3f9c128deadbeef".to_string(),
        total_duration: 2847.3,
        jobs: vec![
            JobReport { name: "prepare:environment".into(), stage: "prepare".into(),
                        status: JobStatus::Passed, duration_secs: Some(12.4),
                        runner: Some("runner-01".into()) },
            JobReport { name: "build:firmware:myboard".into(), stage: "build".into(),
                        status: JobStatus::Passed, duration_secs: Some(2340.1),
                        runner: Some("runner-02".into()) },
            JobReport { name: "build:firmware:myboard-debug".into(), stage: "build".into(),
                        status: JobStatus::Passed, duration_secs: Some(2298.7),
                        runner: Some("runner-03".into()) },
            JobReport { name: "validate:legal".into(), stage: "validate".into(),
                        status: JobStatus::Passed, duration_secs: Some(145.2),
                        runner: Some("runner-01".into()) },
            JobReport { name: "validate:image-size".into(), stage: "validate".into(),
                        status: JobStatus::Passed, duration_secs: Some(3.1),
                        runner: Some("runner-01".into()) },
            JobReport { name: "test:qemu-smoke".into(), stage: "test".into(),
                        status: JobStatus::Passed, duration_secs: Some(78.9),
                        runner: Some("runner-04".into()) },
            JobReport { name: "promote:release".into(), stage: "promote".into(),
                        status: JobStatus::Passed, duration_secs: Some(24.6),
                        runner: Some("runner-01".into()) },
        ],
    }
}

fn main() {
    let report = make_demo_report();
    report.print_ascii_report();
    report.print_ascii_bar_chart();

    // Exit code reflects pipeline status
    let any_failed = report.jobs.iter().any(|j| j.status == JobStatus::Failed);
    if any_failed {
        std::process::exit(1);
    }
}
```

### `dl/` Cache Manager (Rust)

```rust
//! cache_manager.rs
//! Manages the Buildroot dl/ download cache:
//! - Lists cached packages with sizes
//! - Validates file hashes against Buildroot .hash files
//! - Prunes entries older than a configurable number of days
//! - Prints an ASCII size summary chart
//!
//! Compile: rustc cache_manager.rs -o cache_manager
//! Or with Cargo: see [dependencies] note in main()

use std::collections::BTreeMap;
use std::fs;
use std::path::{Path, PathBuf};
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug)]
struct CacheEntry {
    path:          PathBuf,
    size_bytes:    u64,
    age_days:      f64,
}

fn scan_cache(dl_dir: &Path) -> Vec<CacheEntry> {
    let mut entries = Vec::new();

    let read_dir = match fs::read_dir(dl_dir) {
        Ok(r) => r,
        Err(e) => {
            eprintln!("Cannot read dl/ dir: {e}");
            return entries;
        }
    };

    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or_default()
        .as_secs_f64();

    for item in read_dir.flatten() {
        let path = item.path();
        if !path.is_file() { continue; }

        // Skip .hash, .lock, and other metadata files
        let ext = path.extension().and_then(|e| e.to_str()).unwrap_or("");
        if ext == "hash" || ext == "lock" { continue; }

        let meta = match fs::metadata(&path) {
            Ok(m) => m,
            Err(_) => continue,
        };

        let size_bytes = meta.len();
        let mtime = meta.modified()
            .ok()
            .and_then(|t| t.duration_since(UNIX_EPOCH).ok())
            .map(|d| d.as_secs_f64())
            .unwrap_or(0.0);

        let age_days = if mtime > 0.0 {
            (now - mtime) / 86400.0
        } else {
            0.0
        };

        entries.push(CacheEntry { path, size_bytes, age_days });
    }

    entries.sort_by(|a, b| b.size_bytes.cmp(&a.size_bytes));
    entries
}

fn format_bytes(bytes: u64) -> String {
    const GB: u64 = 1_073_741_824;
    const MB: u64 = 1_048_576;
    const KB: u64 = 1_024;

    if bytes >= GB {
        format!("{:.2} GB", bytes as f64 / GB as f64)
    } else if bytes >= MB {
        format!("{:.1} MB", bytes as f64 / MB as f64)
    } else if bytes >= KB {
        format!("{:.0} KB", bytes as f64 / KB as f64)
    } else {
        format!("{bytes} B")
    }
}

fn print_cache_report(entries: &[CacheEntry], top_n: usize) {
    let total_bytes: u64 = entries.iter().map(|e| e.size_bytes).sum();
    let total_files = entries.len();

    println!("=================================================================");
    println!("  BUILDROOT dl/ CACHE REPORT");
    println!("=================================================================");
    println!("  Total files: {total_files}");
    println!("  Total size:  {}", format_bytes(total_bytes));
    println!();

    println!("  TOP {} LARGEST ENTRIES", top_n.min(total_files));
    println!("  {}", "-".repeat(65));

    let max_size = entries.first().map(|e| e.size_bytes).unwrap_or(1);
    let shown = entries.iter().take(top_n);

    for entry in shown {
        let fname = entry.path.file_name()
            .and_then(|n| n.to_str())
            .unwrap_or("?");
        let bar_len = ((entry.size_bytes as f64 / max_size as f64) * 30.0) as usize;
        let bar = "#".repeat(bar_len.max(1));
        let pct = (entry.size_bytes as f64 / total_bytes as f64) * 100.0;

        println!("  {fname:<40} {size:>9}  [{bar:<30}] {pct:.1}%",
                 fname = &fname[..40.min(fname.len())],
                 size  = format_bytes(entry.size_bytes),
                 bar   = bar,
                 pct   = pct);
    }

    println!();
    println!("  AGE DISTRIBUTION");
    println!("  {}", "-".repeat(65));

    let mut buckets: BTreeMap<&str, (u64, u64)> = BTreeMap::new();
    buckets.insert("< 1 day",    (0, 0));
    buckets.insert("1-7 days",   (0, 0));
    buckets.insert("8-30 days",  (0, 0));
    buckets.insert("31-90 days", (0, 0));
    buckets.insert("> 90 days",  (0, 0));

    for e in entries {
        let bucket = if      e.age_days < 1.0  { "< 1 day" }
                     else if e.age_days < 7.0  { "1-7 days" }
                     else if e.age_days < 30.0 { "8-30 days" }
                     else if e.age_days < 90.0 { "31-90 days" }
                     else                      { "> 90 days" };
        if let Some(entry) = buckets.get_mut(bucket) {
            entry.0 += 1;
            entry.1 += e.size_bytes;
        }
    }

    for (label, (count, bytes)) in &buckets {
        println!("  {label:<14} {count:>4} files  {size:>10}",
                 label = label,
                 count = count,
                 size  = format_bytes(*bytes));
    }

    println!("=================================================================");
}

fn prune_old_entries(entries: &[CacheEntry], max_age_days: f64, dry_run: bool) {
    let to_prune: Vec<&CacheEntry> = entries.iter()
        .filter(|e| e.age_days > max_age_days)
        .collect();

    if to_prune.is_empty() {
        println!("  No entries older than {max_age_days:.0} days to prune.");
        return;
    }

    let prune_bytes: u64 = to_prune.iter().map(|e| e.size_bytes).sum();
    println!("  Pruning {} entries ({}) older than {:.0} days {}",
             to_prune.len(),
             format_bytes(prune_bytes),
             max_age_days,
             if dry_run { "[DRY RUN]" } else { "" });

    for entry in &to_prune {
        let fname = entry.path.file_name()
            .and_then(|n| n.to_str())
            .unwrap_or("?");
        println!("  DELETE  {fname}  ({} days old)", entry.age_days as u64);
        if !dry_run {
            if let Err(e) = fs::remove_file(&entry.path) {
                eprintln!("  ERROR removing {:?}: {e}", entry.path);
            }
        }
    }
}

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let dl_dir = args.get(1).map(Path::new).unwrap_or(Path::new("dl"));
    let max_age: f64 = args.get(2)
        .and_then(|s| s.parse().ok())
        .unwrap_or(90.0);
    let dry_run = args.get(3).map(|s| s == "--dry-run").unwrap_or(true);

    let entries = scan_cache(dl_dir);
    print_cache_report(&entries, 15);
    println!();
    prune_old_entries(&entries, max_age, dry_run);
}
```

### Legal Manifest Analyzer (Rust)

```rust
//! legal_analyzer.rs
//! Parses the Buildroot legal-info manifest.csv and generates a
//! license compatibility report, grouping packages by license family.
//!
//! Compile: rustc legal_analyzer.rs -o legal_analyzer

use std::collections::{BTreeMap, HashSet};
use std::fs;
use std::path::Path;

#[derive(Debug, Clone)]
struct Package {
    name:     String,
    version:  String,
    license:  String,
}

#[derive(Debug, Clone, PartialEq)]
enum LicenseCategory {
    Permissive,   // MIT, BSD-*, Apache-2.0, ISC, Zlib, etc.
    CopyLeft,     // GPL-*, LGPL-*, AGPL-*
    Proprietary,  // Closed-source
    Unknown,
}

impl LicenseCategory {
    fn classify(license: &str) -> Self {
        let lic = license.to_lowercase();

        if lic.contains("gpl") || lic.contains("agpl") {
            LicenseCategory::CopyLeft
        } else if lic.contains("proprietary") || lic.contains("commercial") {
            LicenseCategory::Proprietary
        } else if lic.contains("mit")
            || lic.contains("bsd")
            || lic.contains("apache")
            || lic.contains("isc")
            || lic.contains("zlib")
            || lic.contains("unlicense")
            || lic.contains("public domain")
        {
            LicenseCategory::Permissive
        } else {
            LicenseCategory::Unknown
        }
    }

    fn label(&self) -> &str {
        match self {
            LicenseCategory::Permissive  => "Permissive",
            LicenseCategory::CopyLeft    => "CopyLeft  ",
            LicenseCategory::Proprietary => "Commercial",
            LicenseCategory::Unknown     => "Unknown   ",
        }
    }
}

fn parse_manifest(path: &Path) -> Vec<Package> {
    let content = match fs::read_to_string(path) {
        Ok(c) => c,
        Err(e) => {
            eprintln!("Cannot read manifest: {e}");
            return vec![];
        }
    };

    let mut packages = Vec::new();
    let mut lines = content.lines();
    lines.next(); // skip header

    for line in lines {
        let fields: Vec<&str> = line.splitn(6, ',').collect();
        if fields.len() < 3 { continue; }
        packages.push(Package {
            name:    fields[0].trim_matches('"').to_string(),
            version: fields[1].trim_matches('"').to_string(),
            license: fields[2].trim_matches('"').to_string(),
        });
    }

    packages
}

fn print_legal_report(packages: &[Package], denylist: &HashSet<&str>) {
    let mut by_category: BTreeMap<String, Vec<&Package>> = BTreeMap::new();
    let mut violations = Vec::new();

    for pkg in packages {
        let category = LicenseCategory::classify(&pkg.license);
        by_category
            .entry(category.label().trim().to_string())
            .or_default()
            .push(pkg);

        if denylist.contains(pkg.license.as_str()) {
            violations.push(pkg);
        }
    }

    println!("=================================================================");
    println!("  BUILDROOT LEGAL-INFO ANALYSIS");
    println!("=================================================================");
    println!("  Total packages: {}", packages.len());

    // Category breakdown bar chart
    println!("\n  LICENSE CATEGORY BREAKDOWN");
    println!("  {}", "-".repeat(60));

    let max_count = by_category.values().map(|v| v.len()).max().unwrap_or(1);
    for (cat, pkgs) in &by_category {
        let bar_len = (pkgs.len() as f64 / max_count as f64 * 35.0) as usize;
        let bar = "#".repeat(bar_len.max(1));
        let pct = (pkgs.len() as f64 / packages.len() as f64) * 100.0;
        println!("  {cat:<12} [{bar:<35}] {count:>3} ({pct:.0}%)",
                 cat = cat, bar = bar, count = pkgs.len(), pct = pct);
    }

    // CopyLeft packages (need source disclosure)
    println!("\n  COPYLEFT PACKAGES (require source disclosure)");
    println!("  {}", "-".repeat(60));
    if let Some(pkgs) = by_category.get("CopyLeft") {
        for pkg in pkgs {
            println!("  {name:<40} {ver:<15} {lic}",
                     name = pkg.name, ver = pkg.version, lic = pkg.license);
        }
    } else {
        println!("  (none)");
    }

    // Violations
    println!("\n  POLICY VIOLATIONS");
    println!("  {}", "-".repeat(60));
    if violations.is_empty() {
        println!("  PASS: No policy violations detected");
    } else {
        for pkg in &violations {
            println!("  FAIL: {name} {ver} — {lic}",
                     name = pkg.name, ver = pkg.version, lic = pkg.license);
        }
    }

    println!("=================================================================");

    if !violations.is_empty() {
        std::process::exit(1);
    }
}

fn main() {
    let manifest_path = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "output/legal-info/manifest.csv".to_string());

    // Define denied licenses for this project
    let denylist: HashSet<&str> = [
        "GPL-3.0",
        "GPL-3.0+",
        "AGPL-3.0",
        "AGPL-3.0+",
        "SSPL-1.0",
    ].iter().cloned().collect();

    let packages = parse_manifest(Path::new(&manifest_path));
    if packages.is_empty() {
        eprintln!("No packages found in manifest.");
        std::process::exit(2);
    }

    print_legal_report(&packages, &denylist);
}
```

---

## Summary

```
CHAPTER 28 — CI/CD PIPELINE INTEGRATION: KEY CONCEPTS
=======================================================

  +-----------------------------------------------------------+
  |  CONCEPT            WHAT IT DOES           WHY IT MATTERS |
  +-----------------------------------------------------------+
  |  dl/ caching        Persists downloaded     Cuts 45min    |
  |                     source tarballs         to 2min on    |
  |                     across CI runs          cache hit     |
  +-----------------------------------------------------------+
  |  ccache             Caches compiler         10x speedup   |
  |                     outputs by source       on incremental|
  |                     file + flags hash       builds        |
  +-----------------------------------------------------------+
  |  -j$(nproc)         Parallel make jobs      Full CPU      |
  |                     inside each package     utilisation   |
  |                     build                   on any runner |
  +-----------------------------------------------------------+
  |  Artifact           Promotes validated     What is tested |
  |  promotion          builds through dev ->   is exactly    |
  |                     staging -> production   what ships    |
  +-----------------------------------------------------------+
  |  Docker containers  Identical host env      Reproducible  |
  |                     for every runner        builds, no    |
  |                     and developer           host pollution|
  +-----------------------------------------------------------+
  |  make legal-info    Scans all packages       Enforces OSS |
  |  gating             for license metadata,   license       |
  |                     blocks non-compliant    compliance    |
  |                     licenses                automatically |
  +-----------------------------------------------------------+

  RECOMMENDED PIPELINE STRUCTURE
  ================================
  [prepare] -> [build] -> [validate] -> [test] -> [promote]
      |            |           |            |           |
   Docker       make        legal-info   QEMU       upload
   setup        -j$(nproc)  image-size   smoke      to registry
   env check    restore     check        test       on tag

  CRITICAL CONFIGURATION SETTINGS
  ================================
  BR2_DL_DIR      = /cache/buildroot-dl  (shared NFS/S3 cache)
  BR2_CCACHE      = y                    (enable ccache)
  BR2_CCACHE_DIR  = /cache/ccache        (shared ccache store)
  BR2_JLEVEL      = $(nproc)             (auto parallel jobs)
  ccache hash_dir = false                (portable cache keys)

  LANGUAGES USED IN THIS CHAPTER
  ================================
  C       — license_check.c, build_stats.c
            Low-level tools suitable for minimal build environments
  C++     — artifact_verify.cpp, pipeline_config.cpp
            Object-oriented pipeline utilities with STL containers
  Rust    — ci_monitor.rs, cache_manager.rs, legal_analyzer.rs
            Safe, expressive CI tooling with strong type guarantees

  COMMON PITFALLS
  ================================
  - Not setting ccache hash_dir=false (cache misses on different runners)
  - Caching the entire output/ dir (too large; only cache dl/ and ccache)
  - Running Buildroot as root (it will refuse)
  - Forgetting to run legal-info after adding new packages
  - Using WidthType.PERCENTAGE in docx tables (breaks Google Docs rendering)
  - Not keeping legal-info artifacts for 90+ days (audit trail requirement)
```

Integrating Buildroot into a modern CI/CD pipeline, whether GitLab CI or GitHub Actions,
requires careful attention to the unique characteristics of embedded Linux builds: long
cycle times, large download footprints, and stringent license obligations. By combining
persistent caching of `dl/` and `ccache`, aggressive parallelism via `-j$(nproc)`,
Docker-based build containers for reproducibility, and an automated `make legal-info` gate
for compliance, teams can achieve fast, reliable, and legally sound firmware delivery
pipelines. The code examples in C, C++, and Rust demonstrate how each layer of the pipeline
can be instrumented, monitored, and enforced programmatically, giving embedded teams the
same level of CI/CD maturity that software teams have long enjoyed.

---

*End of Chapter 28 — CI/CD Pipeline Integration (GitLab / GitHub Actions)*