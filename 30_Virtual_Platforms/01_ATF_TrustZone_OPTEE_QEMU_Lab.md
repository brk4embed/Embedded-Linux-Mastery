# ATF, TrustZone, OP-TEE & Secure Boot — QEMU Practice Lab

> **Goal:** Run a complete TrustZone security stack on your laptop using QEMU — no real hardware needed. You will build ATF (ARM Trusted Firmware), OP-TEE (the open-source equivalent of QTEE), Linux, and practice everything your SC7180/SC7280 production work relied on.

---

## What You Will Build (The Complete Picture)

```
┌─────────────────────────────────────────────────────────────────┐
│                    QEMU ARM64 virt machine                       │
│                    (your laptop emulates this)                   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  EL3 — Secure Monitor (ATF BL31)                          │  │
│  │  "The gatekeeper between Secure and Non-Secure worlds"    │  │
│  │  You will BUILD this from source (TF-A)                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│         ↕ SMC calls (Secure Monitor Call)                        │
│  ┌──────────────────────┐   ┌──────────────────────────────┐   │
│  │  Secure World (EL1-S)│   │  Non-Secure World (EL1-NS)   │  │
│  │                      │   │                              │  │
│  │  OP-TEE OS           │   │  Linux Kernel                │  │
│  │  (= QTEE equivalent) │   │  + OP-TEE driver             │  │
│  │                      │   │  + optee-client (= QSEE)     │  │
│  │  Trusted Apps (TAs): │   │                              │  │
│  │  - Hello World TA    │   │  Normal Apps:                │  │
│  │  - Crypto TA         │   │  - xtest (test suite)        │  │
│  │  - Secure Storage TA │   │  - optee_example             │  │
│  └──────────────────────┘   └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Understanding the Architecture First (Read Before Building)

### EL Levels Explained Simply

```
EL = Exception Level = privilege level on ARM64

EL3 (Most privileged — "System Administrator")
  → Only ATF (ARM Trusted Firmware) BL31 runs here
  → Handles SMC (Secure Monitor Calls)
  → Switches CPU between Secure and Non-Secure worlds
  → NEVER returns to EL3 after init (just handles SMC exceptions)

EL2 (Hypervisor — "Building Manager")
  → Optional: used for virtualization (KVM, pKVM)
  → Not used in basic TrustZone setup

EL1-S (Secure EL1 — "Bank Vault")
  → OP-TEE OS runs here (on Qualcomm: QTEE)
  → Can access BOTH Secure AND Non-Secure memory
  → Hosts Trusted Applications (TAs)
  → Linux CANNOT reach here directly

EL1-NS (Non-Secure EL1 — "Regular Office")
  → Linux kernel runs here
  → CAN ONLY access Non-Secure memory
  → To ask Secure World for service: must use SMC instruction
    → CPU jumps to EL3 (ATF) → ATF forwards to OP-TEE → response returns

EL0 (User Space — "Employees")
  → Regular apps (bash, python) run here
  → OP-TEE client library + Trusted Application clients run here
```

### What Happens When Linux Wants a Secure Service

```
Linux App: "Decrypt this data using the key stored in Secure World"
    ↓
optee-client library (EL0-NS)
    ↓ invokes
TEE supplicant (EL0-NS daemon) + kernel TEE driver (EL1-NS)
    ↓ executes
SMC instruction (CPU exception to EL3)
    ↓
ATF BL31 (EL3) receives SMC, identifies it as OP-TEE call
    ↓ switches world
OP-TEE OS (EL1-S) receives the request
    ↓ opens session to
Trusted Application (TA) running in OP-TEE
    ↓ does work, returns result
OP-TEE → ATF → Linux kernel → optee-client → App

Total: ~5 context switches, ~microseconds
```

### Secure Boot Chain (What You Will Practice)

```
Power ON
    ↓
QEMU ROM (equivalent to SoC BootROM)
    ↓ loads
ATF BL1 (Boot Loader stage 1)
  → Verifies BL2 signature using public key
    ↓ loads (if signature OK)
ATF BL2 (Boot Loader stage 2)
  → Verifies BL31 (Secure Monitor) signature
  → Verifies BL32 (OP-TEE) signature
  → Verifies BL33 (U-Boot) signature
    ↓ loads all verified images
ATF BL31 → OP-TEE (BL32) → U-Boot (BL33) → Linux Kernel
    
If ANY signature check fails → BOOT STOPS (Secure Boot)
```

---

## Part 1: Install All Prerequisites

```bash
sudo apt update
sudo apt install -y \
    qemu-system-arm \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    gcc-arm-linux-gnueabihf \
    device-tree-compiler \
    python3 python3-pip python3-pyelftools \
    python3-pycparser \
    make cmake ninja-build \
    libssl-dev openssl \
    uuid-dev \
    acpica-tools \
    xz-utils \
    cpio \
    rsync \
    git curl wget \
    flex bison \
    bc libncurses-dev \
    pkg-config

# Python packages needed for ATF and OP-TEE build
pip3 install pyelftools pycparser cryptography

