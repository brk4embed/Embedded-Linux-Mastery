# 01 — Master Boot Flow Diagram

> **The most important diagram in this repository.**  
> This is the complete power-on to AI-inference-running boot flow for an ARM64 embedded Linux system (RK3588 / Radxa Rock 5B+ primary reference).

---

## The Complete Boot Flow

```mermaid
sequenceDiagram
    participant PWR as ⚡ Power/PMIC
    participant HW as 🔩 Hardware
    participant BROM as 🔒 BootROM (EL3)
    participant SPL as 🔄 SPL/TPL (EL3)
    participant ATF as 🛡️ ATF BL31 (EL3)
    participant OPTEE as 🔐 OP-TEE (EL1-S)
    participant UBOOT as 🥾 U-Boot (EL1)
    participant KERN as 🐧 Kernel (EL1)
    participant DRV as ⚙️ Drivers
    participant INIT as 🚀 systemd
    participant APP as 📱 Application

    Note over PWR: t=0ms  Power applied
    PWR->>HW: VCC_3V3, VDD_CPU rails ramp up
    HW->>HW: PGOOD asserted (~2ms)
    HW->>BROM: CPU reset deasserts, PC=0xFFFF0000
    
    Note over BROM: t=2ms  BootROM starts
    BROM->>BROM: Minimal clock init (no DDR)
    BROM->>BROM: Read boot strap pins / eFuse
    BROM->>BROM: Check boot media priority
    Note over BROM: SPI NOR → eMMC → SD → USB
    BROM->>HW: Read 4KB header from boot media
    BROM->>BROM: Verify header magic number
    BROM->>HW: Load TPL to IRAM (192KB max)
    BROM->>SPL: Jump to TPL entry point
    
    Note over SPL: t=15ms  SPL/TPL starts
    SPL->>HW: Configure PLLs to full speed
    SPL->>HW: Initialize UART (first console output!)
    Note over SPL: "DDR init..." appears on UART
    SPL->>HW: DDR4X training sequence
    Note over HW: Write leveling, DQ calib, Vref training
    HW-->>SPL: DRAM alive (8GB available)
    SPL->>HW: Read ATF + OP-TEE + U-Boot from storage
    SPL->>HW: Load to DRAM at target addresses
    SPL->>ATF: Jump to ATF BL31 (EL3)
    
    Note over ATF: t=300ms  ARM Trusted Firmware
    ATF->>ATF: Set up EL3 exception vectors
    ATF->>ATF: Configure PSCI (CPU hotplug/suspend)
    ATF->>HW: Configure GIC (secure/non-secure IRQ split)
    ATF->>HW: Set up TrustZone memory firewall
    Note over ATF: Secure memory = OP-TEE RAM
    ATF->>OPTEE: Jump to OP-TEE BL32
    
    Note over OPTEE: t=350ms  OP-TEE Secure OS
    OPTEE->>OPTEE: Init secure heap, thread pool
    OPTEE->>OPTEE: Register secure interrupt handlers
    OPTEE->>ATF: Return to ATF (OPTEE_ENTRY_RETURN)
    ATF->>UBOOT: Jump to U-Boot (EL1, non-secure)
    
    Note over UBOOT: t=400ms  U-Boot bootloader
    UBOOT->>HW: Full peripheral initialization
    Note over UBOOT: USB, PCIe, display, network, storage
    UBOOT->>UBOOT: Read environment variables
    UBOOT->>UBOOT: Execute bootcmd
    UBOOT->>HW: Read kernel Image from eMMC/SD
    UBOOT->>HW: Read DTB from eMMC/SD
    UBOOT->>UBOOT: Set bootargs (cmdline)
    UBOOT->>KERN: booti: x0=DTB_addr, jump to kernel
    
    Note over KERN: t=1200ms  Linux kernel entry
    KERN->>KERN: head.S: EL2/EL1 mode check
    KERN->>KERN: Decompress if compressed Image
    KERN->>KERN: Enable MMU, set up page tables
    KERN->>KERN: start_kernel(): setup_arch()
    KERN->>HW: Parse Device Tree (DTB → device_node tree)
    KERN->>KERN: Subsystem init: irq, mm, sched, rcu, timers
    KERN->>DRV: driver_init(): scan platform/I2C/SPI buses
    
    Note over DRV: t=2000ms  Driver probe begins
    DRV->>DRV: Match DT compatible strings to drivers
    DRV->>HW: UART driver probe → /dev/ttyS2
    DRV->>HW: I2C driver probe → 4x I2C buses
    DRV->>HW: PCIe driver probe → NVMe / Ethernet
    DRV->>HW: USB driver probe → xHCI controller
    DRV->>HW: GPU driver probe → Mali-G610
    DRV->>HW: NPU driver probe → RKNN NPU 3.0
    DRV->>HW: eMMC driver probe → /dev/mmcblk0
    
    KERN->>INIT: kernel_init: exec /sbin/init (PID 1)
    
    Note over INIT: t=3000ms  systemd starts
    INIT->>INIT: Parse service units
    INIT->>HW: udev: create /dev nodes
    INIT->>INIT: Start network manager
    INIT->>INIT: Start SSH daemon
    INIT->>APP: Start application services
    
    Note over APP: t=5000ms  Application running
    APP->>HW: Open /dev/rknpu
    APP->>HW: Load RKNN model to NPU
    APP->>APP: Run AI inference
    APP-->>APP: 🎯 SYSTEM FULLY OPERATIONAL
```

