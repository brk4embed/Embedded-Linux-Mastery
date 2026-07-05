# 13 — Qualcomm Debugging Tools & Techniques

> Qualcomm SoC debugging requires mastery of a specialized toolchain: QXDM/QCAT for diag logs, CoreSight for hardware tracing, ramdump/minidump for crash analysis, and JTAG/Trace32 for deep hardware debugging.

---

## Section Structure

```
13_Qualcomm_Debugging/
├── 01_QXDM_QCAT_Diag.md             ← Qualcomm diagnostic protocol, log filtering
├── 02_Ramdump_Minidump.md           ← Collecting and parsing crash dumps
├── 03_CoreSight_ETM.md              ← ARM CoreSight trace infrastructure on Snapdragon
├── 04_Trace32_JTAG.md               ← Lauterbach T32 CMM scripts for QCOM
├── 05_SMEM_Debug.md                 ← Shared memory (SMEM) and subsystem crash analysis
├── 06_QTEE_TrustZone_Debug.md       ← TrustZone/QTEE debugging techniques
├── 07_Coreboot_SC7180_Debug.md      ← YOUR specific debug experience
├── 08_ADB_ChromeOS_Debug.md         ← adb + ChromeOS debugging workflow
└── 09_Qualcomm_Interview_QA.md      ← 40 Qualcomm-specific interview questions
```

---

## Qualcomm Debug Ecosystem Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                 Qualcomm Debug Ecosystem                         │
│                                                                  │
│  ┌────────────────┐   ┌─────────────────┐   ┌───────────────┐  │
│  │  QXDM / QCAT   │   │   Trace32 T32   │   │  ADB / ADPL   │  │
│  │  (Diag logs)   │   │  (JTAG debug)   │   │  (Android/    │  │
│  │                │   │                 │   │   ChromeOS)   │  │
│  └───────┬────────┘   └────────┬────────┘   └──────┬────────┘  │
│          │ Diag protocol       │ JTAG/SWD           │ USB ADB   │
│  ┌───────▼────────────────────▼────────────────────▼─────────┐  │
│  │                    Qualcomm SoC (SC7180/SC7280)            │  │
│  │                                                            │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌────────────┐  │  │
│  │  │  APSS    │  │  MPSS    │  │  ADSP  │  │  QTEE/TZ   │  │  │
│  │  │  (Linux) │  │  (Modem) │  │ (Audio)│  │ (TrustZone)│  │  │
│  │  └──────────┘  └──────────┘  └────────┘  └────────────┘  │  │
│  │                                                            │  │
│  │  ┌────────────────────────────────────────────────────┐   │  │
│  │  │          CoreSight Debug Infrastructure             │   │  │
│  │  │  ETM (trace) → ETB (buffer) → TMC → TPIU → JTAG   │   │  │
│  │  └────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. QXDM / QCAT — Qualcomm Diagnostic Protocol

### What It Is
QXDM (Qualcomm eXtensible Diagnostic Monitor) is Qualcomm's proprietary log collection tool. It uses the **Diag protocol** over USB, UART, or network to stream real-time logs from all Qualcomm subsystems.

**QCAT** (Qualcomm Code Analysis Tool) is the offline log parser for `.qmdl2` and `.isf` files.

### Linux-Side: `/dev/diag` Interface

```bash
# Check if Diag device is exposed
ls /dev/diag*
# For ChromeOS with Qualcomm: may be /dev/diag or USB interface

# Diag device appears as USB interface class 0xFF (vendor-specific)
lsusb -v | grep -A5 "Vendor.*Qualcomm\|idVendor.*05c6"

# QRTR (Qualcomm IPC Router) for userspace diag access
cat /sys/bus/msm_ipc/devices/*/node_id   # IPC Router nodes

# Check if qrtr-ns (name service) is running
ps aux | grep qrtr
```

### Open Source Diag Tools

```bash
# qcomdiag — open source diag parser
git clone https://github.com/bkerler/edl
pip install pyserial pyusb

# diag-router — Linux userspace Diag routing daemon
# (part of ModemManager project)
mmcli -m 0 --verbose

# Using libdiag (Linaro project)
# Enables reading Qualcomm diag logs from Linux without QXDM
```

### Log Mask Configuration (Key Concept)