# Verify QEMU supports ARM64 with secure mode
qemu-system-aarch64 -machine virt,secure=on -nographic -kernel /dev/null 2>&1 | head -5
# Expected: "Unable to load kernel" or similar — means secure=on is supported

echo "=== All prerequisites installed ==="
```

---

## Part 2: Create the Workspace

```bash
# Create a dedicated workspace
mkdir -p ~/trustzone-lab
cd ~/trustzone-lab

# We will have this structure:
# ~/trustzone-lab/
# ├── atf/              ← ARM Trusted Firmware (BL1, BL2, BL31)
# ├── optee_os/         ← OP-TEE Secure OS (BL32 = equivalent of QTEE)
# ├── optee_client/     ← OP-TEE client library (user space = equivalent of QSEE client)
# ├── optee_test/       ← xtest — comprehensive test suite
# ├── optee_examples/   ← Hello World TA and other example TAs
# ├── u-boot/           ← U-Boot bootloader (BL33)
# ├── linux/            ← Linux kernel with OP-TEE driver
# ├── busybox/          ← Minimal root filesystem
# └── out/              ← All compiled outputs go here

mkdir -p ~/trustzone-lab/out
echo "Workspace created at ~/trustzone-lab"
```

---

## Part 3: Build ATF (ARM Trusted Firmware)

### What Is ATF?

ATF (also called TF-A: Trusted Firmware-A) is the **open-source reference implementation of EL3 firmware**. Qualcomm's production firmware at EL3 is proprietary (XBL + QTZ), but ATF is functionally identical for learning.

ATF provides:
- **BL1**: First code that runs from ROM, loads BL2
- **BL2**: Loads and verifies BL31, BL32, BL33
- **BL31**: The Secure Monitor (stays resident at EL3 FOREVER, handles SMC calls)
- **BL32**: Optional — this is where OP-TEE lives
- **BL33**: Non-secure bootloader (U-Boot)

```bash
cd ~/trustzone-lab

# Clone ATF (TF-A)
git clone https://git.trustedfirmware.org/TF-A/trusted-firmware-a.git atf
# OR mirror on GitHub (faster sometimes):
git clone https://github.com/ARM-software/arm-trusted-firmware.git atf

cd atf

# Check latest stable tag
git tag | grep "^v" | sort -V | tail -5

# Use a stable version
git checkout v2.10  # or latest stable

# Build ATF for QEMU ARM64 virt platform
# Platform: qemu — QEMU's ARM64 virt machine
# BL32_RAM_LOCATION=tdram — OP-TEE in DRAM (simpler for learning)
# SPD=opteed — Secure Payload Dispatcher for OP-TEE

make -j$(nproc) \
    CROSS_COMPILE=aarch64-linux-gnu- \
    ARCH=aarch64 \
    PLAT=qemu \
    BL32_RAM_LOCATION=tdram \
    SPD=opteed \
    DEBUG=1 \
    LOG_LEVEL=50 \
    bl1 bl2 bl31

# What was built:
ls -la build/qemu/debug/
# bl1.bin  → runs from QEMU ROM
# bl2.bin  → loaded by BL1
# bl31.bin → Secure Monitor (stays resident at EL3)
# Notice: no bl32.bin here — that is OP-TEE, built separately

echo "=== ATF build complete ==="
ls -lh build/qemu/debug/bl*.bin
```

### What Each BL Stage Does (Walk Through the Code)

```bash
# Let's look at the QEMU platform entry point
cat plat/qemu/qemu/include/platform_def.h | grep -E "BL[0-9]|DRAM|ROM" | head -20

# Key addresses for QEMU platform:
# BL1:  0x0000000000000000 (ROM base)
# BL2:  0x000000000e000000 (DRAM)
# BL31: 0x000000000e040000 (DRAM, stays resident)
# BL32: 0x000000000e100000 (DRAM, OP-TEE)
# BL33: 0x000000004a000000 (DRAM, U-Boot/Linux)
```

---

## Part 4: Build OP-TEE OS (The QTEE Equivalent)

### What Is OP-TEE?

OP-TEE (Open Portable Trusted Execution Environment) is the **open-source version of QTEE**. Qualcomm's QTEE is proprietary and NDA-protected. OP-TEE is functionally identical, fully open-source, and runs on QEMU.

OP-TEE provides:
- A secure OS running at EL1-S
- A framework for writing **Trusted Applications (TAs)**
- Crypto services (AES, RSA, SHA)
- Secure storage (encrypted key-value store)
- GlobalPlatform TEE API (industry standard — same API on Qualcomm/Trustonic/Samsung)

```bash
cd ~/trustzone-lab

# Clone OP-TEE OS
git clone https://github.com/OP-TEE/optee_os.git
cd optee_os

# Use a stable version
git checkout 4.2.0   # check latest: git tag | tail -5

# Build OP-TEE for QEMU ARM64 virt
# PLATFORM=vexpress-qemu_armv8a → QEMU ARM64 virt machine
make -j$(nproc) \
    CROSS_COMPILE=aarch64-linux-gnu- \
    PLATFORM=vexpress-qemu_armv8a \
    DEBUG=1 \
    CFG_TEE_CORE_LOG_LEVEL=4 \
    CFG_CORE_ASLR=n \
    CFG_CORE_HEAP_SIZE=524288

