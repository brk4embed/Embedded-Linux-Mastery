# 14 — AMD/x86 Debugging

> AMD debugging covers UEFI/EDK2 firmware, PCIe enumeration, SMU power management, and x86-specific debug tools — increasingly relevant as AMD gains embedded/automotive market share.

---

## Section Structure

```
14_AMD_Debugging/
├── 01_UEFI_EDK2_Debug.md             ← UEFI shell, DXE driver debug, OVMF
├── 02_PCIe_Analysis.md               ← lspci, setpci, PCIe link training debug
├── 03_SMU_Power_Debug.md             ← AMD SMU, power management, telemetry
├── 04_x86_UEFI_Variables.md          ← efivarfs, reading/writing UEFI vars
├── 05_Coreboot_AMD.md                ← Coreboot for AMD platforms (Chromebooks)
├── 06_JTAG_x86.md                    ← Intel/AMD JTAG with OpenOCD/XDP
└── 07_Linux_x86_Perf.md              ← perf, PMU counters, Intel VTune
```

---

## UEFI Debug Quick Reference

```bash
# Boot into UEFI shell (press Esc/F2 at POST)
# In UEFI shell:
Shell> map -r                    # list all block devices
Shell> fs0:                      # switch to first filesystem
Shell> ls                        # list directory
Shell> bcfg boot dump            # show boot order

# From Linux: read UEFI variables
ls /sys/firmware/efi/efivars/
cat /sys/firmware/efi/efivars/BootOrder-8be4df61...

# efibootmgr
efibootmgr -v                    # verbose boot entry list
efibootmgr --create --disk /dev/sda --part 1 \
  --label "Linux" --loader "\EFI\BOOT\bootx64.efi"
```

---

## PCIe Analysis

```bash
# List all PCIe devices with topology
lspci -tv

# Detailed device info including link speed
lspci -vvv -s <BDF>

# Read/write PCIe config space
setpci -s 00:1c.0 0x50.w          # read 2 bytes at offset 0x50
setpci -s 00:1c.0 0x50.w=0x0042   # write

# Check PCIe link speed
cat /sys/bus/pci/devices/0000:00:1c.0/current_link_speed
cat /sys/bus/pci/devices/0000:00:1c.0/current_link_width

# AER (Advanced Error Reporting)
dmesg | grep -i "pcie\|aer\|corrected\|uncorrected"
```

---

## Coreboot on AMD (Chromebook)

AMD-based Chromebooks (e.g., Guybrush, Mancomb) use Coreboot + Picasso/Mendocino SoCs:

```
src/soc/amd/
├── common/         ← Shared AMD SoC code
├── picasso/        ← Ryzen Embedded R1000/V1000
├── mendocino/      ← AMD Ryzen 5000/6000 Chromebook platforms
└── glinda/         ← Next-gen AMD Chromebook platform

Key AMD-specific Coreboot files:
- soc.c              ← SoC initialization sequence
- agesa/             ← AMD Generic Encapsulated SW Architecture (DDR, PSP)
- Makefile.inc       ← Build rules including AGESA binary blob
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What is UEFI? How is it different from legacy BIOS? |
| **Intermediate** | How do you debug a PCIe device that is not enumerated by the kernel? |
| **Advanced** | What is AGESA? How does AMD's boot flow differ from ARM SoCs? |
