# 17 — QEMU Virtualization

> QEMU is a complete machine emulator. For embedded Linux engineers, it's the ideal first test environment — faster iteration, no hardware needed, full debuggability.

---

## Section Structure

```
17_QEMU_Virtualization/
├── 01_QEMU_Architecture.md           ← How QEMU works (TCG, KVM, device emulation)
├── 02_ARM64_Virtual_Machine.md       ← Boot Linux on qemu-system-aarch64
├── 03_Device_Model_Writing.md        ← Writing a QEMU device model in C
├── 04_DW_UFS_Case_Study.md           ← YOUR DW-UFS 4.0 QEMU device model
├── 05_QEMU_GDB_Integration.md        ← Kernel debugging with GDB + QEMU stub
├── 06_QEMU_Networking.md             ← TAP, SLiRP, virtio-net for QEMU VMs
├── 07_QEMU_Storage.md                ← virtio-blk, NVMe, UFS emulation
├── 08_QEMU_Trace_And_Debug.md        ← QEMU tracing framework, -d flags
└── 09_Custom_Machine_Type.md         ← Creating a custom QEMU machine
```

---

## QEMU Device Model Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    QEMU Process                          │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │   Guest CPU  │    │  Guest RAM   │                   │
│  │  (TCG/KVM)   │    │  (mmap'd)    │                   │
│  └──────┬───────┘    └──────────────┘                   │
│         │ MMIO read/write                                │
│  ┌──────▼────────────────────────────────────────────┐  │
│  │              QEMU Device Model                     │  │
│  │                                                    │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │  │
│  │  │TypeInfo  │  │MemRegion │  │   IRQ / MSI-X    │ │  │
│  │  │(class)   │  │(MMIO map)│  │   (qemu_irq)     │ │  │
│  │  └──────────┘  └──────────┘  └──────────────────┘ │  │
│  │                                                    │  │
│  │  realize() → init state, map MMIO, connect IRQ     │  │
│  │  read()    → handle guest MMIO reads               │  │
│  │  write()   → handle guest MMIO writes              │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Your DW-UFS 4.0 Device Model (Case Study)

**Project context:** ARM ADC engagement, QEMU UFS 4.0 model for pre-silicon validation

### Key Components

```c
/* UFS device type registration */
static const TypeInfo dw_ufs_info = {
    .name          = TYPE_DW_UFS,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(DwUFSState),
    .instance_init = dw_ufs_init,
    .class_init    = dw_ufs_class_init,
};

/* MMIO operations */
static const MemoryRegionOps dw_ufs_ops = {
    .read  = dw_ufs_read,
    .write = dw_ufs_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .valid = {
        .min_access_size = 4,
        .max_access_size = 4,
    },
};

/* UFS UPIU processing */
static void dw_ufs_process_upiu(DwUFSState *s, uint8_t *upiu) {
    uint8_t trans_type = upiu[0] & 0x3F;
    switch (trans_type) {
        case UPIU_TRANSACTION_COMMAND:
            dw_ufs_handle_scsi(s, upiu);
            break;
        case UPIU_TRANSACTION_QUERY_REQ:
            dw_ufs_handle_query(s, upiu);
            break;
    }
}
```

---

## Boot Linux on QEMU ARM64

```bash
# Download or build kernel
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.9.tar.xz
# OR build for ARM64:
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# Download QEMU virt machine firmware (EFI)
apt install qemu-efi-aarch64 qemu-system-arm

# Boot minimal ARM64 VM
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 2G \
  -kernel arch/arm64/boot/Image \
  -append "console=ttyAMA0 root=/dev/vda rw" \
  -drive file=rootfs.ext4,format=raw,if=virtio \
  -nographic

# Debug with GDB
qemu-system-aarch64 ... -s -S &     # -s=:1234, -S=wait for GDB
gdb vmlinux
(gdb) target remote :1234
(gdb) hbreak start_kernel
(gdb) continue
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What is the difference between QEMU TCG mode and KVM mode? |
| **Intermediate** | How does a QEMU device model register MMIO regions? |
| **Advanced** | Describe how you would write a QEMU model for a new DMA controller. |
| **Expert** | How did you validate UFS 4.0 protocol compliance in your QEMU device model? What test vectors did you use? |