---

## Time Budget (Typical RK3588 System)

| Stage | Start | End | Duration | Optimization Potential |
|-------|-------|-----|----------|----------------------|
| Power ramp + POR | 0ms | 2ms | 2ms | None (hardware) |
| BootROM | 2ms | 15ms | 13ms | None (ROM) |
| SPL/TPL (DDR training) | 15ms | 300ms | 285ms | Minimal — DDR training is physics |
| ATF + OP-TEE | 300ms | 400ms | 100ms | Compile-time config |
| U-Boot | 400ms | 1200ms | 800ms | **High** — disable unused drivers |
| Kernel decompression | 1200ms | 1400ms | 200ms | Use uncompressed Image |
| Kernel init + subsystems | 1400ms | 2000ms | 600ms | **High** — modularize drivers |
| Driver probe | 2000ms | 3000ms | 1000ms | **High** — async probe |
| systemd + services | 3000ms | 5000ms | 2000ms | **High** — disable unused services |
| **Total to shell prompt** | | **~5s** | **5000ms** | Target: **< 2s** |

---

## The ARM Exception Level Diagram

```mermaid
flowchart TD
    classDef el3 fill:#4a1a1a,color:#fff,stroke:#c94040
    classDef el2 fill:#1a2a4a,color:#fff,stroke:#4a7ac9
    classDef el1s fill:#2a1a4a,color:#fff,stroke:#8b4ac9
    classDef el1ns fill:#1a3a1a,color:#fff,stroke:#4a9f4a
    classDef el0 fill:#1a1a1a,color:#fff,stroke:#808080

    EL3["EL3 — Secure Monitor\nATF BL31 (resident)\nPSCI, TrustZone config\nSMC handler"]:::el3
    
    EL2["EL2 — Hypervisor\nKVM / pKVM (optional)\nVirtual machine management\n(not used on bare metal)"]:::el2
    
    EL1S["EL1-S — Secure OS\nOP-TEE\nTrusted applications\nSecure storage, crypto"]:::el1s
    
    EL1NS["EL1-NS — Linux Kernel\nU-Boot (before Linux)\nProcess management\nDevice drivers\nMemory management"]:::el1ns
    
    EL0["EL0 — User Space\nYour applications\nShell, SSH, AI runtime\nLibraries (libc, librknn)"]:::el0

    EL3 -->|"SMC return to NS"| EL1NS
    EL3 -->|"ERET to secure"| EL1S
    EL3 -->|"ERET to EL2"| EL2
    EL2 -->|"HVC return"| EL1NS
    EL1NS -->|"SVC / syscall"| EL0
    EL1S -->|"Trusted app"| EL0
    
    EL1NS <-->|"SMC (secure call)"| EL3
    EL1NS <-->|"HVC (hypervisor)"| EL2
```

---

## Memory Map at Each Boot Stage

```mermaid
flowchart LR
    classDef hw fill:#1e3a5f,color:#fff,stroke:#4a90d9
    classDef sw fill:#1a4a1a,color:#fff,stroke:#4a9f4a
    classDef cfg fill:#4a3a00,color:#fff,stroke:#c9a227
    classDef sec fill:#4a1a1a,color:#fff,stroke:#c94040

    subgraph STAGE1["Stage: BootROM (only IRAM)"]
        B1["0x00000000: BootROM code (64KB)"]:::sec
        B2["0xFF8C0000: IRAM (192KB)\n← SPL/TPL loaded here"]:::sw
        B3["BootROM stack in IRAM"]:::sw
    end

    subgraph STAGE2["Stage: After DDR Training (DRAM available)"]
        D1["0x00040000 ATF BL31 (EL3 stays here forever)"]:::sec
        D2["0x00100000 OP-TEE (secure world)"]:::sec
        D3["0x00200000 U-Boot"]:::sw
        D4["0xFF8C0000 IRAM (SPL still running)"]:::sw
        D5["0x08000000 DDR accessible (8GB)"]:::hw
    end

    subgraph STAGE3["Stage: Kernel Running"]
        K1["0x00040000 ATF (still resident in EL3)"]:::sec
        K2["0x00100000 OP-TEE"]:::sec
        K3["0x00480000 Kernel (link addr)"]:::sw
        K4["DTB below kernel"]:::cfg
        K5["initramfs above kernel"]:::sw
        K6["Kernel virtual addr space: 0xFFFF000000000000+"]:::sw
    end
```

---

## Flash Partition Map (eMMC / SD Card)

