# 01 — SoC Internal Architecture (RK3588 Deep Dive)

> Understanding the SoC block diagram is the foundation for all BSP and driver work.  
> Reference board: Radxa Rock 5B+ (RK3588 SoC)

---

## Level 1: The 5-Year-Old Explanation

A SoC (System-on-Chip) is like a city on a tiny piece of silicon. Inside the city there are:
- Brains (CPU cores) — they think and do calculations
- A highway system (bus fabric) — lets all the parts talk to each other
- A memory warehouse (DDR controller) — stores all the data
- Workers with special skills (GPU, NPU, DSP) — do specific jobs faster
- Doors to the outside world (peripherals: UART, I2C, USB, PCIe) — connect to other hardware

---

## Level 2: Overview Diagram

```mermaid
flowchart TD
    classDef cpu fill:#1e3a5f,color:#fff,stroke:#4a90d9
    classDef gpu fill:#1a4a1a,color:#fff,stroke:#4a9f4a
    classDef npu fill:#2d1a4a,color:#fff,stroke:#8b5cf6
    classDef mem fill:#4a3a00,color:#fff,stroke:#c9a227
    classDef bus fill:#2d1a4a,color:#fff,stroke:#8b5cf6
    classDef peri fill:#1a3a3a,color:#fff,stroke:#4ab0b0
    classDef sec fill:#4a1a1a,color:#fff,stroke:#c94040
    classDef video fill:#3a1a3a,color:#fff,stroke:#c054c0

    subgraph CPU_CLUSTER["CPU Subsystem"]
        BIG["Big Cluster\n4× Cortex-A76\n@ 2.4GHz\nL2: 512KB each\nL3: 4MB shared"]:::cpu
        LIT["Little Cluster\n4× Cortex-A55\n@ 1.8GHz\nL2: 128KB each\nL3: 2MB shared"]:::cpu
    end

    subgraph COMPUTE["Compute Accelerators"]
        GPU["Mali-G610 MP4 GPU\n4 shader cores\nOpenGL ES 3.2\nVulkan 1.1\nOpenCL 2.0"]:::gpu
        NPU["NPU 3.0\n3× AI cores\n6 TOPS INT8\n3 TOPS INT4\nRKNN API"]:::npu
        DSP["Hifi4 DSP\nAudio processing\n@ 800MHz"]:::peri
    end

    subgraph MEMORY["Memory Subsystem"]
        DDR_CTL["DDR Controller\nLPDDR4X\n64-bit bus\n@ 2112MT/s\nMax: 32GB"]:::mem
        IRAM["Internal SRAM\n192KB\nFor BootROM/SPL"]:::mem
        L3BIG["L3 Cache\n4MB (A76)"]:::mem
        L3LIT["L3 Cache\n2MB (A55)"]:::mem
    end

    subgraph CONNECTIVITY["Connectivity"]
        USB3["USB 3.1 Gen1\n×2 (Host+OTG)"]:::peri
        USB2["USB 2.0 ×2"]:::peri
        PCIE3["PCIe 3.0 ×4\nNVMe SSD"]:::peri
        PCIE2["PCIe 2.1 ×1\nEthernet"]:::peri
        SATA["SATA 3.0 ×1"]:::peri
        GMAC["GMAC ×2\n2.5G Ethernet"]:::peri
    end

    subgraph STORAGE["Storage"]
        EMMC["eMMC 5.1\n400MB/s"]:::mem
        SDMMC["SD/MMC\nSD 3.0"]:::mem
        SPI_NOR["SPI NOR flash\nBootROM fallback"]:::mem
        UFS["UFS 3.1\n(select configs)"]:::mem
    end

    subgraph DISPLAY["Display & Video"]
        HDMI2["HDMI 2.1\n8K@60fps"]:::video
        DP["DisplayPort 1.4\nvia USB-C"]:::video
        MIPI_DSI["MIPI-DSI ×2\n4-lane"]:::video
        VENC["Video Encoder\n8K@30fps H.265"]:::video
        VDEC["Video Decoder\n8K@60fps H.265"]:::video
        ISP["ISP 3.0\n48MP camera"]:::video
    end

    subgraph SECURITY["Security"]
        BOOTROM_SEC["BootROM\n32KB ROM"]:::sec
        CRYPTO["Crypto Engine\nAES/SHA/RSA"]:::sec
        EFUSE["eFuse\nSecure boot key\n(QFPROM equivalent)"]:::sec
        TZPC["TrustZone\nController"]:::sec
        OTP["OTP memory\n4KB"]:::sec
    end

    subgraph BUS_FABRIC["Bus Fabric (CCI-500)"]
        CCI["CCI-500\nCache Coherent\nInterconnect"]:::bus
        AXI_P["AXI Peripheral\nInterconnect"]:::bus
        NOC["Network-on-Chip\nRK3588 NoC"]:::bus
    end

    CPU_CLUSTER --> CCI
    GPU --> CCI
    NPU --> NOC
    DSP --> AXI_P
    CCI --> DDR_CTL
    CCI --> L3BIG
    CCI --> L3LIT
    NOC --> AXI_P
    AXI_P --> CONNECTIVITY
    AXI_P --> STORAGE
    AXI_P --> DISPLAY
    AXI_P --> SECURITY
    SECURITY --> IRAM
```

