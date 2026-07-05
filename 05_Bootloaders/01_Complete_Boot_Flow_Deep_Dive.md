# Complete Boot Flow Deep Dive — ROM to Application

> **The Most Important Concept in Embedded Linux:**  
> Before your kernel runs, before any driver probes, before your application starts —  
> there is an entire universe of code that runs. Understanding it completely separates  
> an **integration engineer** from a **platform architect**.

---

## The 30,000-Foot View

```
Power ON
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1: ROM Bootloader (BootROM)                              │
│  • Lives in SoC's read-only memory                              │
│  • Cannot be changed by anyone (including you)                  │
│  • Loads the next stage from: NAND, eMMC, SPI Flash, USB, SD   │
│  • Verifies signature if Secure Boot enabled                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ loads
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2: SPL (Secondary Program Loader) / MLO / FSBL           │
│  • Fits in internal SRAM (typically 32-256 KB)                  │
│  • Initializes DDR RAM (external memory controller)             │
│  • Sets up clocks, power rails                                  │
│  • Loads full U-Boot into DDR                                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │ loads
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 3: U-Boot (Universal Bootloader)                         │
│  • Full bootloader running from DDR                             │
│  • Console, environment variables, network, USB                 │
│  • Loads kernel + device tree + initrd                          │
│  • Hands off to kernel via booti/bootm/bootz                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ loads
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4: Linux Kernel                                          │
│  • Decompresses itself (if compressed)                          │
│  • Sets up CPU, MMU, interrupts                                 │
│  • Probes device tree → calls driver probe()                    │
│  • Mounts root filesystem                                       │
│  • Starts PID 1 (init/systemd)                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ executes
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5: User Space                                            │
│  • init / systemd starts services                               │
│  • Applications run                                             │
│  • Your code runs here                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Stage 1: BootROM — The Immutable First Code

### What It Is

The BootROM is code burned into the SoC at manufacturing time. It is the first code that runs after power-on reset. You cannot modify it. You cannot update it. It is the hardware root of trust.

### What It Does (Detailed)

```
1. Power-on reset vector jumps to BootROM
2. Minimal hardware init (PLLs for low-speed clock, watchdog)
3. Read boot mode pins/fuses to determine boot device:
   - Qualcomm SC7180: boot_cfg fuses → USB/eMMC/NAND/SD
   - RK3588: EMMC_BOOT# pin, SDMMC pins
   - TI AM335x: SYSBOOT[4:0] pins
4. Initialize storage controller minimally
5. Read first stage bootloader from storage
6. (If Secure Boot enabled): Verify signature using public key burned in OTP fuses
7. Copy first stage bootloader to internal SRAM
8. Jump to first stage bootloader entry point
```

### BootROM Source Code (Where to Find It)

BootROM is usually proprietary (not public). But you can study:

```bash
# Rockchip BootROM behavior documented in:
# https://opensource.rock-chips.com/wiki_Boot_option

# STM32 BootROM is documented in AN2606 application note
# TI BootROM documented in AM335x Technical Reference Manual

# You can reverse-engineer BootROM behavior by:
# 1. Attach JTAG debugger before boot
# 2. Set breakpoint at reset vector (usually 0x00000000 or 0xFFFF0000)
# 3. Single-step through BootROM code
```

### Boot Device Selection Example (RK3588)

```
RK3588 Boot Priority:
  1. Check BOOT_SARADC value
  2. Check EMMC_BOOT# pin
  3. Try SPI NOR flash at CS0
  4. Try eMMC (HS400 mode, 200MHz)
  5. Try SD card
  6. Try USB (download mode for recovery)

You can force USB download mode by:
  - Holding RECOVERY button during power-on
  - This tells BootROM to use USB Bulk protocol (rkdeveloptool/rkflash)
```

### BootROM Security: OTP Fuses

```c
/* OTP = One-Time Programmable fuses */
/* These are physical fuses inside the chip that you blow permanently */

