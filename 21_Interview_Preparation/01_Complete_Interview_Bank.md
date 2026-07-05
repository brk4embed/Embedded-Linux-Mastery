# Complete Interview Preparation Bank — Embedded Linux, Kernel, AI Engineering

> **For Ravi:** This bank covers every topic you'll face in technical interviews  
> at companies like Qualcomm, Google, Arm, Linaro, Red Hat, and top-tier embedded firms.  
> Study these until you can answer every Expert-level question confidently.

---

## How to Use This File

1. Cover the answer column — try to answer from memory
2. Score yourself: 0 (no idea) → 1 (rough) → 2 (confident) → 3 (could teach it)
3. Focus daily practice on your 2s (nearly there)
4. For each Expert question: answer it AND draw an architecture diagram

---

## Section 1: C Programming and Systems

| Level | Question | Answer |
|-------|----------|--------|
| **B** | What is the difference between `struct` and `union`? | struct: all fields exist simultaneously; union: all fields share same memory, only one at a time |
| **B** | What does `volatile` mean? When do you use it? | Prevents compiler from caching the value in register; use for HW registers, ISR-shared variables |
| **B** | What is `const` correctness? | `const int *p` = ptr to const int (can't change *p); `int * const p` = const ptr (can't change p); `const int * const p` = both |
| **I** | Explain the difference between stack and heap. When would embedded code prefer stack? | Stack: automatic, fast, limited; Heap: dynamic, fragmentation risk; Embedded prefers stack or static allocation to avoid fragmentation and malloc latency |
| **I** | What is a function pointer? Write an example showing callback registration. | `typedef void (*cb_t)(int); void register_callback(cb_t fn, int arg);` Used in Linux as file_operations, platform_driver ops |
| **I** | What are the `__packed`, `__aligned(N)` attributes? | Compiler directives; __packed: no padding between fields (for hardware registers/protocols); __aligned(4): force 4-byte alignment |
| **A** | What is undefined behavior in C? Give 3 examples. | Signed integer overflow, reading uninitialized variable, NULL dereference; UB means compiler can do anything; avoid with -fsanitize=undefined |
| **A** | Explain strict aliasing. How does it affect kernel code? | Compiler assumes different-type pointers don't alias; kernel uses `__may_alias` and `container_of` carefully to avoid violations |
| **E** | What is the C11 memory model? How does it relate to kernel atomics? | Defines sequential consistency, acquire/release semantics; kernel uses `smp_load_acquire`, `smp_store_release` to implement these without full barriers |

---

## Section 2: Boot Flow

| Level | Question | Answer |
|-------|----------|--------|
| **B** | What is a bootloader? Name 3 examples. | Software that runs before OS; U-Boot, Coreboot, GRUB |
| **B** | What is SPL? Why is it needed? | Secondary Program Loader; fits in internal SRAM to initialize DDR before full U-Boot loads |
| **I** | What is a FIT image? What are its advantages over uImage? | Flattened Image Tree: container for kernel+DTB+initramfs+signatures; supports multiple configs, cryptographic signing, verified boot |
| **I** | How does U-Boot pass the device tree to the Linux kernel? | Places DTB in memory, puts physical address in register x0 (ARM64). Kernel reads x0 at entry point in head.S |
| **I** | What is `bootargs` in U-Boot? Give an example. | Kernel command line string: `root=/dev/mmcblk0p2 rw rootwait console=ttyS2,1500000 earlycon` |
| **A** | Walk me through what happens between power-on and `start_kernel()`. | BootROM → SPL (DDR init) → U-Boot (FIT load, DT setup) → kernel head.S (setup_arch, unflatten_device_tree) → start_kernel() |
| **A** | How does Secure Boot work in the U-Boot → Linux chain? | Keys compiled into U-Boot; FIT signed with RSA key; U-Boot verifies signature before booting; hash of public key burned in OTP fuses |
| **E** | Explain Qualcomm's XBL/ABL boot flow vs standard U-Boot SPL/U-Boot | XBL = UEFI-based SPL equivalent; ABL = LK/UEFI-based U-Boot equivalent; both chains ultimately hand off to Linux at same point |
| **E** | How would you add a new boot device to U-Boot SPL? | Implement `struct spl_boot_device`, add to `spl_boot_list`, implement `spl_load_image_<device>()`, register with `SPL_LOAD_IMAGE_METHOD` |