---

## Level 3: The CPU Subsystem

### Big.LITTLE Architecture

```
RK3588 CPU Topology:
─────────────────────────────────────────────────────
  Big Cluster (performance)     Little Cluster (efficiency)
  ─────────────────────────     ─────────────────────────
  4× Cortex-A76 cores           4× Cortex-A55 cores
  2.4GHz max                    1.8GHz max
  512KB L2 per core             128KB L2 per core
  4MB L3 shared                 2MB L3 shared
  ARMv8.2-A ISA                 ARMv8.2-A ISA
  Out-of-order execute          In-order execute
  10-wide decode                5-wide decode

BSP Impact:
  - /sys/devices/system/cpu/cpu0-3  = A55 (little)
  - /sys/devices/system/cpu/cpu4-7  = A76 (big)
  - CPUFreq driver: rockchip-cpufreq.c
  - Thermal: rockchip-thermal.c
```

**Kernel command line to verify:**
```bash
# On Radxa 5B+ or QEMU with RK3588 DTB:
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
# 1800000  (A55: 1.8GHz)

cat /sys/devices/system/cpu/cpu4/cpufreq/cpuinfo_max_freq
# 2400000  (A76: 2.4GHz)

# Current topology
cat /sys/devices/system/cpu/cpu4/topology/core_id
```

### Cache Coherency (Critical for Driver Writing)

```
          CPU Core (A76)
             │
           L1 I$     L1 D$    (private, per-core)
             │         │
             └────┬────┘
                  │
                L2 $     (512KB, private per A76 core)
                  │
                  └──── CCI-500 ─────┐
                                     │
                              L3 Cache (4MB, shared A76)
                                     │
                              DDR Controller
                                     │
                               LPDDR4X DRAM
```

**Why this matters for driver writing:**
- DMA operations: device writes to DRAM, CPU needs to see correct data
- If CPU's cache line has stale data → coherency bug → wrong values read
- Solution: `dma_alloc_coherent()` or explicit `dma_sync_*` operations

```c
/* Correct DMA buffer allocation (coherent, no explicit cache ops) */
dma_addr_t dma_addr;
void *cpu_addr = dma_alloc_coherent(dev, size, &dma_addr, GFP_KERNEL);
// Device writes to dma_addr, CPU reads from cpu_addr → always consistent

/* Non-coherent (streaming DMA) — need explicit sync */
dma_addr_t dma_addr = dma_map_single(dev, cpu_ptr, size, DMA_FROM_DEVICE);
// ... device DMA transfer ...
dma_unmap_single(dev, dma_addr, size, DMA_FROM_DEVICE);
// NOW safe to read from cpu_ptr (cache has been invalidated)
```

---

## Level 4: Memory Map