/* Typical OTP fuse layout for secure boot: */
struct otp_layout {
    uint8_t  secure_boot_enable;    /* bit 0: if set, verify all stages */
    uint8_t  jtag_disable;          /* bit 1: if set, JTAG locked forever */
    uint8_t  hash_type;             /* SHA256/SHA384/SHA512 */
    uint8_t  public_key_hash[32];   /* SHA256 of vendor's signing key */
};

/*
 * WARNING: Blowing OTP fuses is IRREVERSIBLE.
 * If you blow the secure_boot_enable fuse without a valid signing key,
 * your SoC is PERMANENTLY BRICKED.
 * This is why production secure boot is carefully staged:
 * 1. Test hardware (no fuses blown)
 * 2. Pre-production (test signing key)
 * 3. Production (real signing key, fuses blown at factory)
 */
```

---

## Stage 2: SPL — The DDR Initializer

### Why SPL Exists

When BootROM hands off control, the SoC has only **internal SRAM** available (typically 256KB-4MB). External DDR RAM is not initialized yet. Full U-Boot is 500KB-2MB and needs to run from DDR.

SPL solves this: it's a tiny bootloader that fits in SRAM, initializes DDR, then loads full U-Boot into DDR.

### SPL Memory Map (Example: RK3588)

```
Internal SRAM: 0xFF000000 - 0xFF3FFFFF (4MB)
  BootROM code:    0xFF000000 - 0xFF07FFFF (512KB)
  SPL runs here:   0xFF080000 - 0xFF2FFFFF (2MB)
  BootROM stack:   0xFF3FF000 - 0xFF3FFFFF (4KB)

After SPL initializes DDR:
  U-Boot relocates: 0x00200000 (DDR) ← SPL loads U-Boot here
  Device Tree:      0x01000000
  Kernel:           0x02000000
```

### SPL Source Code Walkthrough (U-Boot SPL)

```bash
# SPL source in U-Boot:
ls u-boot/common/spl/
# spl.c          ← entry point
# spl_fat.c      ← load from FAT filesystem
# spl_mmc.c      ← load from eMMC/SD
# spl_nand.c     ← load from NAND
# spl_net.c      ← load over network (TFTP)
```

```c
/* u-boot/common/spl/spl.c — SPL entry point */
void board_init_r(gd_t *dummy, ulong dummy_addr)
{
    /* board_init_r is called after BSS clear and stack setup */
    
    /* Step 1: Serial console */
    debug(">>spl:board_init_r()\n");
    
    /* Step 2: Timer init */
    timer_init();
    
    /* Step 3: Board-specific early init (clocks, regulators) */
    board_early_init_r();
    
    /* Step 4: DDR initialization — THE CRITICAL STEP */
    /* This calls the board-specific DRAM init */
    dram_init();
    /* After this returns, DDR is functional and we can use it */
    
    /* Step 5: Load next boot stage */
    /* Try each boot device in order */
    for (i = 0; i < ARRAY_SIZE(spl_boot_list); i++) {
        boot_device = spl_boot_list[i];
        if (spl_load_image(boot_device) == 0)
            break;   /* successfully loaded */
    }
    
    /* Step 6: Jump to U-Boot proper (or kernel if direct boot) */
    jump_to_image_no_args(&spl_image);
    /* This does NOT return */
}
```

### DDR Initialization Deep Dive

DDR initialization is the hardest part of SPL. It involves:

```c
/* Conceptual DDR init sequence (varies by SoC and DDR type) */
void dram_init(void)
{
    /* 1. Enable DDR controller clock */
    clk_enable(DDR_CTRL_CLK);
    
    /* 2. Configure DDR PLL (set DDR frequency) */
    /* Example: LPDDR5 at 3200MT/s for RK3588 */
    pll_set_rate(DPLL, 1600000000);   /* 1600MHz × 2 = 3200MT/s */
    
    /* 3. Reset DDR PHY and controller */
    reset_assert(DDR_PHY_RESET);
    reset_assert(DDR_CTRL_RESET);
    udelay(10);
    reset_deassert(DDR_PHY_RESET);
    reset_deassert(DDR_CTRL_RESET);
    
    /* 4. Configure DDR PHY (signal timing, impedance) */
    /* These values come from board schematic + JEDEC spec */
    phy_write(PHY_DX_GCR0, 0x40200204);   /* I/O configuration */
    phy_write(PHY_PGCR1,   0x0200E0E3);   /* PLL config */
    
    /* 5. Run DDR PHY training */
    /* Training calibrates delays for: write leveling, read/write DQ */
    ret = phy_training();
    if (ret)
        panic("DDR training failed!");
    
    /* 6. Configure DRAM controller timing */
    /* tRCD, tRP, tRAS, tRC, CL etc. from JEDEC or manufacturer spec */
    ctrl_write(DRAMTMG0, PACK(tRAS_min, tRAS_max, tFAW, tWR));
    ctrl_write(DRAMTMG1, PACK(tRC, tRCD, tRRD));
    
    /* 7. Initialize DRAM (MRS commands) */
    dmc_send_mrs(0, MR1_VALUE);  /* Mode Register Set commands */
    dmc_send_mrs(1, MR2_VALUE);
    
    /* 8. Verify DDR works */
    test_ddr_rw((void *)CONFIG_SYS_SDRAM_BASE, 0x1000);
}
```

**Lab Exercise: Observe DDR training output**

```bash
# Add this to your SPL debug output
CONFIG_DEBUG_UART=y
CONFIG_SPL_SERIAL=y
CONFIG_LOGLEVEL=7