---

## Section 3: Linux Kernel Internals

| Level | Question | Answer |
|-------|----------|--------|
| **B** | What is a kernel module? How do you load and unload it? | Dynamically loadable code; `insmod`/`modprobe` to load; `rmmod` to unload; `lsmod` to list |
| **B** | What is the difference between `kmalloc` and `vmalloc`? | kmalloc: physically contiguous, must be small; vmalloc: virtually contiguous, can be large; DMA needs kmalloc |
| **B** | What is `GFP_KERNEL` vs `GFP_ATOMIC`? | GFP_KERNEL: can sleep, for process context; GFP_ATOMIC: cannot sleep, for IRQ/spinlock context |
| **I** | Explain the Linux scheduler. What algorithm does it use? | CFS (Completely Fair Scheduler); uses red-black tree ordered by vruntime; lowest vruntime runs next; fair CPU time based on nice values |
| **I** | What is `copy_to_user`? Why can't you use `memcpy` instead? | Safely copies to user-space address; validates pointer, handles page faults; memcpy would OOPS on bad user pointer |
| **I** | What is the difference between softirq, tasklet, and workqueue? | softirq: runs in interrupt context, must be compiled in; tasklet: built on softirq, serialized; workqueue: process context, can sleep |
| **A** | Explain RCU. When would you use it over a spinlock? | Read-Copy-Update: lock-free for read-heavy; readers never wait; writers make copy, update pointer, wait for grace period; use for lookup tables read much more than written |
| **A** | What is `lockdep`? How does it detect deadlocks? | Kernel lock dependency tracker; builds graph of lock → lock dependencies; reports if cycle detected (would cause deadlock) |
| **A** | Explain the kernel's memory zones (DMA, Normal, HighMem). | DMA: low addresses accessible by old DMA hardware; Normal: kernel-mapped memory; HighMem: unmapped (only on 32-bit); ARM64 has no HighMem |
| **E** | Walk through a page fault: what happens from fault to data loaded? | CPU triggers exception → `do_page_fault()` → find VMA → check permissions → allocate page → call fault handler (file/anon) → map into page table → return |
| **E** | How does the slab allocator work? What are SLAB vs SLUB vs SLOB? | Manages fixed-size object caches to avoid fragmentation; SLUB: default (simpler, less memory), SLAB: older (more debugging), SLOB: tiny systems |

---

## Section 4: Device Drivers

| Level | Question | Answer |
|-------|----------|--------|
| **B** | What is `probe()`? When is it called? | Called by kernel when a device matches a driver (by compatible string or name); returns 0 success, negative error |
| **B** | What does `devm_kzalloc` do differently from `kzalloc`? | Device-managed: auto-freed on device removal or probe failure; prevents resource leaks |
| **B** | What is `platform_set_drvdata`? Why do we use it? | Stores driver's private data in device struct; allows retrieval in IRQ handlers, sysfs, suspend/resume via `platform_get_drvdata` |
| **I** | Explain the Device Tree. What problem does it solve? | Hardware description language; decouples kernel from board-specific code; describes hardware topology without hardcoding in C |
| **I** | What is `of_match_table`? How does the kernel use it? | Array of `{.compatible = "vendor,device"}` entries; kernel compares DT `compatible` property against all registered drivers; calls probe() on match |
| **I** | What is `-EPROBE_DEFER`? When should a driver return it? | Return when a dependency (clock, regulator, GPIO) is not yet available; kernel reschedules probe; used by PMIC/clock drivers not ready at boot |
| **A** | Explain how an I2C transaction works from driver to wire. | Driver calls `i2c_transfer()`; I2C subsystem calls `master_xfer()` of adapter driver; adapter writes START+ADDR+DATA+STOP to I2C controller registers; controller sends on wire |
| **A** | What is DMA coherence? When do you need `dma_sync_single_for_*`? | CPU cache vs device seeing same memory; coherent: always in sync; streaming: explicitly sync before device reads (for_device) and before CPU reads (for_cpu) |
| **A** | How does the kernel's `device_add_groups` create sysfs attributes? | Creates files in `/sys/devices/.../` by calling `sysfs_create_files()`; each `DEVICE_ATTR_RO/WO/RW` maps to a show/store callback |
| **E** | Design a complete platform driver with PM, IRQ, DMA, and sysfs for a custom UART. | probe(): ioremap, clk_prepare_enable, regulator_enable, request_irq, dma_request_chan, register serial port, add sysfs; remove(): reverse; suspend(): save registers, disable clock; resume(): restore |

