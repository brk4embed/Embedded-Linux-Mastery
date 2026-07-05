# 22 — Certifications

> Strategic certifications that add credibility to your profile as an embedded Linux engineer and AI architect.

---

## Section Structure

```
22_Certifications/
├── 01_LFCE_LFCS_Guide.md             ← Linux Foundation Certified Engineer/Sysadmin
├── 02_CKA_Kubernetes.md              ← Certified Kubernetes Admin (for MLOps/Edge AI)
├── 03_Yocto_Certification.md         ← Yocto Project Developer Certification
├── 04_ARM_Accreditation.md           ← ARM Accredited Engineer program
├── 05_AWS_For_Embedded.md            ← AWS IoT + Greengrass + Graviton certs
└── 06_Study_Plans.md                 ← 90-day study plans for each cert
```

---

## Certification Priority Matrix

| Certification | Provider | Cost | Relevance to Your Profile | Priority |
|--------------|----------|------|--------------------------|----------|
| **Linux Foundation Certified Engineer (LFCE)** | LF | $395 | Direct — kernel/sysadmin | ⭐⭐⭐⭐⭐ |
| **Embedded Linux Engineer (ELE)** | LF | $295 | Direct — embedded Linux | ⭐⭐⭐⭐⭐ |
| **Yocto Project Developer** | YP | Free | Direct — BSP/build systems | ⭐⭐⭐⭐ |
| **AWS IoT / Greengrass** | AWS | $300 | Edge AI deployment | ⭐⭐⭐ |
| **CKA (Kubernetes)** | CNCF | $395 | MLOps/AI infrastructure | ⭐⭐⭐ |
| **ARM Accredited Engineer** | ARM | Free | ARM ecosystem credibility | ⭐⭐⭐⭐ |

---

## Linux Foundation Embedded Linux (ELE) — Study Plan

### Exam Domains
1. **Boot Process** (20%) — U-Boot, Coreboot, boot parameters
2. **Kernel Internals** (25%) — Compilation, configuration, modules
3. **Device Drivers** (20%) — Platform, character, block drivers
4. **Build Systems** (15%) — Yocto, Buildroot, cross-compilation
5. **Debugging** (20%) — GDB, KGDB, dynamic debug, ftrace

### 90-Day Study Plan

| Week | Focus | Resources |
|------|-------|-----------|
| 1-2 | Boot process: U-Boot, Coreboot, DT | 05_Bootloaders/, 08_Board_Bringup/ |
| 3-4 | Kernel internals: config, build, modules | 06_Linux_Kernel/ |
| 5-6 | Device drivers: character, platform, I2C | 07_Device_Drivers/ |
| 7-8 | Build systems: Yocto, Buildroot | 28_Yocto_Build_Mastery/ |
| 9-10 | Debugging: GDB, ftrace, KASAN | 09_Debugging/ |
| 11 | Practice exams | LF practice portal |
| 12 | Review weak areas + final prep | Focus on gaps |

---

## ARM Accredited Engineer Program

```
ARM Developer Program:
1. Register at developer.arm.com
2. Complete online learning paths:
   - ARM Architecture Fundamentals
   - Cortex-A Programming
   - TrustZone Security
3. Pass proctored online exam (free)
4. Receive digital badge for LinkedIn

Relevant to your profile:
- ARM big.LITTLE (RK3588 uses Cortex-A76/A55)
- TrustZone/QTEE (your ChromeOS experience)
- AMBA/AXI (used in your QEMU device model)
```

---

## Yocto Project Developer Certification

```bash
# The exam tests practical Yocto skills:
# - Writing recipes from scratch
# - Creating custom layers
# - BSP layer for a specific board
# - Troubleshooting build failures

# Practice: Build your Radxa 5B+ BSP layer
# See: 28_Yocto_Build_Mastery/README.md
```

---

## Resources

- [Linux Foundation Training](https://training.linuxfoundation.org/)
- [Yocto Project](https://www.yoctoproject.org/development/technical-faq/)
- [ARM Developer](https://developer.arm.com/training)
- [AWS Training](https://aws.amazon.com/training/learn-about/iot/)