Diag protocol uses **log masks** to enable specific log categories:
```
Log Code ranges:
0x1000-0x1FFF: QCHAT
0x4000-0x4FFF: Modem (MPSS)
0x5000-0x5FFF: Applications (APSS/Linux)
0xB000-0xBFFF: ADSP
0x14000-0x14FFF: GPU/DSP

To enable UFS-related logs (custom code example):
LOG_CODE_UFS_COMMAND     = 0x5C9A
LOG_CODE_UFS_COMPLETION  = 0x5C9B
```

---

## 2. Ramdump & Minidump

### What Is a Ramdump?
When a Qualcomm SoC crashes (kernel panic, watchdog, subsystem crash), it can dump the entire RAM content to storage (eMMC, USB) for offline analysis.

**Minidump** is a lightweight variant that dumps only essential structures (saves space on devices with limited storage).

### Enabling Ramdump on Linux

```bash
# Check if ramdump is configured
zcat /proc/config.gz | grep -i "RAMDUMP\|MINIDUMP\|PSTORE"

# Typical kernel config for Qualcomm ramdump:
# CONFIG_QCOM_MEMORY_DUMP_V2=y
# CONFIG_QCOM_MINIDUMP=y
# CONFIG_PSTORE=y
# CONFIG_PSTORE_RAM=y

# Check crash dump location after reboot
ls /sys/bus/platform/drivers/qcom_ramdump/
ls /dev/ramdump*

# On Android/ChromeOS: ramdumps land in /data/tombstones/ or /var/log/
```

### Minidump Segments

```bash
# Minidump registers segments with the TZ/EL3 firmware
# Each subsystem registers what to include:
# - APSS: task_struct, kernel stacks, important global vars
# - MPSS: modem state, IPC buffers
# - ADSP: audio state

# Linux kernel minidump registration (driver example):
#include <soc/qcom/minidump.h>

struct md_region md_entry;
strlcpy(md_entry.name, "MYDRIVER", sizeof(md_entry.name));
md_entry.virt_addr = (uintptr_t)my_buffer;
md_entry.phys_addr = virt_to_phys(my_buffer);
md_entry.size = sizeof(my_buffer);
msm_minidump_add_region(&md_entry);
```

### Parsing a Ramdump with `crash`

```bash
# Install crash tool
apt install crash

# Get vmlinux with debug symbols (must match kernel version exactly)
# From Qualcomm BSP: out/target/product/<board>/obj/KERNEL_OBJ/vmlinux

# Parse ramdump
crash vmlinux /path/to/ramdump/DDR_0.BIN@0x80000000,DDR_1.BIN@0x100000000

# Inside crash:
crash> bt           # backtrace of crashed task
crash> log          # kernel message buffer
crash> ps           # all processes at time of crash
crash> files        # open files of crashed task
crash> kmem -s      # slab allocator state
crash> rd -a <addr> # read memory
crash> sym <addr>   # resolve address to symbol
crash> dis <func>   # disassemble function
crash> struct task_struct.comm <addr>  # read struct field
```

### Ramdump to ELF Conversion

```bash
# Qualcomm provides ram_dump_parse (proprietary) or use open-source:
# dumpextractor (GitHub: andersson/dumpextractor)
python3 dumpextractor.py --debug ramdump_dir/ --output coredump.elf

# Then use crash:
crash vmlinux coredump.elf
```

---

## 3. CoreSight / ETM — Hardware Trace on Snapdragon

### Architecture Overview

```
CPU Core
  │
  ├── ETM (Embedded Trace Macrocell)
  │     └── Generates instruction/data trace stream
  │
  └── Cross-Trigger Interface (CTI)
        └── Halt/trigger coordination

ETM output → Trace Funnel → ETB (Embedded Trace Buffer)
                               └── 64KB-256KB on-chip RAM
                              OR
                          TMC (Trace Memory Controller)
                               └── Streams to DDR (larger traces)
                                    └── TPIU → External JTAG probe
```

### Linux CoreSight Support