# What was built:
ls -lh out/arm-plat-vexpress/core/
# tee.elf   → OP-TEE ELF with debug symbols
# tee.bin   → raw binary loaded by ATF as BL32
# tee-header_v2.bin → header for ATF FIP package

echo "=== OP-TEE OS build complete ==="
```

### Understand OP-TEE's Secure Memory Layout

```bash
# Look at the memory map
cat out/arm-plat-vexpress/core/tee.map | head -30
# You will see:
# LOAD_TEE_RAM_START   0x0e100000  ← Secure memory start
# CFG_TZDRAM_START     0x0e100000  ← TrustZone DRAM start
# CFG_TZDRAM_SIZE      0x00f00000  ← 15MB for secure world

# OP-TEE has TWO memory regions:
# 1. Secure RAM (SRAM-like, fast, small)   → core code
# 2. Secure DRAM (larger, for TAs)         → TA loading
```

---

## Part 5: Build Your First Trusted Application (TA)

### What Is a Trusted Application?

A **Trusted Application** is code that runs INSIDE OP-TEE (the Secure World). It:
- Has a unique UUID (like a software serial number)
- Is loaded into Secure World memory when needed
- Provides a service to Normal World (Linux apps)
- Cannot be tampered with by Linux even if Linux is compromised

Think of it as a **secure plugin** that Linux apps can call but cannot modify.

```bash
cd ~/trustzone-lab

# Clone OP-TEE examples (includes Hello World TA)
git clone https://github.com/linaro-swg/optee_examples.git
cd optee_examples

# Look at Hello World TA structure
ls hello_world/
# host/   → Normal World application (runs in Linux user space)
# ta/     → Trusted Application (runs in OP-TEE Secure World)

# The TA interface (shared header):
cat hello_world/ta/include/hello_world_ta.h
```

The Hello World TA shows the complete pattern:

```c
/* hello_world_ta.h — THE CONTRACT between Normal and Secure world */

/* UUID — every TA has a unique identifier */
#define TA_HELLO_WORLD_UUID \
    { 0x8aaaf200, 0x2450, 0x11e4, \
        { 0xab, 0xe2, 0x00, 0x02, 0xa5, 0xd5, 0xc5, 0x1b} }

/* Commands the TA supports */
#define TA_HELLO_WORLD_CMD_INC_VALUE    0
#define TA_HELLO_WORLD_CMD_DEC_VALUE    1
```

```c
/* ta/hello_world_ta.c — RUNS IN SECURE WORLD (OP-TEE) */
TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx, uint32_t cmd_id,
    uint32_t param_types, TEE_Param params[4])
{
    switch (cmd_id) {
    case TA_HELLO_WORLD_CMD_INC_VALUE:
        inc_value(param_types, params);   /* increments a counter securely */
        return TEE_SUCCESS;
    default:
        return TEE_ERROR_BAD_PARAMETERS;
    }
}
```

```c
/* host/main.c — RUNS IN NORMAL WORLD (Linux user space) */
int main(void) {
    TEEC_Context ctx;
    TEEC_Session sess;
    TEEC_UUID uuid = TA_HELLO_WORLD_UUID;

    /* Open connection to TEE (goes through kernel → ATF → OP-TEE) */
    TEEC_InitializeContext(NULL, &ctx);
    TEEC_OpenSession(&ctx, &sess, &uuid,
        TEEC_LOGIN_PUBLIC, NULL, NULL, &origin);

    /* Call the Trusted Application */
    TEEC_InvokeCommand(&sess, TA_HELLO_WORLD_CMD_INC_VALUE, &op, &origin);

    /* Close session */
    TEEC_CloseSession(&sess);
    TEEC_FinalizeContext(&ctx);
}
```

```bash
# Build the Hello World example
cd ~/trustzone-lab/optee_examples

# Build TA (runs in Secure World)
make -C hello_world/ta \
    CROSS_COMPILE=aarch64-linux-gnu- \
    TA_DEV_KIT_DIR=../optee_os/out/arm-plat-vexpress/export-ta_arm64 \
    O=../out/ta

# Build Host app (runs in Linux Normal World)
make -C hello_world/host \
    CROSS_COMPILE=aarch64-linux-gnu- \
    TEEC_EXPORT=../optee_client/out/export/usr \
    O=../out/host

ls ../out/ta/
# 8aaaf200-2450-11e4-abe2-0002a5d5c51b.ta  ← compiled TA binary
# UUID is the filename!
```

---

## Part 6: Build OP-TEE Client (= QSEE Client Equivalent)

The **OP-TEE Client** is the Normal World (Linux) side of the TEE interface. It includes:
- `libteec` — shared library that Linux apps use to talk to TEE
- `tee-supplicant` — daemon that handles TA loading, secure storage

This is functionally what Qualcomm's QSEE client provides on Android/ChromeOS.

```bash
cd ~/trustzone-lab

git clone https://github.com/OP-TEE/optee_client.git
cd optee_client

make -j$(nproc) \
    CROSS_COMPILE=aarch64-linux-gnu- \
    WITH_TEEACL=0

