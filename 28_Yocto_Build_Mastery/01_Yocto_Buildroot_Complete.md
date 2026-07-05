# Yocto and Buildroot Complete Guide — Building Embedded Linux from Scratch

> **Analogy:** Yocto is like a full chef's kitchen — every tool, every ingredient,  
> customizable everything. Buildroot is like a meal kit — faster, simpler, just enough.  
> This guide teaches both until you can build a production embedded Linux distro.

---

## Chapter 1: Why Not Just Use Ubuntu?

```
Ubuntu on embedded:
  - 2-4 GB rootfs (you need 16MB)
  - Starts 300+ services you don't need
  - Package updates break your product
  - Contains GPL libraries you don't need to ship
  - No control over kernel version

Custom embedded Linux with Yocto/Buildroot:
  - 4-32 MB rootfs (only what you need)
  - Boot in 3 seconds
  - Reproducible builds (same inputs → same output, always)
  - Full control over every component version
  - License compliance manifest generated automatically
```

---

## Part 1: Buildroot (Learn This First)

### Why Buildroot First?

Buildroot compiles in 20-60 minutes, has a 3-level menu config, and you understand what's happening.
Yocto takes 4-8 hours first build, has layers/recipes/BitBake/metadata. Learn Buildroot basics first.

### Installation and Setup

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt-get install -y \
    build-essential git libncurses5-dev unzip bc \
    python3 python3-pip wget cpio rsync

# Get Buildroot
cd ~/
git clone https://git.buildroot.net/buildroot
cd buildroot
git checkout 2024.02  # LTS release

# Directory structure:
ls
# arch/          - CPU architecture support
# board/         - Board-specific files and configs
# configs/       - Pre-made defconfigs for known boards
# dl/            - Downloaded sources (cache)
# output/        - Build output
# package/       - All packages (one directory per package)
# system/        - Root filesystem skeleton
# toolchain/     - Cross-compiler configuration

# Pre-made configs for Radxa:
ls configs/ | grep -i rock
# rock64_defconfig
# rock_pi_4_defconfig
# (Rock 5B+ may not be here — we'll create it)
```

### Buildroot Quick Start: QEMU ARM64

```bash
# Step 1: Configure for QEMU arm64
make qemu_aarch64_virt_defconfig

# Step 2: Customize (optional)
make menuconfig
# Key options to explore:
# Target options → AArch64 little endian
# Toolchain → Buildroot toolchain (or external)
# System configuration → hostname, banner, root password
# Kernel → Linux version
# Target packages → your app and its dependencies
# Filesystem images → ext2/ext4 size

# Step 3: Build everything
make -j$(nproc)
# First run: 30-90 minutes (downloads + builds toolchain + kernel + rootfs)
# Subsequent runs: 2-5 minutes (incremental)

# Step 4: Run in QEMU
output/images/start-qemu.sh
# If this file exists, run it directly.
# Otherwise:
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a53 \
    -nographic \
    -smp 4 \
    -m 1G \
    -kernel output/images/Image \
    -append "rootwait root=/dev/vda console=ttyAMA0" \
    -drive file=output/images/rootfs.ext4,if=virtio,format=raw \
    -netdev user,id=eth0 \
    -device virtio-net-pci,netdev=eth0
```

### Adding a Custom Package to Buildroot

```bash
# Create package directory
mkdir -p package/my_sensor_daemon

# Package Config file
cat > package/my_sensor_daemon/Config.in << 'EOF'
config BR2_PACKAGE_MY_SENSOR_DAEMON
    bool "my_sensor_daemon"
    depends on BR2_PACKAGE_LIBGPIOD
    help
      Reads temperature sensors and exposes via REST API.
      
      Depends on libgpiod for GPIO access.
EOF

# Package Makefile (Buildroot's .mk format)
cat > package/my_sensor_daemon/my_sensor_daemon.mk << 'EOF'
################################################################################
#
# my_sensor_daemon
#
################################################################################

MY_SENSOR_DAEMON_VERSION = 1.0
MY_SENSOR_DAEMON_SITE = $(BR2_EXTERNAL)/src/my_sensor_daemon
MY_SENSOR_DAEMON_SITE_METHOD = local
MY_SENSOR_DAEMON_DEPENDENCIES = libgpiod

define MY_SENSOR_DAEMON_BUILD_CMDS
    $(MAKE) CC=$(TARGET_CC) -C $(@D) all
endef

define MY_SENSOR_DAEMON_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/my_sensor_daemon \
        $(TARGET_DIR)/usr/bin/my_sensor_daemon
    $(INSTALL) -D -m 0644 $(@D)/my_sensor_daemon.service \
        $(TARGET_DIR)/etc/systemd/system/my_sensor_daemon.service
endef

$(eval $(generic-package))
EOF

# Add to package/Config.in (or use BR2_EXTERNAL mechanism):
# menu "Custom packages"
#   source "package/my_sensor_daemon/Config.in"
# endmenu

# Enable in menuconfig:
make menuconfig
# Target packages → My custom packages → my_sensor_daemon