# You'll see something like:
# DDR: init start
# DDR: LPDDR5 3200MT/s training...
# DDR: Write leveling PASS
# DDR: Read DQS training PASS  
# DDR: Write DQ training PASS
# DDR: 16 GiB
```

---

## Stage 3: U-Boot — The Full Bootloader

### U-Boot Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    U-Boot Architecture                  │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Console    │  │  Environment │  │  Boot Logic   │  │
│  │  (UART/USB) │  │  (env vars)  │  │  (bootcmd)    │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Storage    │  │  Network     │  │  USB          │  │
│  │  eMMC/SD/   │  │  TFTP/NFS/   │  │  DFU/fastboot │  │
│  │  NAND/SPI   │  │  HTTP        │  │  mass storage │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Driver Model (DM) — mirrors Linux driver model │    │
│  │  GPIO, Pinctrl, Clock, Regulator, I2C, SPI, USB │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Board Support (board/vendor/boardname/)        │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### U-Boot Source Code Walkthrough

```bash
# Key files to understand:
u-boot/
├── arch/arm/cpu/armv8/    # ARM64-specific code
│   ├── start.S            # Very first instructions after reset
│   ├── cpu.c              # CPU initialization
│   └── cache.S            # Cache operations
├── arch/arm/lib/
│   └── crt0_64.S          # C runtime setup (BSS clear, stack)
├── common/
│   ├── main.c             # main_loop() — interactive console
│   ├── bootm.c            # boot kernel from memory
│   ├── image.c            # FIT image parsing
│   └── env/               # Environment variable storage
├── drivers/               # All hardware drivers
├── board/                 # Board-specific code
│   └── rockchip/rock5b/   # Radxa Rock 5B board code
├── configs/               # Kconfig defconfigs
│   └── rock5b_defconfig   # Radxa Rock 5B config
└── include/configs/       # Platform-specific defines
```

### U-Boot Initialization Chain

```c
/* arch/arm/cpu/armv8/start.S — Entry point (reset vector) */
_start:
    b reset          /* Branch to reset handler */

reset:
    /* 1. Set CPU to AArch64 EL3 */
    /* 2. Initialize CPU state (disable interrupts, clear flags) */
    /* 3. Set up initial stack */
    /* 4. Jump to _main */

/* arch/arm/lib/crt0_64.S */
ENTRY(_main)
    /* 1. Set up initial stack (in SRAM) */
    /* 2. Clear BSS section */
    /* 3. Call board_init_f() */