ls out/export/usr/lib/
# libteec.so  ← library apps link against
# libteec.so.1
ls out/export/usr/sbin/
# tee-supplicant  ← daemon (must be running for TAs to work)
```

---

## Part 7: Build OP-TEE Test Suite (xtest)

`xtest` is the comprehensive test suite — it tests ALL OP-TEE features including crypto, storage, GlobalPlatform compliance. This is what validates your secure implementation works.

```bash
cd ~/trustzone-lab

git clone https://github.com/OP-TEE/optee_test.git
cd optee_test

make -j$(nproc) \
    CROSS_COMPILE=aarch64-linux-gnu- \
    TA_DEV_KIT_DIR=../optee_os/out/arm-plat-vexpress/export-ta_arm64 \
    TEEC_EXPORT=../optee_client/out/export/usr \
    OPTEE_CLIENT_EXPORT=../optee_client/out/export/usr \
    O=../out/xtest

ls ../out/xtest/
# xtest                          ← test runner binary (runs in Linux)
# *.ta                           ← test Trusted Applications (run in OP-TEE)
```

---

## Part 8: Build U-Boot (BL33 — Non-Secure Bootloader)

U-Boot runs at EL1-NS (Non-Secure EL1) after ATF hands over control. It loads the Linux kernel.

```bash
cd ~/trustzone-lab

git clone https://source.denx.de/u-boot/u-boot.git
# OR GitHub mirror:
git clone https://github.com/u-boot/u-boot.git
cd u-boot

git checkout v2024.04   # stable version

# Configure for QEMU ARM64
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- qemu_arm64_defconfig

# Build
make -j$(nproc) ARCH=arm CROSS_COMPILE=aarch64-linux-gnu-

ls u-boot.bin
# u-boot.bin  ← this becomes BL33 (handed control by ATF)
```

---

## Part 9: Build Linux Kernel with OP-TEE Driver

```bash
cd ~/trustzone-lab

# Use a recent stable kernel
git clone --depth=1 --branch v6.6 \
    https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git linux
cd linux

# Start with ARM64 defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig

# Enable OP-TEE driver (if not already enabled)
./scripts/config --enable CONFIG_TEE
./scripts/config --enable CONFIG_OPTEE
./scripts/config --enable CONFIG_OPTEE_INSECURE_LOAD_USER_TA

# Verify
grep "CONFIG_OPTEE" .config
# CONFIG_TEE=y
# CONFIG_OPTEE=y

# Build (takes ~10-20 minutes)
make -j$(nproc) \
    ARCH=arm64 \
    CROSS_COMPILE=aarch64-linux-gnu-

ls arch/arm64/boot/
# Image       ← kernel image
# Image.gz    ← compressed kernel

echo "=== Linux kernel built ==="
```

---

## Part 10: Build Root Filesystem with OP-TEE Tools

```bash
cd ~/trustzone-lab

# Build BusyBox for minimal rootfs
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xf busybox-1.36.1.tar.bz2
cd busybox-1.36.1

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
# Enable static linking (simpler for embedded)
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

cd ~/trustzone-lab

# Create rootfs directory structure
mkdir -p rootfs/{bin,sbin,etc,proc,sys,dev,lib,lib64,usr/lib,usr/sbin,tmp,lib/optee_armtz}

# Install BusyBox
make -C busybox-1.36.1 \
    ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    CONFIG_PREFIX=rootfs install

# Copy OP-TEE client library and supplicant
cp optee_client/out/export/usr/lib/libteec.so* rootfs/lib/
cp optee_client/out/export/usr/sbin/tee-supplicant rootfs/sbin/

# Copy xtest binary
cp out/xtest/xtest rootfs/bin/ 2>/dev/null || true

