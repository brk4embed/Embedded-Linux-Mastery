# Complete Board Bringup Guide — From Dead Board to Running Linux

> **Ravi's Context:** You've done SC7180/SC7280 bringup at Eximietas.  
> This guide formalizes that knowledge AND teaches you how to bring up  
> a completely new board (like a custom RK3588 design) from absolute zero.

---

## What is "Board Bringup"?

Imagine you receive a PCB from the hardware team. The chip is soldered. No software runs.  
Nothing on the screen. No console output. The board is "dead."

Board bringup = making that board run Linux, step by step.

```
Power On (nothing works)
    ↓ You enable: voltage rails, clock, UART console
    ↓ You see: "U-Boot SPL" (first sign of life!)
    ↓ You enable: DDR, eMMC/SD
    ↓ You see: "U-Boot" (bootloader working!)
    ↓ You enable: DT, kernel config
    ↓ You see: "Starting kernel..."
    ↓ You enable: rootfs, init
    ↓ You reach: login prompt (COMPLETE!)
```

---

## Phase 1: Lab Setup and Safety

### Equipment Checklist

```
REQUIRED:
  □ UART-to-USB adapter (CP2102 or CH340, 3.3V logic)
  □ Multimeter (voltage probing)
  □ Oscilloscope (optional but invaluable)
  □ Jumper wires and probe clips
  □ Stable 5V/3A power supply (not phone charger)
  □ Laptop with Linux (for development)
  □ microSD card (Class 10+)
  □ Spare components matching board BOM

NICE TO HAVE:
  □ Logic analyzer (8-channel, e.g., Saleae Logic)
  □ JTAG/SWD debugger (for Cortex-A: OpenOCD; for Qualcomm: Trace32)
  □ USB hub with power delivery monitor
  □ Heat gun + flux (for rework)
```

### UART Connection (Always First Step)

```
Most ARM SoCs expose debug UART at 115200 or higher baud.
Find it from:
  1. Datasheet (look for "debug UART", "console UART")
  2. Reference board schematic (search "debug" or "uart")
  3. Existing kernel DTS file

For Radxa Rock 5B+:
  UART2 on 40-pin header
  Pin 6:  GND   (black wire)
  Pin 8:  TX    (connect to RX of adapter)  
  Pin 10: RX    (connect to TX of adapter)
  Baud:   1500000 (1.5 Mbaud — non-standard!)

Connect UART adapter, then:
  sudo screen /dev/ttyUSB0 1500000
  # OR
  picocom -b 1500000 /dev/ttyUSB0
  # OR
  minicom -b 1500000 -D /dev/ttyUSB0

Power on the board. If ANYTHING appears, celebrate! Even garbage bytes
mean the UART path is correct (baud rate may be wrong).
```

---

## Phase 2: Power Sequencing

### Verify Power Rails (Before Anything Else)

```bash
# Use multimeter before powering on anything:
# 1. Continuity check: no shorts on key rails (3.3V, 1.8V, core voltage)
# 2. Resistance to ground: core VDD should be ~1-10 ohms (not 0!)

# Typical RK3588 power rails:
# VCC5V0_SYS:  5.0V  (from charger/USB-C)
# VCC_3V3:     3.3V  (IO)
# VCCIO_1V8:   1.8V  (DDR IO)
# VDD_CPU:     0.75-1.0V (adjustable, from PMIC)
# VDD_GPU:     0.75-1.0V (adjustable, from PMIC)
# VCC_DDR:     varies (DDR4: 1.2V, LPDDR5: 1.05V)
```

### Power Sequencing Diagram (RK3588 typical)

```
PMIC powers up in order:
  1. VCC_3V3     (PMIC itself needs 5V → 3.3V first)
  2. VCC_DDR     (DDR needs stable voltage before init)
  3. VDD_CPU     (CPU core voltage)
  4. VDD_GPU     (GPU core voltage)
  5. VCC_NPU     (NPU core voltage)
  6. VCCIO_1V8   (IO voltage)
  
Total time: ~5-10ms from power-on to SPL starts
```

---

## Phase 3: U-Boot SPL Bringup

### Understanding What SPL Does

SPL (Secondary Program Loader) is the first code that runs from writeable storage.
It must fit in a tiny amount of SRAM (typically 256KB on RK3588).