# Build only this package (incremental):
make my_sensor_daemon
make my_sensor_daemon-rebuild   # force rebuild
make my_sensor_daemon-dirclean  # clean and rebuild from scratch
```

### Buildroot for Radxa Rock 5B+

```bash
# Create Rock 5B+ config
cat > configs/rock5b_plus_defconfig << 'EOF'
BR2_aarch64=y
BR2_TOOLCHAIN_BUILDROOT_CXX=y
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_CUSTOM_GIT=y
BR2_LINUX_KERNEL_CUSTOM_REPO_URL="https://github.com/radxa/kernel.git"
BR2_LINUX_KERNEL_CUSTOM_REPO_VERSION="linux-6.1-stan-rkr1"
BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG=y
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="$(BR2_EXTERNAL)/board/rock5b/linux.config"
BR2_LINUX_KERNEL_DTS_SUPPORT=y
BR2_LINUX_KERNEL_INTREE_DTS_NAME="rockchip/rk3588-rock-5b-plus"
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
BR2_TARGET_ROOTFS_EXT2_SIZE="512M"
BR2_PACKAGE_OPENSSH=y
BR2_PACKAGE_PYTHON3=y
BR2_PACKAGE_LIBGPIOD=y
EOF

make rock5b_plus_defconfig
make -j$(nproc)
```

---

## Part 2: Yocto Project (Production Grade)

### Understanding Yocto Architecture

```
Yocto Core Concepts:
                                          
  Metadata (Recipes, Config, Classes)     
       ↓                                  
  BitBake (build engine — like make)      
       ↓                                  
  Shared State Cache (sstate-cache)       
  ← speeds up repeated builds             
       ↓                                  
  Build Directory (tmp/)                  
  ← compiled output                       
       ↓                                  
  Images (core-image-minimal.rootfs.ext4) 

Layers:
  meta/                  ← Poky core (OpenEmbedded)
  meta-poky/             ← Yocto reference distro
  meta-yocto-bsp/        ← Reference BSPs (qemu, etc.)
  meta-openembedded/     ← Extra packages (networking, etc.)
  meta-rockchip/         ← Rockchip SoC support
  meta-radxa/            ← Radxa board support
  meta-my-product/       ← YOUR layer (your recipes)
```

### Yocto Installation

```bash
# Dependencies
sudo apt-get install -y \
    gawk wget git-core diffstat unzip texinfo \
    gcc-multilib build-essential chrpath socat cpio \
    python3 python3-pip python3-pexpect \
    xz-utils debianutils iputils-ping \
    libsdl1.2-dev xterm

# Clone Poky (Yocto reference distribution)
cd ~/
git clone -b scarthgap https://git.yoctoproject.org/poky
# scarthgap = Yocto 5.0 LTS (2024)

cd poky

# Set up build environment (creates build/ directory)
source oe-init-build-env build-rock5b
# This puts you inside build-rock5b/

# Key configuration files:
ls conf/
# bblayers.conf   ← which layers to use
# local.conf      ← your build settings
```

### Configure Your Build

```bash
# conf/local.conf — key settings
cat >> conf/local.conf << 'EOF'
# Target machine
MACHINE = "rock-5b-plus"

# Parallel build optimization
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j8"

# Disk space management
INHERIT += "rm_work"    # delete build files after packaging (saves disk space)

# Package manager
PACKAGE_CLASSES = "package_ipk"  # use ipk (like .deb but for embedded)

# SDK (for cross-compilation by customers)
SDKMACHINE = "x86_64"

# License auditing
INHERIT += "archiver"
ARCHIVER_MODE[src] = "patched"
EOF

# conf/bblayers.conf — which layers to include
cat > conf/bblayers.conf << 'CONF'
BBLAYERS ?= " \
  ${TOPDIR}/../poky/meta \
  ${TOPDIR}/../poky/meta-poky \
  ${TOPDIR}/../poky/meta-yocto-bsp \
  ${TOPDIR}/../meta-openembedded/meta-oe \
  ${TOPDIR}/../meta-openembedded/meta-python \
  ${TOPDIR}/../meta-openembedded/meta-networking \
  ${TOPDIR}/../meta-rockchip \
  ${TOPDIR}/../meta-radxa \
  ${TOPDIR}/../meta-my-product \
  "
CONF

# Add layers
cd ~/
git clone -b scarthgap https://github.com/openembedded/meta-openembedded
git clone -b scarthgap https://github.com/JeffyCN/meta-rockchip
# Get meta-radxa from Radxa GitHub
```

### Writing a Yocto Recipe

Recipes end in `.bb` and describe how to build one package:

```bash
# Create your layer first
bitbake-layers create-layer ~/meta-my-product
# Creates: meta-my-product/
#   ├── conf/layer.conf
#   ├── COPYING.MIT
#   ├── README
#   └── recipes-example/example/example_0.1.bb

# Create a recipe for your embedded AI daemon
mkdir -p ~/meta-my-product/recipes-daemon/sensor-ai-daemon/