# Copy Trusted Applications to OP-TEE TA directory
# OP-TEE loads TAs from /lib/optee_armtz/
cp optee_os/out/arm-plat-vexpress/export-ta_arm64/uuid/*.ta rootfs/lib/optee_armtz/ 2>/dev/null || true
cp out/xtest/*.ta rootfs/lib/optee_armtz/ 2>/dev/null || true
cp out/ta/*.ta rootfs/lib/optee_armtz/ 2>/dev/null || true

# Create init script
cat > rootfs/etc/init.d/S01optee << 'EOF'
#!/bin/sh
# Start tee-supplicant daemon (needed for TA loading)
mkdir -p /lib/optee_armtz
/sbin/tee-supplicant &
echo "OP-TEE supplicant started"
EOF
chmod +x rootfs/etc/init.d/S01optee

# Create /init (PID 1)
cat > rootfs/init << 'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev 2>/dev/null || mdev -s

echo ""
echo "=== TrustZone Lab — QEMU ARM64 ==="
echo ""

# Start tee-supplicant for OP-TEE
/sbin/tee-supplicant &
sleep 1

# Check if OP-TEE driver loaded
if [ -e /dev/tee0 ]; then
    echo "✓ OP-TEE: /dev/tee0 present — Secure World connected!"
else
    echo "✗ OP-TEE: /dev/tee0 not present — check kernel config"
fi

echo ""
echo "Available commands:"
echo "  xtest              — Run all OP-TEE tests"
echo "  xtest -t regression — Run regression tests"
echo ""

exec /bin/sh
EOF
chmod +x rootfs/init

# Package rootfs as CPIO initramfs
cd rootfs
find . | cpio -o -H newc | gzip > ../out/rootfs.cpio.gz
cd ..

echo "=== Root filesystem created: out/rootfs.cpio.gz ==="
ls -lh out/rootfs.cpio.gz
```

---

## Part 11: Create the ATF FIP (Firmware Image Package)

ATF's **FIP (Firmware Image Package)** bundles all firmware stages into one signed package:

```
FIP = [ BL2 | BL31 (ATF) | BL32 (OP-TEE) | BL33 (U-Boot) ]
         ↑ Each piece can be individually verified
```

```bash
cd ~/trustzone-lab/atf

# Build the FIP creation tool
make -C tools/fiptool

# Create the FIP image combining all stages
./tools/fiptool/fiptool create \
    --tb-fw build/qemu/debug/bl2.bin \
    --soc-fw build/qemu/debug/bl31.bin \
    --tos-fw ../optee_os/out/arm-plat-vexpress/core/tee-header_v2.bin \
    --tos-fw-extra1 ../optee_os/out/arm-plat-vexpress/core/tee-pager_v2.bin \
    --tos-fw-extra2 ../optee_os/out/arm-plat-vexpress/core/tee-pageable_v2.bin \
    --nt-fw ../u-boot/u-boot.bin \
    out/fip.bin

echo "=== FIP image created ==="
ls -lh out/fip.bin

# Inspect the FIP contents:
./tools/fiptool/fiptool info out/fip.bin
```

---

## Part 12: Run the Complete Stack in QEMU

### Assemble All Pieces

```bash
cd ~/trustzone-lab

# Copy required files to out/ directory
cp atf/build/qemu/debug/bl1.bin out/
cp linux/arch/arm64/boot/Image out/

ls -lh out/
# bl1.bin        ← ATF BL1 (runs first in QEMU ROM)
# fip.bin        ← FIP: BL2 + BL31(ATF) + BL32(OP-TEE) + BL33(U-Boot)
# Image          ← Linux kernel
# rootfs.cpio.gz ← Root filesystem with OP-TEE tools
```

### Boot Script

```bash
cat > ~/trustzone-lab/run_qemu.sh << 'SCRIPT'
#!/bin/bash
# run_qemu.sh — Boot the complete TrustZone stack

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
OUT="$SCRIPT_DIR/out"

echo "=== Starting QEMU TrustZone Lab ==="
echo "BL1  (ATF):   $OUT/bl1.bin"
echo "FIP  (all):   $OUT/fip.bin"
echo "Linux:        $OUT/Image"
echo "RootFS:       $OUT/rootfs.cpio.gz"
echo ""
echo "Press Ctrl+A X to exit QEMU"
echo "==============================="
echo ""

qemu-system-aarch64 \
    -machine virt,secure=on,gic-version=3 \
    -cpu cortex-a57 \
    -smp 2 \
    -m 1024 \
    -no-acpi \
    -bios "$OUT/bl1.bin" \
    -drive if=pflash,format=raw,index=1,file="$OUT/fip.bin" \
    -kernel "$OUT/Image" \
    -initrd "$OUT/rootfs.cpio.gz" \
    -append "console=ttyAMA0,38400 keep_bootcon root=/dev/ram rdinit=/init" \
    -nographic \
    -serial mon:stdio \
    -d unimp \
    2>&1 | tee "$OUT/boot.log"
SCRIPT

chmod +x ~/trustzone-lab/run_qemu.sh
echo "Boot script created: ~/trustzone-lab/run_qemu.sh"
```

### Run It!

```bash
cd ~/trustzone-lab
./run_qemu.sh
```

**What you will see (step by step):**

```
NOTICE:  BL1: v2.10(debug):v2.10
NOTICE:  BL1: Built: 2026-07-05
INFO:    BL1: RAM 0x0e042000 - 0x0e04a000
...
NOTICE:  BL2: v2.10(debug):v2.10
INFO:    Loading image id=1 at address 0xe040000   ← BL31 loading
INFO:    Loading image id=4 at address 0xe100000   ← OP-TEE (BL32) loading
INFO:    Loading image id=5 at address 0x4a000000  ← U-Boot (BL33) loading

NOTICE:  BL31: v2.10(debug):v2.10
NOTICE:  BL31: Secure Monitor installed at EL3    ← ATF BL31 running!

I/TC: OP-TEE version 4.2.0                        ← OP-TEE booting!
I/TC: Primary CPU initializing
I/TC: TEE RAM   RW [0x0e100000 - 0x0e500000]      ← Secure memory
I/TC: TA RAM    RW [0x0e500000 - 0x0f000000]
I/TC: Initialized                                  ← OP-TEE ready!

U-Boot 2024.04                                     ← U-Boot starting
=> (auto-boot)

[    0.000000] Booting Linux on physical CPU 0x0   ← Linux starting!
[    1.234567] optee: probing for OP-TEE
[    1.234890] optee: OP-TEE API revision: 2.0     ← OP-TEE driver found!

=== TrustZone Lab — QEMU ARM64 ===
✓ OP-TEE: /dev/tee0 present — Secure World connected!

/ #                                                 ← Shell prompt!
```

---

## Part 13: Practice Exercises

### Exercise 1: Verify the Full Stack (5 min)

Once you have the shell prompt:

```bash
# Inside QEMU shell:

# 1. Check OP-TEE device
ls /dev/tee*
# /dev/tee0  /dev/teepriv0

# 2. Check kernel loaded OP-TEE driver
dmesg | grep -i "optee\|tee"
# [    1.234] optee: probing for OP-TEE
# [    1.235] optee: OP-TEE API revision: 2.0
# [    1.236] tee_compat: /dev/tee0 created

# 3. Check secure memory mapping
cat /proc/iomem | grep -i "secure\|optee"

# 4. Run one hello_world test
xtest 1001    # runs the Hello World TA test
```

### Exercise 2: Run the Full Test Suite (20 min)

```bash
# Inside QEMU shell:

# Run ALL regression tests
xtest -t regression

# Expected output:
# regression 1001 OK       ← OS Basics
# regression 1002 OK       ← Crypto: AES
# regression 1003 OK       ← Crypto: RSA
# regression 4001 OK       ← Storage
# regression 6001 OK       ← Secure storage
# ...
# Result: xx tests passed, 0 tests failed

# Run only crypto tests
xtest -t regression 1000    # 1000-series = crypto

# Verbose output
xtest -v 1001
```

### Exercise 3: Write and Load Your Own Trusted Application (30 min)

Create a simple "Calculator TA" that does arithmetic in Secure World:

```bash
# On your HOST machine (not QEMU):
cd ~/trustzone-lab

mkdir -p my_calculator_ta/ta my_calculator_ta/host

# Create TA header (the contract)
cat > my_calculator_ta/ta/include/calc_ta.h << 'EOF'
#ifndef CALC_TA_H
#define CALC_TA_H

/* UUID for Calculator TA — generate your own at uuidgenerator.net */
#define TA_CALC_UUID \
    { 0xdeadbeef, 0x1234, 0x5678, \
      { 0xab, 0xcd, 0xef, 0x01, 0x23, 0x45, 0x67, 0x89 } }