```bash
# Check CoreSight subsystem
ls /sys/bus/coresight/devices/
# Should list: etm0, etm1, ..., etf0, tmc_etf0, funnel0, replicator, tpiu

# Enable ETM tracing (function flow trace)
echo 1 > /sys/bus/coresight/devices/etm0/enable_source

# Configure trace filter (trace only specific address range)
echo 0 > /sys/bus/coresight/devices/etm0/addr_range_idx
echo 0xffffffc000800000 > /sys/bus/coresight/devices/etm0/addr_start
echo 0xffffffc000900000 > /sys/bus/coresight/devices/etm0/addr_stop
echo 1 > /sys/bus/coresight/devices/etm0/addr_type  # 1=range filter

# Read trace from ETB
cat /dev/tmc_etf0 > /tmp/trace.bin

# Decode with OpenCSD
ocsdecodetest -ss_dir /sys/bus/coresight/devices/etm0/ \
              -trace_file /tmp/trace.bin
```

### perf + CoreSight Integration

```bash
# Record with CoreSight ETM via perf
perf record -e cs_etm/@tmc_etf0/ --per-thread -- my_program

# Decode trace
perf report --stdio

# AutoFDO profile from ETM trace (for GCC/Clang PGO)
perf inject --itrace=l64u200ns -i perf.data -o perf_inject.data
```

---

## 4. Trace32 / JTAG

### T32 CMM Script for SC7180/SC7280

```
; sc7180_debug.cmm — Trace32 script for SC7180 bring-up debug

RESET
SYStem.CPU SC7180          ; or SC7280
SYStem.JtagClock 10MHz
SYStem.Option DUALPORT ON
SYStem.Option ResBreak OFF
SYStem.Up

; Load symbols
Data.LOAD.Elf vmlinux /NoCODE /GNU

; Set breakpoint at kernel entry
Break.Set start_kernel /HARD

; Continue
Go

; After break — examine state
Register.View
Data.List

; Read register
Print R(PC)
Print R(SP)
Print R(X19)

; Read memory
Data.Dump 0xFFFFFFC000000000 0x100

; Print task list
v/task *task_ptr
```

### Connecting T32 to a Qualcomm JTAG Target

```
Hardware setup:
1. Identify JTAG connector on board (usually 10-pin ARM standard)
2. Connect Lauterbach JTAG probe (LA-7730, LA-3505, PowerDebug)
3. Power on board in Maskrom/JTAG-hold mode (if supported)
   - SC7180: No Maskrom JTAG hold — must halt via T32 after BootROM
4. Launch T32 and load the platform CMM script

Qualcomm-specific JTAG notes:
- QTEE (TrustZone) runs at EL3/EL1-S — T32 can debug both NS and Secure world
- After ATF/QTEE init, SCR_EL3.NS=1 for Non-Secure world
- Use SYStem.Option TrustZone ON to switch between worlds
- QSEE (Qualcomm Secure Execution Env) has anti-debug protections in production fuses
```

### Common T32 Debug Workflows

```
; 1. Debug boot hang (SPL/U-Boot not reaching UART)
SYStem.Up
Go.Indirect start_spl_main   ; set SW break at SPL entry
; If SPL hangs: check DDR init, check clock config

; 2. Kernel NULL pointer — find exact crash location  
Data.LOAD.Elf vmlinux
Break.Set stext /HARD        ; catch kernel entry
Go
; Wait for panic, then:
Register.View                ; shows exact PC at crash
Data.List                    ; shows source line
v/task current               ; shows task_struct of crashed process

; 3. Read QFPROM eFuse via JTAG (before Linux boots)
Data.In 0x00780000++0x100    ; QFPROM base address on SC7180
```

---

## 5. SMEM — Shared Memory Debug

### What is SMEM?
Qualcomm SoCs use a shared memory region accessible by all processors (APSS/Linux, MPSS/Modem, ADSP, TZ). It contains boot information, remote processor status, and IPC state.

```bash
# Linux SMEM driver: drivers/soc/qcom/smem.c
# SMEM base address: platform-specific, found in DT
# grep "qcom,smem" in arch/arm64/boot/dts/qcom/

# Read SMEM from Linux
cat /sys/kernel/debug/qcom_smem/info     # if debugfs entry exists

# Subsystem crash status via SMEM
cat /sys/bus/platform/devices/*/state   # remoteproc states
cat /sys/class/remoteproc/remoteproc*/state

# If modem (MPSS) crashes:
dmesg | grep -i "mpss\|subsystem\|pil\|remoteproc"
# Look for: "msm_subsystem_restart: Restarting mpss"

# Collect MPSS ramdump when it crashes
cat /dev/ramdump_modem > mpss_dump.bin
```