/* common/board_f.c — board_init_f() */
void board_init_f(ulong boot_flags)
{
    /* Run list of init functions */
    static const init_fnc_t init_sequence_f[] = {
        setup_mon_len,          /* calculate monitor length */
        initf_malloc,           /* malloc init */
        log_init,               /* logging init */
        initf_bootstage,        /* boot timing */
        initf_console_record,   /* early console */
        arch_cpu_init,          /* CPU-specific init */
        mach_cpu_init,          /* SoC-specific CPU init */
        initf_dm,               /* Driver Model init */
        board_early_init_f,     /* board-specific early init */
        timer_init,             /* timer */
        env_init,               /* environment */
        init_baud_rate,         /* baud rate from env */
        serial_init,            /* UART fully initialized */
        console_init_f,         /* console */
        display_options,        /* print banner */
        dram_init,              /* DRAM size detection */
        reserve_round_4k,       /* align reserve area */
        reserve_mmu,            /* MMU tables */
        reserve_video,          /* video buffer */
        reserve_uboot,          /* U-Boot itself */
        reserve_malloc,         /* malloc pool */
        ...
        setup_reloc,            /* setup relocation */
        NULL,
    };
    
    for (init_fnc_ptr = init_sequence_f; *init_fnc_ptr; ++init_fnc_ptr) {
        if ((*init_fnc_ptr)()) 
            hang();   /* if any init fails, halt */
    }
    
    /* Relocate U-Boot to top of DRAM */
    relocate_code(gd->relocaddr);
}
```

### FIT Image — How Kernel is Packaged

FIT (Flattened Image Tree) is the modern way to package kernel + DTB + initramfs into one signed image.

```bash
# FIT image source (its.file):
cat boot.its
```

```
/dts-v1/;

/ {
    description = "Radxa Rock 5B+ Linux Boot Image";
    #address-cells = <1>;
    
    images {
        kernel {
            description = "Linux Kernel";
            data = /incbin/("Image");           /* ARM64 kernel image */
            type = kernel;
            arch = arm64;
            os = linux;
            compression = gzip;                  /* or none/lzma */
            load = <0x02000000>;                 /* load address in DDR */
            entry = <0x02000000>;               /* entry point */
            hash-1 {
                algo = sha256;
            };
        };
        
        fdt-rock5b {
            description = "Flattened Device Tree for Rock 5B";
            data = /incbin/("rk3588-rock-5b.dtb");
            type = flat_dt;
            arch = arm64;
            compression = none;
            load = <0x01000000>;
            hash-1 {
                algo = sha256;
            };
        };
        
        ramdisk {
            description = "initramfs";
            data = /incbin/("initramfs.cpio.gz");
            type = ramdisk;
            arch = arm64;
            os = linux;
            compression = gzip;
            hash-1 {
                algo = sha256;
            };
        };
    };
    
    configurations {
        default = "config-rock5b";
        
        config-rock5b {
            description = "Radxa Rock 5B+";
            kernel = "kernel";
            fdt = "fdt-rock5b";
            ramdisk = "ramdisk";
            signature {
                algo = sha256,rsa2048;      /* signature algorithm */
                key-name-hint = "dev";       /* signing key name */
                sign-images = "kernel", "fdt", "ramdisk";
            };
        };
    };
};
```

```bash
# Build FIT image
mkimage -f boot.its boot.itb

# Verify FIT image contents
mkimage -l boot.itb
# FIT description: Radxa Rock 5B+ Linux Boot Image
# Image 0 (kernel)
#  Description: Linux Kernel
#  Type: Kernel Image
#  Compression: gzip compressed
#  Load Address: 0x02000000
#  Entry Point: 0x02000000
#  Hash algo: sha256
#  Hash value: ab34ef...

# Sign FIT image (for secure boot)
mkimage -F -k keys/ -K u-boot.dtb -r boot.itb
```

### U-Boot Environment Variables

```bash
# Default U-Boot boot command (bootcmd)
# This is what U-Boot executes automatically after delay

printenv bootcmd
# run distro_bootcmd;
# → tries bootscripts from: mmc0, mmc1, usb0, pxe, dhcp