cat > ~/meta-my-product/recipes-daemon/sensor-ai-daemon/sensor-ai-daemon_1.0.bb << 'EOF'
SUMMARY = "AI-powered sensor monitoring daemon"
DESCRIPTION = "Reads I2C sensors and runs anomaly detection using TFLite"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=abc123..."

# Where to get the source
SRC_URI = "git://github.com/brk4embed/sensor-ai-daemon.git;branch=main;protocol=https"
SRCREV = "${AUTOREV}"        # use latest commit (pin to hash in production)

S = "${WORKDIR}/git"          # source directory after checkout

# Build dependencies (needed at compile time)
DEPENDS = "tflite python3"

# Runtime dependencies (needed at runtime)
RDEPENDS:${PN} = "python3-numpy libgpiod"

# Inherit autotools if using autoconf/automake
# inherit autotools pkgconfig

# Custom compile step (if CMake)
inherit cmake

# Install step
do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/sensor_ai_daemon ${D}${bindir}/
    
    install -d ${D}${systemd_unitdir}/system/
    install -m 0644 ${S}/sensor-ai-daemon.service \
                    ${D}${systemd_unitdir}/system/
}

# Enable systemd service
inherit systemd
SYSTEMD_SERVICE:${PN} = "sensor-ai-daemon.service"
SYSTEMD_AUTO_ENABLE = "enable"

# Files in this package
FILES:${PN} = " \
    ${bindir}/sensor_ai_daemon \
    ${systemd_unitdir}/system/sensor-ai-daemon.service \
"
EOF
```

### Creating a Custom Image

```bash
# meta-my-product/recipes-images/images/my-product-image.bb
cat > ~/meta-my-product/recipes-images/images/my-product-image.bb << 'EOF'
SUMMARY = "Industrial AI monitoring system image"

# Inherit from base image
require recipes-core/images/core-image-minimal.bb

# Packages to include
IMAGE_INSTALL:append = " \
    openssh \
    python3 \
    python3-pip \
    python3-numpy \
    libgpiod \
    libgpiod-tools \
    i2c-tools \
    tflite \
    sensor-ai-daemon \
    curl \
    htop \
    nano \
"

# Image size (in KB)
IMAGE_ROOTFS_SIZE = "524288"  # 512 MB

# Features
IMAGE_FEATURES += " \
    ssh-server-openssh \
    package-management \
    debug-tweaks \
"

# Remove debug-tweaks in production!
# debug-tweaks: empty root password, debug tools
EOF

# Build the image
bitbake my-product-image
# First build: 4-8 hours (builds compiler, kernel, all packages)
# Subsequent: 10-30 minutes (uses sstate-cache)

# Output:
ls tmp/deploy/images/rock-5b-plus/
# my-product-image-rock-5b-plus.rootfs.ext4  ← flash this to eMMC
# Image                                        ← kernel
# rk3588-rock-5b-plus.dtb                     ← device tree
```

### Building an SDK (Cross-Compilation Toolchain)

```bash
# Generate SDK that developers can use on their laptops
bitbake my-product-image -c populate_sdk

# Output: tmp/deploy/sdk/poky-glibc-x86_64-my-product-image-aarch64-toolchain-5.0.sh

# Install SDK on developer's laptop
./poky-glibc-x86_64-my-product-image-aarch64-toolchain-5.0.sh
# Installs to /opt/poky/5.0/

# Use SDK to cross-compile your application
source /opt/poky/5.0/environment-setup-aarch64-poky-linux
echo $CC    # aarch64-poky-linux-gcc with all flags pre-set
echo $SYSROOT   # path to target sysroot (all libs, headers)

# Build your app (no need to modify Makefile)
make my_app   # uses $CC automatically
```

---

## Comparison: When to Use What

| Scenario | Best Choice | Reason |
|----------|-------------|--------|
| Learning / prototyping | Buildroot | Fast setup, simple config |
| Mass-market product | Yocto | Reproducible, license compliance, SDK |
| Quick hobby project | Buildroot | Done in an afternoon |
| Product with OTA updates | Yocto + SWUpdate | swupdate recipe in meta-oe |
| Company with multiple products | Yocto | Shared BSP layer, multiple image recipes |
| Customer needs to build apps | Yocto | SDK generation |
| Chromebook / complex BSP | Yocto | All ChromeOS Coreboot-based boards use Yocto |

---

## Debugging Build Issues

```bash
# Yocto: find out why a recipe failed
bitbake sensor-ai-daemon -v 2>&1 | tail -50

# Show all tasks for a recipe
bitbake sensor-ai-daemon -c listtasks

# Clean and rebuild one recipe
bitbake sensor-ai-daemon -c cleansstate
bitbake sensor-ai-daemon

# Inspect the built package contents
bitbake sensor-ai-daemon -c package
ls tmp/packages-split/sensor-ai-daemon/

# Buildroot: verbose build
make V=1 my_sensor_daemon 2>&1 | tail -50

# Debug: run shell inside build environment
bitbake sensor-ai-daemon -c devshell
# Opens a shell with all environment variables set
# $CC, $CFLAGS, $SYSROOT all configured
# You can manually run: $CC hello.c -o hello
```