---

## 6. QTEE / TrustZone Debug

### QTEE Architecture (SC7180/SC7280)

```
EL3: ARM Trusted Firmware (ATF) — BL31
     └── Handles SMC calls, switches between secure/non-secure

EL1-S: QTEE (Qualcomm Trusted Execution Environment)
       └── Qualcomm's TrustZone OS (proprietary)
       └── Hosts TAs (Trusted Applications): DRM, biometrics, crypto

EL1-NS: Linux kernel (non-secure world)
        └── Communicates with QTEE via SMC calls through ATF
```

### Debugging TrustZone from Linux

```bash
# QSEECOM interface (Linux ↔ QTEE)
ls /dev/qseecom

# Send command to TA (via QSEECOM ioctl)
# Usually via Qualcomm vendor lib, not directly

# TZ log via TZDBG
cat /sys/kernel/debug/tzdbg/log         # TZ debug log
cat /sys/kernel/debug/tzdbg/qsee_log    # QSEE (QTEE userspace TA) log
cat /sys/kernel/debug/tzdbg/hyp_log     # Hypervisor log (if enabled)

# TZ crash dump
# If TZ crashes: system usually reboots immediately
# TZ crash info may be in /proc/last_kmsg or ramdump

# Check ATF (BL31) version
cat /sys/firmware/devicetree/base/firmware/scm/qcom,soc-id 2>/dev/null
strings /proc/device-tree/firmware/scm/compatible 2>/dev/null
```

### SMC Call Debugging (your SC7180 experience)

```c
/* Linux kernel SMC call to QTEE/ATF */
#include <linux/arm-smccc.h>

struct arm_smccc_res res;

/* Example: QFPROM read via SMC */
arm_smccc_smc(
    QCOM_SCM_SVC_FUSE,          /* service ID */
    QCOM_SCM_FUSE_READ,         /* command ID */
    row_address, mask, 0, 0,    /* args */
    0, 0,
    &res
);

if (res.a0 != 0)
    dev_err(dev, "SCM call failed: %ld\n", res.a0);
```

---

## 7. Your SC7180/SC7280 Debug Experience

### QFPROM eFuse Debug Workflow

```bash
# 1. Verify QFPROM memory map in DT
grep -r "qcom,qfprom\|qcom,sc7180-qfprom" \
    arch/arm64/boot/dts/qcom/

# 2. Check nvmem provider registered
cat /sys/bus/nvmem/devices/*/name
# Should show: qfprom0 (or similar)

# 3. Read a specific fuse cell
# Cell is defined in DT as nvmem-cells
cat /sys/bus/nvmem/devices/qfprom0/nvmem

# 4. Debug probe failure
echo 8 > /proc/sys/kernel/printk
modprobe qfprom
dmesg | grep -i "qfprom\|nvmem\|probe"

# 5. If probe fails: check register base address matches DT vs TRM
# Common mistake: SC7180 QFPROM base ≠ SC7280 QFPROM base
```

### Coreboot Bring-Up Debug (Serial Trace)

```
# SC7180/SC7280 Coreboot debug output:
# 1. Enable UART in early init (before DDR):
#    src/soc/qualcomm/sc7180/early_init.c
#    qcom_uart_init() must be called before cbmem

# 2. Enable verbose Coreboot logging:
#    menuconfig → Debugging → Console log level = SPEW (7)

# 3. Common SC7180 bring-up failures seen:
#    "QSPI init failed" → SPI clock config, chip select polarity
#    "QC Library init failed" → QTEE / QCLib binary mismatch
#    "DDR training timeout" → Check VDD_DDR rail, CLK source
#    "Depthcharge: vboot2 fail" → signature mismatch, wrong key

# 4. Coreboot debug console output (example from SC7280):
    QCOM: DDR_FREQ: 2133 MHz
    QCOM: DDRSS Frequency: 1066 MHz
    QCOM: PMIC PM8150 init done
    QCOM: TZ init done
    CBMEM: CBMEM base at 0x85e00000
    Payload: Depthcharge
    ChromeOS: VB2_SUCCESS
```