# Manual boot sequence
# Step 1: Load kernel from eMMC partition 1
mmc dev 0                                       # select eMMC device 0
load mmc 0:1 ${kernel_addr_r} /boot/Image       # load kernel to RAM

# Step 2: Load device tree
load mmc 0:1 ${fdt_addr_r} /boot/rk3588-rock-5b.dtb

# Step 3: Set kernel command line
setenv bootargs "root=/dev/mmcblk0p2 rw rootwait console=ttyS2,1500000 earlycon"

# Step 4: Boot!
booti ${kernel_addr_r} - ${fdt_addr_r}
# booti: boot ARM64 Image
# -: no ramdisk
# ${fdt_addr_r}: device tree address

# Standard addresses (from config header):
kernel_addr_r = 0x02000000
fdt_addr_r    = 0x01000000
ramdisk_addr_r = 0x04000000
```

### Adding a Custom U-Boot Command

```c
/* u-boot/cmd/my_cmd.c — custom U-Boot command */
#include <command.h>
#include <dm.h>

static int do_my_cmd(struct cmd_tbl *cmdtp, int flag, 
                     int argc, char *const argv[])
{
    if (argc < 2) {
        return CMD_RET_USAGE;
    }
    
    printf("Custom command called with arg: %s\n", argv[1]);
    
    /* Access hardware from U-Boot */
    if (strcmp(argv[1], "ddr_test") == 0) {
        /* DDR stress test */
        uint32_t *mem = (uint32_t *)0x02000000;
        printf("DDR test at 0x02000000...");
        for (int i = 0; i < 1024; i++) {
            mem[i] = i;
        }
        for (int i = 0; i < 1024; i++) {
            if (mem[i] != i) {
                printf("FAIL at offset %d\n", i);
                return CMD_RET_FAILURE;
            }
        }
        printf("PASS\n");
    }
    
    return CMD_RET_SUCCESS;
}

/* Macro to register command */
U_BOOT_CMD(
    mycmd, 2, 0, do_my_cmd,
    "my custom command",
    "mycmd ddr_test  - run DDR stress test\n"
);
```

---

## Stage 4: Linux Kernel Boot

### Kernel Image Formats

| Format | Architecture | Description |
|--------|-------------|-------------|
| `Image` | ARM64 | Uncompressed kernel (most common for ARM64) |
| `zImage` | ARM32 | Self-decompressing kernel |
| `bzImage` | x86 | Big zImage, standard on PC |
| `uImage` | Any | Legacy U-Boot wrapped image |
| `Image.gz` | ARM64 | Gzip compressed Image |

### ARM64 Kernel Entry Point

```c
/* arch/arm64/kernel/head.S — Very first kernel code */

__HEAD
_head:
    /*
     * DO NOT MODIFY. Image header expected by Linux boot-loaders.
     * The following magic is required in the first 8 bytes.
     */
    b   primary_entry               /* Branch to kernel start */
    .long   0                       /* Reserved (was endian flag) */
    le64sym _kernel_offset_le       /* Image load offset */
    le64sym _kernel_size_le         /* Effective size of kernel image */
    le64sym _kernel_flags_le        /* Informative flags */
    .quad   0                       /* Reserved */
    .quad   0                       /* Reserved */
    .quad   0                       /* Reserved */
    .ascii  ARM64_IMAGE_MAGIC       /* Magic number */
    .long   pe_header - _head       /* Offset to PE COFF header */
pe_header:
    /* PE header for EFI boot */
    ...

primary_entry:
    /* Step 1: Preserve boot parameters (x0 = DTB physical address!) */
    
    /* Step 2: Determine if we're primary or secondary CPU */
    bl  record_mmu_state
    
    /* Step 3: Disable caches and MMU */
    
    /* Step 4: Map the kernel (create identity mapping) */
    bl  __cpu_setup         /* CPU-specific setup */
    
    /* Step 5: Enable MMU */
    bl  __primary_switch
    
    /* Now running with virtual addresses */
    bl  start_kernel        /* Jump to C code! */