```c
/* Simplified SPL flow (arch/arm/lib/crt0_64.S → common/spl/spl.c): */

void board_init_f(ulong dummy)
{
    /* 1. Initialize CPU (clocks, cache) */
    cpu_init();
    
    /* 2. Initialize debug UART — THIS IS WHY WE SEE FIRST OUTPUT */
    debug_uart_init();
    
    /* 3. Initialize DDR — most complex part */
    dram_init();        /* calls vendor DDR initialization code */
    
    /* 4. Copy full U-Boot from storage to DDR */
    spl_load_image_from_mmc(...);   /* or SPI, NAND, etc. */
}

/* After board_init_f: jump to full U-Boot in DDR */
```

### Porting SPL to a New RK3588 Board

```bash
# Base your port on closest existing board:
# For Radxa Rock 5B+ → use rock5b as reference
ls configs/ | grep rock5
# rock5b_defconfig
# rock5b_rk3588_defconfig

# Create your board
cp configs/rock5b_defconfig configs/my_rk3588_board_defconfig
vim configs/my_rk3588_board_defconfig

# Key SPL configuration items:
CONFIG_SPL=y
CONFIG_SPL_SERIAL=y            # UART in SPL
CONFIG_SPL_MMC=y               # Load U-Boot from eMMC/SD in SPL
CONFIG_SPL_DM_SEQ_ALIAS=y      # DM aliases

# Create board directory:
mkdir -p board/mycompany/my_rk3588/
cat > board/mycompany/my_rk3588/my_rk3588.c << 'EOF'
#include <common.h>

int board_init(void) {
    return 0;
}

int dram_init(void) {
    return fdtdec_setup_mem_size_base();   /* read from DT */
}
EOF

# Create device tree:
cp arch/arm/dts/rk3588-rock-5b.dts arch/arm/dts/rk3588-my-board.dts
# Edit to match YOUR hardware schematic

# Build:
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- my_rk3588_board_defconfig
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

### DDR Initialization Deep Dive

DDR init is the hardest part of bringup:

```c
/*
 * Why DDR init is hard:
 * - Training: DDR needs calibration (DQ training, timing calibration)
 * - Vendor blob: Most SoC vendors provide binary DDR init code
 * - Parameters: PCB layout affects timing (trace length → delays)
 *
 * For Rockchip RK3588:
 * - DDR init is in: arch/arm/mach-rockchip/sdram.c
 * - Vendor provides: lpddr4_init.c, lpddr5_init.c
 * - Timing parameters: set in DT (ddr-config-con, etc.)
 */

/* If DDR fails, you see: */
// "DRAM init failed!" or just silence (UART stops working)

/* Debug DDR issues:
 * 1. Check power (VDDQ_DDR stable at correct voltage?)
 * 2. Check clocks (DDR clock source enabled?)
 * 3. Try reducing DDR speed (find "ddr_speed_bin" in DTS)
 * 4. Compare with known-good board's DDR parameters
 * 5. Oscilloscope: check data lines for training patterns
 */

/* Typical DDR speeds to try:
 * LP5: 6400 → try 4266 → try 3200 (reduce until stable)
 * LP4X: 4266 → try 3200 → try 2400
 */
```

---

## Phase 4: Kernel Bringup

### Step-by-Step Kernel Enable Sequence

Never try to enable everything at once. This is the correct order:

```
Step 1: UART + printk working
  Enable: CONFIG_SERIAL_8250 or CONFIG_SERIAL_AMBA_PL011
  Expected: kernel boot messages on console
  
Step 2: CPU topology
  Enable: CONFIG_SMP, correct number of CPUs in DTS
  Expected: "Brought up N CPUs" in dmesg
  
Step 3: Interrupts
  Enable: GIC (Generic Interrupt Controller) in DTS
  Expected: no "BUG: No IRQ handler" messages
  
Step 4: Clock framework
  Enable: clock controller driver for your SoC
  Expected: clocks shown in /sys/kernel/debug/clk/
  
Step 5: PMIC / Regulator
  Enable: regulator driver, regulator DTS nodes
  Expected: voltages reported in dmesg
  
Step 6: Storage (eMMC/SD)
  Enable: SDHCI or vendor MMC controller driver
  Expected: /dev/mmcblk0 appears
  