---

## 8. ADB / ChromeOS Debugging

### ChromeOS Developer Mode Debug

```bash
# Enable Developer Mode (physical: Esc+Refresh+Power, then Space)
# Or via crossystem on already-rooted device:

# Check system state
crossystem | grep -E "dev_boot|mainfw_type|cros_debug"

# Enable full debug output
crossystem cros_debug=1

# Access ChromeOS serial console
# Connect via UART header on board (115200, 8N1)
# Or via SSH (Developer Mode enables SSH server)

# adb over USB
adb shell dmesg | grep -i error
adb shell cat /proc/last_kmsg   # last boot's kernel log
adb bugreport > bugreport.zip   # full system state

# Coreboot / BIOS log from within ChromeOS
cbmem -c        # read coreboot console log
cbmem -l        # list cbmem tables

# Check verified boot state
vbutil_kernel --verify /dev/sda4
```

### Crosh / Chrome Shell Commands

```bash
# In crosh (Ctrl+Alt+T in Chrome browser):
crosh> shell              # drop to bash (Developer Mode only)
crosh> ping 8.8.8.8
crosh> top                # process list

# From bash in ChromeOS:
cat /sys/kernel/debug/cros_ec/console_log  # EC (embedded controller) log
ectool version                              # EC firmware version
mosys platform name                         # board identification
```

---

## 9. Qualcomm Interview Q&A

| Level | Question | Expected Answer |
|-------|----------|-----------------|
| **Basic** | What is the Diag protocol? | Qualcomm proprietary protocol for real-time log streaming from all SoC subsystems over USB/UART |
| **Basic** | What is SMEM? | Shared memory region accessible by all processors (APSS, MPSS, ADSP) for IPC and status sharing |
| **Intermediate** | How do you collect a ramdump after a kernel panic on SC7180? | Enable `CONFIG_QCOM_MEMORY_DUMP_V2`, collect from `/dev/ramdump*`, parse with `crash` + vmlinux |
| **Intermediate** | What is CoreSight ETM? How do you use it for debugging? | ARM hardware trace infrastructure; ETM generates instruction trace → ETB/TMC → JTAG or perf |
| **Advanced** | You see "MPSS subsystem restarting" in dmesg. How do you debug it? | Collect MPSS ramdump from `/dev/ramdump_modem`, check SMEM state, look at QRTR/QSEECOM logs, examine IPC errors |
| **Advanced** | Describe the SC7180 secure boot chain. | BootROM → QTEE (EL3 ATF) → QCLib/XBL → Coreboot (non-secure) → Depthcharge → Kernel; each stage verified by eFuse-rooted keys via QFPROM |
| **Expert** | How would you debug a TrustZone crash that causes a silent system reset? | Enable TZDBG, collect TZ log via `/sys/kernel/debug/tzdbg/log`, use T32 with TrustZone mode to debug EL3, check ATF panic handler output |
| **Expert** | In your QFPROM driver, how did you handle the SMC call failure case? What retry logic did you implement? | [Your specific answer based on implementation details] |

---

## Quick Reference: Qualcomm Debug Commands

```bash
# === System Info ===
cat /proc/device-tree/model                    # board/SoC model
cat /sys/devices/soc0/soc_id                   # Qualcomm SoC ID
cat /sys/devices/soc0/serial_number            # fused serial (via QFPROM)

# === Subsystems ===
cat /sys/class/remoteproc/remoteproc*/state     # MPSS/ADSP/WCSS state
cat /sys/bus/platform/devices/*/subsystem_state # all subsystem states

# === CoreSight ===
ls /sys/bus/coresight/devices/
echo 1 > /sys/bus/coresight/devices/etm0/enable_source

# === QTEE / TrustZone ===
cat /sys/kernel/debug/tzdbg/log
cat /sys/kernel/debug/tzdbg/qsee_log

# === QFPROM ===
cat /sys/bus/nvmem/devices/*/nvmem | xxd | head

# === Boot Info (Coreboot/Depthcharge) ===
cbmem -c        # Coreboot console log
cbmem -l        # Coreboot memory tables
crossystem      # Chromebook/ChromeOS system state

# === Crash / Ramdump ===
ls /dev/ramdump*
crash vmlinux /dev/ramdump_apss
```