---

## Section 5: Embedded Linux System Design

| Level | Question | Answer |
|-------|----------|--------|
| **B** | What is a Device Tree blob (DTB)? How is it different from DTS? | DTS: source text; DTB: compiled binary (output of `dtc`); kernel uses DTB; you write DTS |
| **B** | What is `rootfs`? What formats can it be? | Root filesystem: all files the OS needs; formats: ext4, squashfs, JFFS2, UBIFS, NFS, initramfs |
| **I** | What is Yocto? How does it differ from Buildroot? | Yocto: full Linux distro build system (layers, recipes, metadata); Buildroot: simpler, faster; Yocto for complex products; Buildroot for quick prototypes |
| **I** | Explain A/B partition scheme for OTA updates. | Two copies of kernel+rootfs (A and B); boot from A; update B while A runs; set B as active; reboot; if B fails rollback to A; enables zero-downtime updates |
| **A** | How would you reduce system boot time from 15s to 3s? | Parallel initcalls (multi-core), remove unused drivers, compressed rootfs in RAM, initramfs instead of disk, reduce kernel init (CONFIG_EMBEDDED), systemd parallelism |
| **A** | Explain UFS command queueing (command queue vs legacy mode). | Legacy: one command at a time; Command Queue (UFS 3.1+): up to 32 commands in flight; host picks from queue, UFS device reorders for efficiency |
| **E** | Design the complete software stack for a product with: QSPI flash + eMMC + camera + neural network accelerator + LTE modem. | BootROM→SPL(QSPI)→U-Boot→Kernel; MTD/NAND for QSPI; eMMC for rootfs; V4L2+ISP for camera; vendor SDK/tflite for NPU; ofono/ModemManager for LTE; systemd services |

---

## Section 6: Qualcomm / ChromeOS Specific (Ravi's Background)

| Level | Question | Answer |
|-------|----------|--------|
| **I** | What is QFPROM? What does the driver do? | Qualcomm Fuse-Programmable ROM; stores OTP fuses (device ID, security config); driver reads fuse values; Ravi has upstreamed this |
| **I** | What is QTEE? How does it differ from upstream OP-TEE? | Qualcomm Trusted Execution Environment; proprietary extension of TrustZone; has Qualcomm-specific secure services; different APIs than OP-TEE |
| **A** | Explain Depthcharge. What role does it play in Chromebook boot? | Google's Chromebook bootloader (replaces GRUB/U-Boot); runs on top of Coreboot; verifies ChromeOS kernel with Verified Boot; can boot Linux directly |
| **A** | How does ChromeOS Verified Boot work end-to-end? | GBB (Google Binary Block) has root key; Coreboot verifies Depthcharge; Depthcharge verifies kernel partition signature; each stage signs the next |
| **E** | You found a regression in QXDM logs during SC7280 board bring-up: UFS probes but DMA transfers fail silently. How do you debug? | Check UFS DMA channel assignment (check IOMMU mappings), verify SMMU is configured for UFS, use QXDM UFS log to see UPIU exchanges, use Trace32 to inspect UFS controller DMA state |

---

## Section 7: AI / Embedded AI

| Level | Question | Answer |
|-------|----------|--------|
| **B** | What is quantization in AI? Why do we use it for embedded? | Reduce model weights from FP32 to INT8/INT4; smaller model, less memory, faster inference; slight accuracy loss; essential for edge deployment |
| **B** | What is ONNX? | Open Neural Network Exchange: standard format for ML models; any framework (PyTorch, TensorFlow) can export to ONNX; then convert to TFLite/RKNN/TensorRT |
| **I** | What is the difference between RKNN and TFLite? | RKNN: Rockchip-specific format for their NPU; TFLite: Google's format for any hardware; RKNN ~5-10× faster on RK3588 NPU vs TFLite on CPU |
| **I** | What is KV Cache in LLM inference? | Attention mechanism recomputes K,V matrices each token; KV cache stores them; avoids recomputation; enables fast token generation after slow prefill |
| **A** | Explain the transformer attention mechanism. What is its time complexity? | Attention(Q,K,V) = softmax(QK^T/√d)V; computes similarity between all token pairs; O(n²d) where n=sequence length; why long contexts are expensive |
| **A** | How would you deploy a 7B parameter LLM on a resource-constrained system? | INT4 quantization (4GB vs 14GB); llama.cpp for CPU; 4 threads max (big cores); system offload for large prompts; consider smaller model (3B Q4) for latency |
| **E** | Design an edge AI pipeline: camera → detection → classification → action with <50ms total latency | Camera→ISP: 10ms; YOLOv8n detection on NPU: 25ms; classification on NPU: 5ms; action trigger: 1ms; total: ~41ms; all NPU inference, DMA for frame transfer, no CPU in hot path |