```
RK3588 64-bit Physical Address Space:
──────────────────────────────────────────────────
0x0000_0000_0000_0000  BootROM (32KB)
0x0000_0000_0002_0000  Reserved
0x0000_0000_0008_0000  PMU (Power Management Unit)
0x0000_0000_0009_0000  GRF (General Register Files — clock/reset control)
0x0000_0000_00C0_0000  GIC-600 (Interrupt Controller)
0x0000_0000_00FD_0000  CRU (Clock & Reset Unit)
0x0000_0000_FD58_C000  IRAM (192KB)
0x0000_0000_FE20_0000  SARADC
0x0000_0000_FE2A_0000  I2C0–I2C8
0x0000_0000_FE60_0000  eMMC (SDHCI)
0x0000_0000_FE8C_0000  USB OTG
0x0000_0000_FE90_0000  USB HOST
0x0000_0000_FE18_0000  SPI0–SPI5
0x0000_0000_0003_0000  DDR Controller
0x0000_0000_0008_0000  DRAM starts (mapped by DDR controller)
```

**Reading the memory map in device tree:**
```bash
# On Radxa 5B+
cat /proc/iomem
# or
cat /sys/kernel/debug/iomap  # if CONFIG_IO_STRICT_DEVMEM=n

# View UART registers:
cat /proc/iomem | grep uart
# fe650000-fe653fff : fe650000.serial
# fe660000-fe663fff : fe660000.serial
```

---

## Level 5: Bus Fabric — How Everything Talks

```mermaid
flowchart LR
    classDef master fill:#1e3a5f,color:#fff,stroke:#4a90d9
    classDef slave fill:#1a4a1a,color:#fff,stroke:#4a9f4a
    classDef bus fill:#2d1a4a,color:#fff,stroke:#8b5cf6

    subgraph MASTERS["AXI Masters"]
        CPU_M["CPU Cores\n(via CCI-500)"]:::master
        GPU_M["Mali-G610"]:::master
        NPU_M["NPU 3.0"]:::master
        ISP_M["ISP 3.0"]:::master
        VPU_M["VPU (Video)"]:::master
        DMA_M["DMAC controllers"]:::master
    end

    subgraph BUS["CCI-500 + NoC"]
        CCI_BUS["CCI-500\nCache Coherent\nInterconnect"]:::bus
        NOC_BUS["Network on Chip\n(non-coherent path)"]:::bus
    end

    subgraph SLAVES["AXI Slaves"]
        DDR_S["DDR Controller\n(DRAM)"]:::slave
        PERI_S["Peripherals\nI2C/SPI/UART"]:::slave
        USB_S["USB Controllers"]:::slave
        PCIE_S["PCIe Controllers"]:::slave
        EMMC_S["eMMC Controller"]:::slave
        EFUSE_S["eFuse/Secure ROM"]:::slave
    end

    MASTERS --> CCI_BUS
    CCI_BUS --> NOC_BUS
    CCI_BUS --> DDR_S
    NOC_BUS --> PERI_S
    NOC_BUS --> USB_S
    NOC_BUS --> PCIE_S
    NOC_BUS --> EMMC_S
    NOC_BUS --> EFUSE_S
```

---

## Interview Questions

**Beginner:**
- What is a SoC? How does it differ from a CPU?
- What is the difference between the "Big" and "Little" CPU clusters in big.LITTLE?
- What is IRAM and why does the BootROM use it?

**Intermediate:**
- What is cache coherency and why does it matter for DMA drivers?
- What is the CCI-500 and what problem does it solve?
- Explain the difference between `dma_alloc_coherent` and `dma_map_single`.

**Advanced:**
- How does the NPU access DRAM without going through the CPU's cache domain?
- Explain AXI protocol: what are burst transactions and why are they important for performance?
- How would you trace a bus hang (AXI timeout) on an RK3588?

**Expert:**
- Design a driver that implements zero-copy DMA from a camera ISP to an NPU. What memory regions, IOMMU configuration, and synchronization are needed?
- How does the SMMU-600 provide DMA isolation for different devices? What attack does it prevent?
- Explain how the GIC-600 handles an IRQ from the UART RX FIFO to the kernel ISR.

---

*Related: [../03_Memory_Map.md](03_Memory_Map.md) | [../08_Interrupt_Architecture.md](08_Interrupt_Architecture.md)*  
*Labs: [../../05_Practical_Labs/Lab_04_Read_PMIC_Over_I2C.md](../../05_Practical_Labs/Lab_04_Read_PMIC_Over_I2C.md)*