```

### start_kernel() — The Main C Entry

```c
/* init/main.c — start_kernel() — the most important function in Linux */
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
    char *command_line;
    char *after_dashes;
    
    set_task_stack_end_magic(&init_task);   /* set up PID 1 (init) */
    smp_setup_processor_id();               /* set CPU ID */
    
    /* Architecture-specific setup (MMU, caches, exceptions) */
    setup_arch(&command_line);
    /*
     * setup_arch() does ENORMOUS amount of work:
     * - Parses device tree (unflatten_device_tree)
     * - Sets up memory (memblock allocator)
     * - Sets up page tables
     * - Sets up exception handlers
     * - Initializes clocksource
     */
    
    /* Parse kernel command line */
    setup_command_line(command_line);
    parse_early_param();   /* process "earlycon=" etc */
    
    /* Memory subsystem */
    build_all_zonelists(NULL);
    page_alloc_init();
    mm_init();             /* buddy allocator, slab allocator */
    kmem_cache_init();     /* slab cache for kernel objects */
    
    /* Interrupt subsystem */
    irq_init();
    init_IRQ();
    tick_init();
    
    /* Timers */
    hrtimers_init();
    timekeeping_init();
    
    /* Process scheduler */
    sched_init();
    
    /* File systems */
    vfs_caches_init();
    
    /* Networking */
    net_ns_init();
    
    /* Devices */
    driver_init();     /* bus types, device model */
    
    /* Calibrate delays */
    calibrate_delay();
    
    /* Start init process */
    arch_call_rest_init();
    /* → rest_init() → kernel_init() → execve("/sbin/init") */
}
```

### Device Tree Processing

```c
/* drivers/of/fdt.c — unflatten_device_tree() */

/*
 * The DTB (Device Tree Blob) is a binary file passed by U-Boot in register x0.
 * The kernel converts this binary blob into a tree of of_node structs.
 *
 * Binary DTB format:
 *   magic: 0xD00DFEED
 *   totalsize: total size in bytes
 *   off_dt_struct: offset to structure block
 *   off_dt_strings: offset to strings block
 *   ...
 *
 * After unflattening, each DT node becomes:
 * struct device_node {
 *     char *name;           "my-counter"
 *     char *full_name;      "/soc/counter@fe650000"
 *     struct property *properties;  → list of key-value pairs
 *     struct device_node *parent;
 *     struct device_node *child;
 *     struct device_node *sibling;
 * };
 */

void __init unflatten_device_tree(void)
{
    __unflatten_device_tree(initial_boot_params, NULL,
                            &of_root, early_init_dt_alloc_memory_arch,
                            false);
    /* After this: of_root points to root "/" node */
    /* All DT nodes are accessible via of_find_node_by_path() etc. */
}
```

### Driver Probe Chain

```c
/*
 * After device model init (driver_init()), the kernel walks the DT
 * and creates platform_device for each DT node with "compatible" property.
 * Then it matches each device against registered drivers.
 */

/* platform_bus_type match function */
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);
    
    /* 1. Try OF (Device Tree) match first */
    if (of_driver_match_device(dev, drv))   /* compare compatible strings */
        return 1;
    
    /* 2. Try ACPI match */
    if (acpi_driver_match_device(dev, drv))
        return 1;
    
    /* 3. Try id_table match */
    if (pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;
    
    /* 4. Fall back to name match */
    return (strcmp(pdev->name, drv->name) == 0);
}