```mermaid
flowchart TD
    classDef part fill:#1e3a5f,color:#fff,stroke:#4a90d9
    classDef sw fill:#1a4a1a,color:#fff,stroke:#4a9f4a
    classDef cfg fill:#4a3a00,color:#fff,stroke:#c9a227

    subgraph EMMC["eMMC Layout (GPT)"]
        direction LR
        P0["Sector 0\nMBR / GPT header"]:::cfg
        P1["idbloader partition\n(TPL+SPL, ~512KB)"]:::sw
        P2["uboot partition\n(U-Boot, ~2MB)"]:::sw
        P3["trust partition\n(ATF+OP-TEE, ~4MB)"]:::sw
        P4["boot partition\n(kernel+DTB, ~128MB)"]:::sw
        P5["rootfs partition\n(ext4, remaining)"]:::sw
        P6["userdata partition\n(optional, user data)"]:::cfg
    end
    
    P0 --> P1 --> P2 --> P3 --> P4 --> P5 --> P6
```

**Real commands to inspect on Radxa 5B+:**
```bash
# View partition table
lsblk
fdisk -l /dev/mmcblk0

# Read idbloader header
dd if=/dev/mmcblk0 bs=512 skip=64 count=1 | hexdump -C | head

# Check boot partition contents
mount /dev/mmcblk0p3 /mnt/boot
ls -la /mnt/boot/
# extlinux/extlinux.conf  ← U-Boot boot script
# Image                   ← Linux kernel
# rk3588-rock-5b.dtb      ← Device tree
```

---

## Debug Quick Reference

### Stage-by-Stage Failure Diagnosis

```
Q: Board is dead. No UART output. USB device not detected.
A: Power stage. Check:
   - Is 3V3 rail present? (probe with multimeter)
   - Is VDD_CPU_BIG present? (~0.95V under light load)
   - Is PGOOD signal asserted to SoC?
   - Is USB-C cable capable of power delivery?

Q: Board powers on. UART shows garbage characters.
A: UART baud rate mismatch OR SPL clock init issue.
   - Try different baud rates: 1500000, 115200, 921600
   - Check PLL configuration in SPL source

Q: UART shows "DDR init..." then hangs.
A: DDR training failure.
   - Wrong DDR frequency/timing in SPL config
   - Bad DDR solder joint or routing issue
   - Incompatible DRAM part (check SPL support matrix)

Q: "Starting kernel..." then nothing.
A: U-Boot → Kernel handoff issue.
   - Wrong kernel load address in U-Boot bootcmd
   - DTB not loaded or at wrong address
   - Kernel not compiled for this platform (defconfig)

Q: Kernel panics: "Kernel panic - not syncing: VFS: Unable to mount root"
A: rootfs issue.
   - Check: root=/dev/mmcblk0p5 in bootargs (or whatever partition)
   - Check: rootfstype=ext4 in bootargs
   - Check: kernel has mmcblk driver built-in (not as module)

Q: Driver not probing, device not appearing in /dev
A: DT compatible string mismatch.
   - cat /proc/device-tree/[node-name]/compatible
   - grep -r "compatible" drivers/[subsystem]/ | grep "your-string"
```

---

## AI-Assisted Boot Debug Workflow

```
When you have a boot failure, give this prompt to Claude/ChatGPT:

"I'm debugging a boot failure on an RK3588-based board.
The UART output stops after: [paste last 20 lines of output]
My boot configuration:
- TPL/SPL: [version/source]
- U-Boot: [version]  
- Kernel: [version + defconfig]
- DTB: [filename]
- Storage: [eMMC/SD]

What are the most likely causes? What should I check first?"
```

---

## Interview Questions

**Beginner (B):**
- B1: What happens in the very first milliseconds after power is applied to an embedded board?
- B2: Why can't BootROM access DRAM? What does it use instead?
- B3: What is the purpose of DDR training?

**Intermediate (I):**
- I1: Explain the ARM exception levels EL0–EL3 and which software runs at each level.
- I2: What is the role of ATF BL31 as a "resident monitor" — why does it stay resident?
- I3: How does U-Boot pass information to the Linux kernel at handoff?

**Advanced (A):**
- A1: A board outputs "DDR init..." then hangs. Walk through your debugging approach.
- A2: How does OP-TEE communicate with the Linux kernel? What is the TEE driver interface?
- A3: Explain how DT compatible string matching works — what happens between `booti` and the first driver `probe()` call?

**Expert (E):**
- E1: You're bringing up a new SoC. You have schematics and a datasheet but no BSP. The BootROM loads from SPI NOR. Walk through creating the minimal SPL to get UART output.
- E2: How does the kernel's PSCI client (`drivers/firmware/psci/psci.c`) call ATF BL31 for CPU hotplug? Trace the SMC call path.
- E3: Design a secure boot chain for RK3588 that uses hardware-backed key storage. What eFuse bits are involved? How does each stage verify the next?

---

*Previous: [00_The_Big_Picture/01_From_Idea_To_Running_System.md](../00_The_Big_Picture/01_From_Idea_To_Running_System.md)*  
*Next: [02_Boot_Stage_Debug_Guide.md](02_Boot_Stage_Debug_Guide.md)*  
*Labs: [../05_Practical_Labs/Lab_07_QEMU_Full_Boot_Stack.md](../05_Practical_Labs/Lab_07_QEMU_Full_Boot_Stack.md)*