Step 7: USB
  Enable: DWC3 or vendor USB controller
  Expected: USB devices enumerated
  
Step 8: Networking
  Enable: ethernet or WiFi driver
  Expected: network interface appears
  
Step 9: Display
  Enable: DRM/KMS driver
  Expected: display output
  
Step 10: Everything else
  Enable remaining: GPU, NPU, camera, audio, etc.
```

### Creating the Device Tree from Scratch

```bash
# Start from SoC-level DTS:
# - Contains all IP blocks inside the SoC chip
# - Vendor provides this

# Your board DTS adds:
# - Physical connectors (buttons, LEDs, sensors)
# - Which regulator powers which IP
# - Which pins are muxed to what function
# - Board-specific timing parameters

cat > arch/arm64/boot/dts/rockchip/rk3588-my-board.dts << 'EOF'
/dts-v1/;

#include "rk3588.dtsi"            /* SoC base: all IP blocks */
#include "rk3588-pinctrl.dtsi"   /* Pin definitions */

/ {
    model = "MyCompany RK3588 Board v1";
    compatible = "mycompany,rk3588-board", "rockchip,rk3588";

    chosen {
        /* This is where kernel looks for console and rootfs */
        bootargs = "earlycon=uart8250,mmio32,0xfeba0000 console=ttyS2,1500000";
        stdout-path = "serial2:1500000n8";
    };
    
    memory@0 {
        device_type = "memory";
        reg = <0x0 0x00000000 0x0 0x80000000  /* 2GB */
               0x0 0x100000000 0x0 0x80000000>; /* 2GB high */
    };

    /* Power supply from 12V DC input */
    vcc12v_dcin: vcc12v-dcin {
        compatible = "regulator-fixed";
        regulator-name = "vcc12v_dcin";
        regulator-always-on;
        regulator-boot-on;
        regulator-min-microvolt = <12000000>;
        regulator-max-microvolt = <12000000>;
    };

    /* 5V derived from 12V */
    vcc5v0_sys: vcc5v0-sys {
        compatible = "regulator-fixed";
        regulator-name = "vcc5v0_sys";
        regulator-always-on;
        regulator-boot-on;
        regulator-min-microvolt = <5000000>;
        regulator-max-microvolt = <5000000>;
        vin-supply = <&vcc12v_dcin>;
    };
};

/* PMIC on I2C0 — controls CPU/GPU voltage */
&i2c0 {
    status = "okay";
    
    rk806: pmic@23 {
        compatible = "rockchip,rk806";
        reg = <0x23>;
        /* ... regulator children ... */
    };
};

/* Debug UART */
&uart2 {
    status = "okay";
};

/* eMMC */
&sdhci {
    status = "okay";
    bus-width = <8>;        /* 8-bit eMMC */
    non-removable;          /* eMMC is soldered */
    cap-mmc-highspeed;
    mmc-hs200-1_8v;
    pinctrl-names = "default";
    pinctrl-0 = <&emmc_bus8 &emmc_clk &emmc_cmd>;
};
EOF
```

---

## Phase 5: Debugging Bringup Failures

### The Bringup Debug Toolkit

```bash
# 1. earlycon: console before UART driver initializes
# In bootargs:
earlycon=uart8250,mmio32,0xFEB50000 console=ttyS2,1500000

# 2. initcall_debug: show every kernel init function
initcall_debug loglevel=8

# 3. no_console_suspend: keep console during suspend
no_console_suspend

# 4. panic=5: auto-reboot 5 seconds after panic
panic=5

# 5. ftrace early: trace before /sys is mounted
ftrace=function trace_buf_size=256M
```

### Common Failures and Solutions

| Symptom | Likely Cause | Debug Steps |
|---------|-------------|------------|
| No console output at all | Wrong UART pins, wrong baud rate, power issue | Check 3.3V on UART pins, try different baud rates |
| "DRAM init failed" in SPL | DDR training failure | Check DDR power, reduce DDR speed in DTS |
| Kernel hangs after "Starting kernel..." | DT bad (no console, wrong interrupt controller) | Rebuild with `initcall_debug`, check DTS stdout-path |
| "Unable to handle kernel paging request" | NULL dereference in driver | KASAN build, check call stack |
| Driver probe fails with -EPROBE_DEFER | Clock/regulator not ready yet | Normal — will retry. If loops forever: check dependency |
| No `/dev/mmcblk0` | Wrong MMC driver, DT issue | Check `dmesg | grep mmc`, check clock DT node |
| Network not working | PHY not initialized | Check MDIO/PHY DTS, PHY reset GPIO |
| Display blank | PMIC not providing panel voltage | Check panel regulator DTS, HDMI CEC |

### Using JTAG for Bringup (When UART Doesn't Work)

```bash
# JTAG lets you halt the CPU and inspect registers even before UART works