/* When match succeeds, probe() is called */
static int platform_drv_probe(struct device *_dev)
{
    struct platform_driver *drv = to_platform_driver(_dev->driver);
    struct platform_device *dev = to_platform_device(_dev);
    int ret;
    
    ret = drv->probe(dev);   /* YOUR driver's probe() function */
    
    if (ret)
        dev_info(_dev, "probe failed: %d\n", ret);
    
    return ret;
}
```

---

## Stage 5: Init Process and User Space

### PID 1 — The Process Tree Root

```c
/* init/main.c — kernel_init() */
static int __ref kernel_init(void *unused)
{
    /* 1. Wait for all kernel threads to be ready */
    kernel_init_freeable();
    
    /* 2. Free init memory (code only needed for boot) */
    free_initmem();
    
    /* 3. Execute /sbin/init (or whatever is specified in kernel cmdline) */
    if (ramdisk_execute_command) {
        ret = run_init_process(ramdisk_execute_command);
    }
    
    /* Fallback list: try each in order */
    const char *init_filename[] = {
        "/sbin/init",
        "/etc/init",
        "/bin/init",
        "/bin/sh",
        NULL,
    };
    
    for (p = init_filename; *p; p++) {
        ret = run_init_process(*p);
        if (!ret)
            break;
    }
    
    panic("No working init found. Tried: %s. "
          "Try passing init= option to kernel.", execute_command);
}
```

### systemd Boot Process

```
PID 1 = systemd
  │
  ├─ systemd-journald (logging)
  ├─ systemd-udevd (device manager, triggers udev rules)
  ├─ systemd-networkd (network)
  ├─ systemd-resolved (DNS)
  │
  ├─ Basic target (filesystem mounts, getty)
  ├─ Network target
  ├─ Multi-user target
  └─ Your service (myapp.service)
```

```bash
# Create a systemd service for your embedded app
cat /etc/systemd/system/my-embedded-app.service
```

```ini
[Unit]
Description=My Embedded Application
After=network.target
Requires=my-driver.service

[Service]
Type=simple
ExecStart=/usr/bin/my_embedded_app --config /etc/myapp.conf
Restart=on-failure
RestartSec=5

# Resource limits
MemoryLimit=256M
CPUQuota=50%

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

---

## Complete Boot Time Analysis

```bash
# Measure boot time for each stage

# 1. U-Boot timing
setenv bootdelay 0
# Add to board/rockchip/rock5b/rock5b.c:
DECLARE_GLOBAL_DATA_PTR;
int board_late_init(void) {
    printf("U-Boot ready: %lu ms since power-on\n", 
           get_timer(0) - gd->reloc_off);
}

# 2. Kernel boot timing (built-in)
# Add to bootargs:
bootargs="... printk.time=1 initcall_debug"

# See time for each initcall:
dmesg | grep "initcall.*returned"
# [    0.234567] initcall platform_bus_init+0x0/0x50 returned 0

# 3. systemd boot timing
systemd-analyze
# Startup finished in 1.234s (firmware) + 2.345s (loader) + 3.456s (kernel) + 4.567s (userspace) = 11.602s

systemd-analyze blame
# 3.456s my-slow-service.service
# 1.234s network.service
# 0.567s systemd-udevd.service

# 4. Add custom timestamp in your application
#include <sys/time.h>
struct timeval tv;
gettimeofday(&tv, NULL);
printf("App started at: %ld.%06ld\n", tv.tv_sec, tv.tv_usec);
# Compare to /proc/uptime to get time since boot
```

---

## Boot Flow for Qualcomm Platforms (Ravi's Background)

```
Qualcomm SC7180/SC7280 Boot Flow:
  ┌──────────────────────────────────────────────┐
  │  BOOTROM (PBL — Primary Boot Loader)          │
  │  • Internal to SoC, SHA256-signed             │
  │  • Loads XBL from storage                     │
  └───────────────────┬──────────────────────────┘
                      │
                      ▼
  ┌──────────────────────────────────────────────┐
  │  XBL (eXtensible Boot Loader)                 │
  │  = Qualcomm's SPL equivalent                  │
  │  • UEFI-based (EDKII)                         │
  │  • Initializes LPDDR4/5, UFS, USB             │
  │  • Loads ABL (Android Boot Loader)            │
  └───────────────────┬──────────────────────────┘
                      │
                      ▼
  ┌──────────────────────────────────────────────┐
  │  ABL (Android Boot Loader)                    │
  │  = Qualcomm's U-Boot equivalent               │
  │  • Based on UEFI / LK (Little Kernel)         │
  │  • Loads boot.img (kernel + dtb + ramdisk)    │
  │  • Verified boot (Android Verified Boot 2.0)  │
  └───────────────────┬──────────────────────────┘
                      │
                      ▼
  ┌──────────────────────────────────────────────┐
  │  Linux Kernel (same as all platforms)         │
  └──────────────────────────────────────────────┘

ChromeOS (your Coreboot/Depthcharge experience):
  PBL → Coreboot (SPL equivalent) → Depthcharge (kernel loader) → Linux
```