/* Commands */
#define TA_CALC_CMD_ADD   0   /* params[0].value.a + params[0].value.b → params[1].value.a */
#define TA_CALC_CMD_MUL   1   /* params[0].value.a * params[0].value.b → params[1].value.a */

#endif
EOF

# Create the TA implementation (runs in SECURE WORLD)
cat > my_calculator_ta/ta/calc_ta.c << 'EOF'
#include <tee_internal_api.h>
#include <tee_internal_api_extensions.h>
#include "include/calc_ta.h"

/* Called when Normal World opens a session to this TA */
TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types,
    TEE_Param params[4], void **sess_ctx)
{
    (void)param_types; (void)params; (void)sess_ctx;
    DMSG("Calculator TA: session opened");
    return TEE_SUCCESS;
}

/* Called when Normal World sends a command */
TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx, uint32_t cmd_id,
    uint32_t param_types, TEE_Param params[4])
{
    uint32_t a, b, result;
    (void)sess_ctx;

    if (param_types != TEE_PARAM_TYPES(
        TEE_PARAM_TYPE_VALUE_INPUT,
        TEE_PARAM_TYPE_VALUE_OUTPUT,
        TEE_PARAM_TYPE_NONE, TEE_PARAM_TYPE_NONE))
        return TEE_ERROR_BAD_PARAMETERS;

    a = params[0].value.a;
    b = params[0].value.b;

    switch (cmd_id) {
    case TA_CALC_CMD_ADD:
        result = a + b;
        DMSG("SECURE WORLD: %u + %u = %u", a, b, result);
        break;
    case TA_CALC_CMD_MUL:
        result = a * b;
        DMSG("SECURE WORLD: %u * %u = %u", a, b, result);
        break;
    default:
        return TEE_ERROR_BAD_PARAMETERS;
    }

    params[1].value.a = result;
    return TEE_SUCCESS;
}

/* TA lifecycle — required entry points */
TEE_Result TA_CreateEntryPoint(void) { return TEE_SUCCESS; }
void TA_DestroyEntryPoint(void) {}
void TA_CloseSessionEntryPoint(void *sess_ctx) { (void)sess_ctx; }
EOF

# Create the Host application (runs in NORMAL WORLD - Linux)
cat > my_calculator_ta/host/main.c << 'EOF'
#include <stdio.h>
#include <tee_client_api.h>
#include "../ta/include/calc_ta.h"

