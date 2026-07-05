# 24 — Industry Project Portfolio

> Five fully documented portfolio projects that demonstrate your embedded Linux expertise to employers and clients.

---

## Section Structure

```
24_Industry_Project_Portfolio/
├── 01_DW_UFS_QEMU_Model.md           ← Complete doc of your DW-UFS 4.0 project
├── 02_AI_PreSilicon_Platform.md      ← LangGraph pre-silicon validation platform
├── 03_Coreboot_SC7180_SC7280.md      ← 50-patch Coreboot train documentation
├── 04_QFPROM_Linux_Driver.md         ← Upstream Linux NVMEM/QFPROM driver
├── 05_Radxa_5B_Plus_AI_BSP.md        ← Full BSP + AI inference on RK3588
└── 06_Portfolio_Presentation.md      ← How to present these to clients/employers
```

---

## Your 5 Portfolio Projects

### Project 1: DW-UFS 4.0 QEMU Device Model

| Attribute | Details |
|-----------|---------|
| **Client** | ARM ADC (Eximietas Design India) |
| **Tech Stack** | C, QEMU device model framework, UFS 4.0 spec, SCSI/UFS UPIU protocol |
| **Impact** | Enabled pre-silicon software validation without physical hardware |
| **Upstream** | QEMU project (status: in review / merged) |
| **Lines of Code** | ~3,000 lines C |
| **Key Skills** | QEMU TypeInfo, MemoryRegion, QOM, UFS UPIU, SCSI command set |

**One-liner pitch:** "Built the industry's first open-source UFS 4.0 QEMU device model, enabling software teams to validate storage drivers 6 months before silicon tapeout."

---

### Project 2: AI Pre-Silicon Validation Platform

| Attribute | Details |
|-----------|---------|
| **Tech Stack** | Python, LangGraph, Claude API, QEMU, pytest |
| **Architecture** | Multi-agent: Test Generator → QEMU Runner → Log Analyzer → Reporter |
| **Impact** | Reduced manual test analysis time by 70% |
| **Key Skills** | LangGraph, AI agents, kernel log parsing, QEMU automation |

---

### Project 3: Coreboot SC7180/SC7280 (50-patch series)

| Attribute | Details |
|-----------|---------|
| **Platform** | Qualcomm Snapdragon 7c / 7c Gen 2 (ChromeOS Chromebooks) |
| **Tech Stack** | C, Coreboot, Depthcharge, Gerrit, ChromeOS build system |
| **Scope** | SoC init, DDR, UART, QSPI, GPIO, PMIC, TrustZone, vboot2 integration |
| **Upstream** | review.coreboot.org — all patches merged |
| **Impact** | Enabled ChromeOS on two Qualcomm Snapdragon platforms |

---

### Project 4: QFPROM eFuse Linux Driver

| Attribute | Details |
|-----------|---------|
| **File** | `drivers/nvmem/qfprom.c` (upstream Linux kernel) |
| **Subsystem** | NVMEM |
| **Platforms** | SC7180, SC7280 |
| **Function** | Provides access to eFuse cells (serial number, calibration, secure boot fuses) |
| **Status** | Merged to mainline Linux kernel |

---

### Project 5: Radxa Rock 5B+ AI BSP

| Attribute | Details |
|-----------|---------|
| **Hardware** | Radxa Rock 5B+ (Rockchip RK3588, 16GB LPDDR5, 6 TOPS NPU) |
| **Tech Stack** | Yocto, Linux 6.x, rknn-toolkit2, custom AI layer |
| **Deliverables** | Custom BSP layer, NPU driver integration, benchmark suite |
| **Goal** | Demonstrate full-stack embedded AI on production hardware |

---

## Portfolio Presentation Template

```markdown
# [Project Name] — Case Study

## The Problem
[1-2 sentences: what challenge did this solve]

## My Role
[Your specific contribution, not the team's]

## Technical Approach
[Architecture diagram + key technical decisions]

## Results
[Quantified impact: time saved, performance gained, patches merged]

## What I Learned
[Key insights — shows growth mindset]

## Links
- GitHub: [repo]
- Blog post: [URL]
- Presentation: [slides]
```