---

## Labs

### Lab 1: Watch Every Boot Stage on Radxa 5B+

```bash
# On host: connect UART debug cable (4-pin header on board)
# Baud rate: 1500000 (1.5 Mbaud — Rockchip uses this)

sudo tio /dev/ttyUSB0 -b 1500000

# Power on board. You'll see:
# DDR Version V1.11 ...    ← SPL
# LPDDR5 ...
# Trying to boot from MMC1 ...
#
# U-Boot 2024.01 ...        ← U-Boot
# Model: Radxa ROCK 5B+
# ...
# Starting kernel ...        ← kernel handoff
#
# [    0.000000] Booting Linux on physical CPU 0x0 ...  ← kernel
```

### Lab 2: Interrupt U-Boot and Explore

```bash
# During U-Boot countdown (hit any key):
# Autoboot in 3 seconds... (press any key to stop)
[Enter]

# Explore
=> help
=> printenv
=> bdinfo          # board info: DDR size, clock speeds
=> mmc info        # eMMC info
=> mmc part        # partition table

# Load and boot manually:
=> setenv bootargs "root=/dev/mmcblk1p5 rw rootwait console=ttyS2,1500000"
=> load mmc 1:3 ${kernel_addr_r} /boot/Image
=> load mmc 1:3 ${fdt_addr_r} /boot/dtbs/rockchip/rk3588-rock-5b.dtb
=> booti ${kernel_addr_r} - ${fdt_addr_r}
```

### Lab 3: Add initcall_debug to See Driver Probe Order

```bash
# Add to U-Boot bootargs:
setenv extraargs "initcall_debug printk.time=1"

# Boot and observe:
dmesg | grep "initcall"
# [    0.023456] initcall i2c_init+0x0 returned 0 after 123 usecs
# [    0.045678] initcall rockchip_i2c_driver_init+0x0 returned 0 after 45 usecs
# [    0.067890] initcall rk3588_pinctrl_init+0x0 returned 0 after 234 usecs
```

---

## Interview Questions

| Level | Question | Key Answer |
|-------|----------|-----------|
| **Beginner** | What is a bootloader? Why do we need it? | Initializes hardware before OS; kernel can't run without initialized DDR/clocks |
| **Beginner** | What is the difference between SPL and U-Boot? | SPL = tiny, fits SRAM, inits DDR; U-Boot = full bootloader, runs from DDR |
| **Intermediate** | What is a FIT image? What are its advantages? | Container for kernel+DTB+ramdisk+signatures; supports multiple configs, cryptographic signing |
| **Intermediate** | How does U-Boot pass the device tree to the kernel? | U-Boot places DTB in memory, puts address in register x0, kernel reads x0 at entry |
| **Intermediate** | What does `booti` vs `bootm` do? When do you use each? | booti = ARM64 Image format; bootm = legacy uImage format; booti is modern standard |
| **Advanced** | Walk me through what happens between power-on and start_kernel() | BootROM → SPL (DDR init) → U-Boot (FIT load) → kernel entry (setup_arch → start_kernel) |
| **Advanced** | How does Secure Boot work in U-Boot? | Keys compiled in, FIT image signed, mkimage verifies signatures before booting |
| **Advanced** | What is EPROBE_DEFER? At what boot stage does it occur? | Driver returns -EPROBE_DEFER when dependency not ready; happens during initcall, driver re-probed later |
| **Expert** | How would you reduce boot time from 10s to 3s? | Parallel initcalls, skip unnecessary drivers, preload rootfs, reduce kernel init, fastboot systemd |
| **Expert** | Explain Qualcomm's XBL/ABL vs standard U-Boot SPL | Both initialize DDR; XBL is UEFI-based, ABL uses LK, vs U-Boot uses its own driver model |