int main(void)
{
    TEEC_Result res;
    TEEC_Context ctx;
    TEEC_Session sess;
    TEEC_Operation op;
    TEEC_UUID uuid = TA_CALC_UUID;
    uint32_t origin;

    printf("=== Calculator TA Demo ===\n");
    printf("This app runs in NORMAL WORLD (Linux)\n");
    printf("The calculation happens in SECURE WORLD (OP-TEE)\n\n");

    /* Step 1: Connect to TEE */
    res = TEEC_InitializeContext(NULL, &ctx);
    if (res != TEEC_SUCCESS) { printf("TEEC_InitializeContext failed\n"); return 1; }

    /* Step 2: Open session to Calculator TA */
    res = TEEC_OpenSession(&ctx, &sess, &uuid,
        TEEC_LOGIN_PUBLIC, NULL, NULL, &origin);
    if (res != TEEC_SUCCESS) { printf("TEEC_OpenSession failed: 0x%x\n", res); return 1; }

    printf("Session opened to Calculator TA in Secure World\n\n");

    /* Step 3: Call ADD command */
    memset(&op, 0, sizeof(op));
    op.paramTypes = TEEC_PARAM_TYPES(
        TEEC_VALUE_INPUT, TEEC_VALUE_OUTPUT, TEEC_NONE, TEEC_NONE);
    op.params[0].value.a = 42;
    op.params[0].value.b = 58;

    res = TEEC_InvokeCommand(&sess, TA_CALC_CMD_ADD, &op, &origin);
    if (res == TEEC_SUCCESS)
        printf("Secure World computed: 42 + 58 = %u\n", op.params[1].value.a);

    /* Step 4: Call MUL command */
    op.params[0].value.a = 7;
    op.params[0].value.b = 6;
    res = TEEC_InvokeCommand(&sess, TA_CALC_CMD_MUL, &op, &origin);
    if (res == TEEC_SUCCESS)
        printf("Secure World computed: 7 * 6 = %u\n", op.params[1].value.a);

    /* Step 5: Close session */
    TEEC_CloseSession(&sess);
    TEEC_FinalizeContext(&ctx);

    printf("\nDone. The numbers were never visible to Normal World OS!\n");
    return 0;
}
EOF

echo "Calculator TA source created. Build with:"
echo "  cd my_calculator_ta/ta && make TA_DEV_KIT_DIR=..."
echo "  cd my_calculator_ta/host && make TEEC_EXPORT=..."
```

### Exercise 4: Observe the SMC Call (Deep Debug)

Enable ATF debug logging to see EVERY SMC call between Linux and Secure World:

```bash
# Rebuild ATF with maximum logging
cd ~/trustzone-lab/atf
make -j$(nproc) \
    CROSS_COMPILE=aarch64-linux-gnu- \
    ARCH=aarch64 PLAT=qemu \
    SPD=opteed DEBUG=1 \
    LOG_LEVEL=50 \
    ENABLE_BACKTRACE=1 \
    bl1 bl2 bl31

# Rebuild FIP and run
# In QEMU output, you will see:
# INFO:    BL31: Received SMC call 0xb2000000 (OP-TEE fast call)
# INFO:    BL31: Forwarding to OP-TEE...
# This is EXACTLY what happens in SC7180/SC7280 when your QFPROM driver
# calls arm_smccc_smc() to read eFuses!
```

### Exercise 5: Secure Storage Test

OP-TEE provides an encrypted key-value store that survives reboots:

```bash
# Inside QEMU shell:

# Run secure storage tests
xtest -t regression 6000

# Test 6001: Write key to secure storage
# Test 6002: Read key back (proves persistence)
# Test 6003: Attempt to read from different session (proves isolation)

# What happens internally:
# Linux app → libteec → kernel TEE driver → SMC → ATF → OP-TEE
# OP-TEE encrypts data with a device key
# Sends encrypted blob to tee-supplicant (Normal World daemon)
# tee-supplicant stores encrypted blob in /data/tee/ (or /tmp/ for us)
# On next read: reverse the process, decrypt in Secure World only
```

### Exercise 6: Secure Boot Simulation (Advanced)

```bash
# Generate RSA keys for Secure Boot signing
cd ~/trustzone-lab/out

# Generate Root of Trust key pair (this is like what's blown into eFuses)
openssl genrsa -out rot_key.pem 2048
openssl rsa -in rot_key.pem -pubout -out rot_pubkey.pem

echo "Root of Trust private key: rot_key.pem (KEEP SECRET — like eFuse)"
echo "Root of Trust public key:  rot_pubkey.pem (can be public)"

# In a real Secure Boot flow (like SC7180):
# 1. rot_pubkey.pem hash is blown into QFPROM eFuse during manufacturing
# 2. BootROM reads eFuse, verifies BL1/XBL signature using this public key
# 3. If signature mismatch → boot rejected
# 4. Your QFPROM driver provides Linux-side access to read (non-secure) fuse data

# Simulate signing ATF BL2:
openssl dgst -sha256 -sign rot_key.pem -out bl2.sig build/qemu/debug/bl2.bin
echo "BL2 signed with Root of Trust key"

# In production ATF with TBB (Trusted Board Boot):
make -j$(nproc) \
    CROSS_COMPILE=aarch64-linux-gnu- \
    PLAT=qemu SPD=opteed DEBUG=1 \
    TRUSTED_BOARD_BOOT=1 \
    GENERATE_COT=1 \
    ROT_KEY=../out/rot_key.pem \
    bl1 bl2 bl31 certificates