# For Rockchip RK3588 with OpenOCD:
# Install
sudo apt-get install openocd

# Connect JTAG adapter to board header:
# (Check board schematic for JTAG header — usually 10-pin)
# TDI, TDO, TMS, TCK, TRST

# OpenOCD config for RK3588:
cat > rk3588.cfg << 'EOF'
source [find interface/jlink.cfg]
transport select jtag
source [find target/rockchip_rk3588.cfg]
EOF

openocd -f rk3588.cfg

# In another terminal:
telnet localhost 4444
# > halt          ← stop CPUs
# > reg cpsr      ← read registers
# > step          ← execute one instruction
# > mem2array data 32 0xFF010000 64   ← read memory
```

---

## Phase 6: Complete Bringup Log — Radxa Rock 5B+ Reference

```
What you should see (annotated):

U-Boot SPL 2024.01 (Jan 01 2024 - 00:00:00 +0000)
                                                    ← SPL started (DDR init DONE)
Trying to boot from MMC1
                                                    ← Loading U-Boot from eMMC
U-Boot 2024.01 (Jan 01 2024 - 00:00:00 +0000)     ← Full U-Boot started!

CPU:   Rockchip RK3588
Model: Radxa ROCK 5B+
DRAM:  16 GiB
Core:  269 devices, 24 uclasses, devicetree: separate
MMC:   mmc@fe2c0000: 1, mmc@fe2e0000: 2
Loading Environment from MMC...
Hit any key to stop autoboot: 3                    ← Press key to enter U-Boot shell

(in autoboot): Loading kernel...
   Loading Device Tree...
   Verifying Checksum ... OK
## Flattened Device Tree blob at 0x08300000
## Loading kernel from FIT Image at 0x00000000...
Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x412fd050]
[    0.000000] Linux version 6.6.0 (gcc 11.4.0)
[    0.000000] Machine model: Radxa ROCK 5B+           ← DT matched!
[    0.000000] earlycon: uart8250 at MMIO32 ...
[    0.000000] KASLR disabled due to lack of seed
[    0.000000] Memory: 16384M - 32M (16352M available)
[    0.090000] SMP: Brought up 8 CPUs                  ← All cores online
[    0.150000] rk3588-pinctrl: probed                  ← Pinctrl ready
[    0.160000] rk3588-clk: probed                      ← Clocks ready
[    0.200000] rk806 pmic: probed                      ← PMIC ready
[    0.250000] mmc0: Card inserted                     ← eMMC found
[    0.300000] EXT4-fs: mounted                        ← Rootfs mounted!

Starting systemd...
[   OK  ] Reached target Graphical Interface
```

---

## Lab Exercises

```bash
# Lab 1: UART bring-up on Radxa 5B+
# 1. Connect UART-USB adapter (3 wires: GND, TX, RX)
# 2. picocom -b 1500000 /dev/ttyUSB0
# 3. Power on board
# 4. Observe complete boot log
# DELIVERABLE: Screenshot of complete boot log

# Lab 2: Boot from SD card with custom kernel
# 1. git clone --depth=1 https://github.com/torvalds/linux
# 2. make ARCH=arm64 rockchip_defconfig
# 3. Add: CONFIG_RADXA_ROCK5B=y (or similar)
# 4. make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j8 Image dtbs
# 5. Copy Image + rk3588-rock-5b-plus.dtb to boot partition
# 6. Boot and verify "Model: Radxa ROCK 5B+" in dmesg

# Lab 3: Bringup QEMU "new board"
# 1. Create custom QEMU machine in hw/arm/my_board.c
# 2. Wire: ARM Cortex-A53 + PL011 UART + GIC
# 3. Build kernel targeting it
# 4. Observe boot on your virtual "new board"
```