---

## Section 8: Debugging and Tools

| Level | Question | Answer |
|-------|----------|--------|
| **B** | What is `dmesg`? What information does it show? | Kernel ring buffer; shows kernel messages since boot; hardware init, driver probes, errors, warnings |
| **B** | How do you find which kernel module owns a device? | `ls -la /sys/bus/platform/devices/*/driver` → shows symlink to driver |
| **I** | What is KASAN? What bugs does it find? | Kernel Address Sanitizer; finds: use-after-free, out-of-bounds read/write, use-after-scope; adds 2× memory overhead |
| **I** | Explain ftrace `function_graph` tracer. How do you use it? | Traces call graph of kernel functions with timing; `echo function_graph > current_tracer; echo 'driver_probe' > set_graph_function; cat trace` |
| **A** | How do you debug a driver that intermittently fails once per week? | Add precise tracing (ftrace/kprobes), increase log level, KASAN+UBSAN builds, sysrq trace, kgdb breakpoints, hardware watchdog to capture state at failure |
| **A** | What is a kernel oops? What information does it contain? | Non-fatal kernel error; contains: error type, faulting address, register dump, call stack, loaded modules; not necessarily fatal (unlike panic) |
| **E** | A production board has a memory corruption happening after 48 hours. How do you debug? | Kmemleak scan, KASAN (if dev board), kfence (low overhead for production), hardware PMU memory monitoring, add canary values around suspicious allocations, bisect recent commits |

---

## Section 9: System Design Interview Questions

These are architectural questions for Staff/Principal level roles:

```
Q: "Design the software architecture for a smart factory camera system
    with 8 cameras, 30 FPS AI detection, 90-day local storage, and
    zero-downtime OTA updates."

Answer framework:
1. Hardware selection: RK3588 (6 TOPS NPU) + 16GB RAM + NVMe
2. Boot: U-Boot → Linux 6.6 LTS
3. Camera: V4L2 pipeline for 8× MIPI CSI (time-multiplexed or multiple SoCs)
4. AI: YOLOv8 on NPU (30 FPS per camera, NPU shared)
5. Storage: 8TB NVMe, ZFS (checksumming), 90-day rolling
6. OTA: SWUpdate + A/B partitions + Mender cloud
7. Networking: Gigabit + optional 4G backup
8. Security: Secure boot, encrypted storage (dm-crypt), audit logs
9. Monitoring: systemd watchdog, health daemon, alert on anomaly

Follow-up: "What would you change if power was limited to 5W?"
→ Lighter SoC (RK3566), reduce camera count, HEVC compression, NPU-only pipeline
```

---

## Section 10: Behavioral Questions (Often Asked)

| Question | Strong Answer Pattern |
|----------|----------------------|
| "Tell me about a complex bug you debugged" | Use STAR: Situation (SC7280 board bring-up), Task (UFS DMA failures), Action (QXDM → Trace32 → register dump → SMMU misconfiguration), Result (fixed, upstreamed) |
| "How do you approach a new platform you've never seen?" | Read TRM (Technical Reference Manual), find existing working platform as reference, bring up UART first, then core clocks, then storage, then peripherals one by one |
| "Describe your open source contribution process" | QFPROM driver: identified missing upstream support, wrote driver, sent to linux-arm-msm list, addressed 3 rounds of review comments, merged in v5.x |
| "How do you stay current in a fast-moving field?" | LWN.net weekly, kernel mailing lists, GitHub watching key repos, embedded.fm podcast, quarterly deep-dives on new technology |
| "What would you do in your first 30 days at this company?" | Week 1: Read documentation, set up dev environment; Week 2: Clone and build existing code; Week 3: Fix first bug; Week 4: Submit first patch, present findings |
