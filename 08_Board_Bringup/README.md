# 08 — Board Bring-Up Methodology

> Board bring-up is the art of making a new SoC or custom PCB boot Linux for the first time. This section documents the systematic methodology used by professional BSP engineers.

---

## Section Structure

```
08_Board_Bringup/
├── 01_Bringup_Methodology.md         ← Systematic approach from power-on to rootfs
├── 02_Power_Rail_Verification.md     ← PMIC, regulators, power sequencing
├── 03_Clock_And_PLL_Setup.md         ← Clock tree, PLLs, DT clock bindings
├── 04_UART_First_Boot.md             ← Getting serial console on a new board
├── 05_DDR_Initialization.md          ← SPL DDR training, LPDDR4/5 bring-up
├── 06_U_Boot_Porting.md              ← Minimal U-Boot port for new board
├── 07_Kernel_DT_Bringup.md           ← Writing the device tree from scratch
├── 08_Driver_Enable_Strategy.md      ← Which drivers to enable first, in what order
├── 09_SC7180_SC7280_Case_Study.md    ← YOUR Coreboot bring-up experience
└── 10_Bringup_Checklist.md           ← Production bring-up checklist (50 items)
```

---

## The Board Bring-Up Phases

```
Phase 0: Pre-bring-up (Before touching hardware)
  ├── Read SoC TRM / datasheet thoroughly
  ├── Understand power sequencing from schematic
  ├── Identify debug interfaces (UART, JTAG, USB)
  └── Set up cross-compile toolchain and build environment

Phase 1: Power & Clocks
  ├── Verify all power rails with multimeter / oscilloscope
  ├── Check PMIC communication (I2C traces)
  ├── Verify oscillators and PLLs lock
  └── Check SoC reset sequence

Phase 2: Boot ROM / SPL
  ├── Get into BootROM mode (Maskrom / DFU / USB)
  ├── Flash minimal SPL via USB/JTAG
  ├── Verify DDR initialization (watch for training output)
  └── First UART output from SPL

Phase 3: U-Boot
  ├── Minimal U-Boot port (DDR + UART + storage)
  ├── Environment variables, bootcmd
  ├── USB/SD boot for iterative development
  └── Network boot (TFTP) for kernel development

Phase 4: Kernel First Boot
  ├── Minimal defconfig (no modules)
  ├── Minimal device tree (SoC + UART + storage)
  ├── Get to init=/bin/sh prompt
  └── Verify basic subsystems (clocks, regulators, interrupts)

Phase 5: Full BSP
  ├── Enable all peripherals one by one
  ├── Run compliance tests (USB, PCIe, storage benchmarks)
  ├── Power management (suspend/resume, cpufreq)
  └── Security hardening (secure boot, fuses)
```

---

## Your Case Study: Coreboot SC7180/SC7280

**Project:** ChromeOS Coreboot bring-up for Qualcomm Snapdragon 7c / 7c Gen 2  
**Scope:** 50-patch train from initial SoC support to production-ready Chromebook

### Key Bring-Up Challenges You Solved

| Challenge | Solution |
|-----------|----------|
| QTEE (TrustZone) initialization order | Correct SMC call sequence in Coreboot QSPI init |
| DDR training timing margins | Tuning CDR parameters in SC7280 DDR PHY config |
| UART log before RAM init | Used semi-hosting / jtag-uart before DDR up |
| eFuse verification (QFPROM) | Wrote the Linux QFPROM nvmem driver (upstream) |
| ChromeOS verified boot integration | Depthcharge integration with vboot2 |

### Coreboot Directory for SC7180/SC7280
```
src/soc/qualcomm/
├── common/         ← Shared Qualcomm SoC code
├── sc7180/         ← SC7180 (Snapdragon 7c) specific
│   ├── include/soc/
│   ├── mmu.c
│   ├── soc.c
│   └── Kconfig
└── sc7280/         ← SC7280 (Snapdragon 7c Gen 2) specific
    ├── include/soc/
    ├── qclib.c     ← QC library interface (TZ, DDR)
    ├── soc.c
    └── Makefile.inc
```

---

## UART First Boot Checklist

```bash
# 1. Identify UART TX/RX pins on schematic
# 2. Set baud rate (usually 115200 for modern SoCs)
# 3. Connect USB-to-UART adapter (no flow control)
# 4. Use minicom / tio / picocom

tio -b 115200 /dev/ttyUSB0

# 5. If no output: check power, check UART mux (may need DT/fuse config)
# 6. If garbage: check baud rate, check UART clock source
# 7. For Qualcomm: UART is usually enabled in Coreboot/XBL before kernel
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What is the first thing you check when a new board has no serial output? |
| **Intermediate** | How do you bring up DDR on a new SoC? What is DDR training? |
| **Advanced** | Describe your bring-up methodology from power-on to first kernel prompt. |
| **Expert** | How would you debug a board that hangs silently after the BootROM stage? |