ls build/qemu/debug/*.crt
# bl2.crt   ← BL2 certificate (signed by Root of Trust)
# bl31.crt  ← BL31 certificate
# soc_fw_key.crt  ← Key certificate chain
```

---

## Part 14: Understanding the Connection to Your Qualcomm Work

### How QEMU OP-TEE Maps to Qualcomm Production

| QEMU Practice (open source) | Qualcomm Production (your work) |
|-----------------------------|----------------------------------|
| ATF BL31 (open source) | Qualcomm's proprietary EL3 TZ Monitor |
| OP-TEE OS (open source) | QTEE (Qualcomm Trusted Execution Environment) |
| OP-TEE Client (libteec) | Qualcomm QSEE client library |
| arm_smccc_smc() call | Same arm_smccc_smc() call in QFPROM driver! |
| TA UUID | qcom,qseecom service UUID |
| /dev/tee0 | /dev/qseecom |
| tee-supplicant | Qualcomm's qseecomd daemon |
| TA signing key | Qualcomm's eFuse-backed signing key |
| FIP secure boot | XBL + QTEE secure boot chain |

**Key insight:** The `arm_smccc_smc()` call your QFPROM driver uses is IDENTICAL in structure to what you practice here. The SMC call numbers differ (Qualcomm uses different function IDs), but the mechanism is the same.

### SMC Call Number Comparison

```c
/* QEMU / OP-TEE SMC (what you practice) */
#define OPTEE_SMC_CALL_GET_OS_UUID    0xb2000000

/* Qualcomm production SMC (what QFPROM uses) */
#define QCOM_SCM_SVC_FUSE             0x12
#define QCOM_SCM_FUSE_READ            0x07
#define QCOM_SCM_CALL_ID              ((0x40000000) | (QCOM_SCM_SVC_FUSE << 8) | QCOM_SCM_FUSE_READ)

/* Both use the SAME CPU instruction: */
arm_smccc_smc(CALL_ID, arg1, arg2, arg3, arg4, arg5, arg6, arg7, &result);
```

---

## Part 15: Quick Reference Commands

```bash
# ─── Building ────────────────────────────────────────────────
# ATF
make -C atf CROSS_COMPILE=aarch64-linux-gnu- PLAT=qemu SPD=opteed DEBUG=1 bl1 bl2 bl31

# OP-TEE OS
make -C optee_os CROSS_COMPILE=aarch64-linux-gnu- PLATFORM=vexpress-qemu_armv8a DEBUG=1

# FIP
./atf/tools/fiptool/fiptool create --tb-fw bl2.bin --soc-fw bl31.bin --tos-fw tee-header.bin --nt-fw u-boot.bin fip.bin

# ─── Running ─────────────────────────────────────────────────
qemu-system-aarch64 -machine virt,secure=on,gic-version=3 \
    -cpu cortex-a57 -m 1024 -no-acpi -nographic \
    -bios bl1.bin -drive if=pflash,format=raw,index=1,file=fip.bin \
    -kernel Image -initrd rootfs.cpio.gz \
    -append "console=ttyAMA0 root=/dev/ram rdinit=/init"

# ─── Testing (inside QEMU) ────────────────────────────────────
xtest                           # run all tests
xtest 1001                      # Hello World TA
xtest -t regression 1000        # crypto tests
xtest -t regression 6000        # secure storage tests
dmesg | grep optee              # verify OP-TEE driver loaded
ls /dev/tee*                    # verify TEE devices

# ─── Debug ───────────────────────────────────────────────────
# ATF logs: add LOG_LEVEL=50 to make command
# OP-TEE logs: add CFG_TEE_CORE_LOG_LEVEL=4 to make command
# See SMC calls: ATF VERBOSE logging
cat out/boot.log | grep -E "SMC|BL3|OP-TEE|optee"
```

---

## Troubleshooting Common Problems

### Problem: "OP-TEE: /dev/tee0 not present"

```bash
# Check kernel config
zcat /proc/config.gz | grep CONFIG_OPTEE
# Must be: CONFIG_OPTEE=y

# Check dmesg for OP-TEE probe
dmesg | grep -i "optee\|tee"

# If not probing: OP-TEE did not start correctly
# → Check ATF log for "BL32" loading
# → Verify SPD=opteed was used in ATF build
```

### Problem: "TEEC_OpenSession failed: 0xffff0008"

```
0xffff0008 = TEEC_ERROR_NO_DATA = TA file not found
→ The .ta file is not in /lib/optee_armtz/
Solution: cp *.ta rootfs/lib/optee_armtz/ and rebuild rootfs
```

### Problem: "ATF BL2 fails to load OP-TEE"

```bash
# Check the FIP contents
./atf/tools/fiptool/fiptool info out/fip.bin
# Must show entries for: BL2, BL31, BL32 (OP-TEE), BL33 (U-Boot)

# If BL32 missing: re-create FIP with all --tos-fw* arguments
```

### Problem: "xtest: command not found"

```bash
# xtest binary not copied to rootfs
# Rebuild rootfs after building xtest:
cp optee_test/out/xtest/xtest rootfs/bin/
find rootfs -name "*.ta" -o -path "*/optee_test/*.ta" | xargs -I{} cp {} rootfs/lib/optee_armtz/
cd rootfs && find . | cpio -o -H newc | gzip > ../out/rootfs.cpio.gz
```
