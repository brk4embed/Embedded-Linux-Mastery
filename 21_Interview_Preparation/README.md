# 21 — Interview Preparation

> 350+ questions. Your real STAR stories. Company-specific prep. Salary negotiation.  
> This section alone is worth 6 months of a prep course.

---

## Section Structure

```
21_Interview_Preparation/
├── 01_Question_Banks/
│   ├── 01_Kernel_QA_Beginner.md        ← 60 beginner questions
│   ├── 02_Kernel_QA_Intermediate.md    ← 60 intermediate questions
│   ├── 03_Kernel_QA_Advanced.md        ← 60 advanced questions
│   ├── 04_Driver_Writing_QA.md         ← 50 driver-specific questions
│   ├── 05_Boot_Flow_QA.md              ← 30 boot/BSP questions
│   ├── 06_Debugging_Scenarios.md       ← 30 debug scenario questions
│   ├── 07_AI_Embedded_QA.md            ← 30 AI + embedded questions
│   └── 08_Behavioral_QA.md             ← 30 behavioral + system design
├── 02_Your_STAR_Stories.md             ← STAR stories from YOUR real projects
├── 03_System_Design_Interview.md       ← Design a BSP from scratch
├── 04_Coding_Interview.md              ← C coding on whiteboard
├── 05_Company_Specific/
│   ├── 01_Qualcomm.md                  ← Qualcomm-style questions
│   ├── 02_ARM.md                       ← ARM Ltd interview guide
│   ├── 03_Google.md                    ← Google embedded/ChromeOS
│   ├── 04_Amazon.md                    ← Amazon embedded/Graviton
│   ├── 05_NVIDIA.md                    ← NVIDIA embedded AI
│   ├── 06_Intel.md                     ← Intel platform bring-up
│   └── 07_Startups.md                  ← Startup embedded questions
└── 06_Salary_Negotiation.md            ← Market rates + negotiation tactics
```

---

## Your STAR Stories (Personalized)

The most valuable interview asset is a prepared STAR story for each major project.

### Story 1: DW-UFS 4.0 QEMU Device Model

**Question it answers:** "Tell me about a complex technical project you led. What was the hardest part?"

```
SITUATION:
ARM Development Centre needed a QEMU-based developer kit for the DW-UFS 4.0
IP block. The kit would allow software teams to develop and test UFS drivers
before silicon was available — critical for reducing time-to-market.

TASK:
As Module Lead, I was responsible for designing and implementing the complete
QEMU device model: UFSHCI 4.0 register model, UniPro transport emulation,
SCSI command processing, and UPIU state machine.

ACTION:
1. Studied the UFSHCI 4.0 spec + DesignWare UFS IP datasheet
2. Designed a QEMU SysBus device with complete UFSHCI register map
3. Implemented the UTRD (UTP Transfer Request Descriptor) processing loop
4. Built the UPIU state machine (command → data transfer → response)
5. Implemented SCSI command handlers (INQUIRY, READ(10), WRITE(10), etc.)
6. Used QEMU's DMA API to read UPIUs from guest memory
7. Validated against the Linux ufshcd driver — achieved successful SCSI inquiry

RESULT:
- ARM software team could test driver code without waiting for silicon
- Found 3 driver bugs before tapeout (estimated: would have cost weeks in
  post-silicon debug)
- Model fidelity: >95% register compatibility with real UFS 4.0 behavior
- Reduced pre-silicon validation time by an estimated 40%

KEY TECHNICAL INSIGHT:
The hardest part was the UPIU state machine — specifically handling the
timing between doorbell write (host signals command ready), data phase
scheduling (ReadyToTransfer UPIU), and completion interrupt. The QEMU
model had to process these asynchronously while maintaining correct ordering.
```

### Story 2: Coreboot 50-Deep Patch Train (SC7180/SC7280)

**Question it answers:** "Tell me about a time you maintained a large, complex codebase."

```
SITUATION:
Qualcomm Chromebook platforms SC7180 and SC7280 required Coreboot-based
firmware. The Coreboot project is upstream-first — all changes must be
reviewed and merged into the main tree, but our customer schedule required
shipping before upstream review was complete.

TASK:
Maintain a patch series of 50+ Coreboot patches: keep them current with
upstream, resolve merge conflicts as Coreboot evolved, respond to reviewer
feedback, and ensure ChromeOS builds continued to work.

ACTION:
1. Set up a rebasing workflow: track upstream + maintain our patch series
2. Used git rebase -i to clean up individual patches
3. For each reviewer comment on review.coreboot.org, evaluated the feedback
   and responded with code changes or technical justification
4. Coordinated with Google ChromeOS team to identify which patches were
   blocking tapeout validation
5. Prioritized patches by ChromeOS team dependency, not submission date

RESULT:
- 30+ patches merged upstream into Coreboot mainline
- QFPROM eFuse driver also upstreamed to Linux kernel
- SC7180/SC7280 platforms shipped on schedule
- My patches are still in mainline Coreboot tree and active today

WHAT I LEARNED:
Open source contribution is a communication skill as much as a coding skill.
Maintainers review 100s of patches. Clear commit messages, small focused
patches, and patient professional responses to feedback accelerate merging
dramatically.
```

### Story 3: AI Pre-Silicon Validation Platform

**Question it answers:** "Tell me about your experience with AI tools. How have you applied AI in your work?"

```
SITUATION:
Post-silicon Linux driver test cases required significant manual work to
convert to pre-silicon QEMU bare-metal tests. Each conversion took hours.
With 50+ test cases per peripheral and multiple peripherals in DW-UFS 4.0
kit, this was a significant bottleneck.

TASK:
Design and build an AI-powered platform that automates the conversion of
Linux post-silicon test cases to QEMU bare-metal firmware tests.

ACTION:
1. Built RAG pipeline: indexed kernel driver test files with embeddings
2. Designed LLM prompting strategy: "Given this Linux test case, generate
   equivalent bare-metal C test that runs on QEMU without kernel"
3. Built Python orchestration with LangChain/LangGraph agents
4. Created validation layer: compile + run each generated test in QEMU,
   capture output, compare against expected
5. Used Claude claude-3-5-sonnet for code generation,
   local Ollama for non-sensitive code review

RESULT (in progress, current project):
- Proof of concept: 10/10 simple test cases converted correctly
- Complex test cases (multi-step I/O with interrupts): 7/10
- Estimated time savings: 3 hours → 15 minutes per test case
- Platform architecture documented for potential open-source release

TECHNICAL CHALLENGES:
The hardest problem: preserving test intent through translation. A Linux
kernel test uses kernel APIs (skipping hardware details). A bare-metal test
must implement those hardware details. The LLM needed context about register
maps, interrupt handling patterns, and QEMU device behavior — all provided
via RAG.
```

---

## Kernel Interview Questions (Sample — 60 Total in Full Section)

### Beginner Level

**B1:** What is a kernel module? How is it different from built-in kernel code?
```
Answer: A kernel module (.ko) is a piece of code that can be loaded and
unloaded from the kernel at runtime, without rebooting. Built-in code is
compiled directly into the kernel image (vmlinux).

Key differences:
- Module: loaded with insmod/modprobe, unloaded with rmmod
- Built-in: loaded at boot, cannot be unloaded
- Modules save boot time and memory (only loaded when needed)
- Built-in: cannot be unloaded → safer for critical subsystems

When to use built-in:
- Core drivers needed before rootfs is mounted (storage drivers)
- Security-critical code
- Performance-critical (modules have slight overhead per call)
```

**B2:** What is the difference between `printk` and `pr_err`?
```
Answer: pr_err() is a thin wrapper around printk() at KERN_ERR log level.
pr_err("message\n") expands to printk(KERN_ERR "message\n").

Prefer:
- pr_err/warn/info/debug for code without a struct device context
- dev_err/warn/info/dbg when you have a 'struct device *dev' — it
  automatically prefixes output with the device name, which is essential
  for multi-instance drivers.

Note: pr_debug() is compiled out at non-DEBUG builds unless
CONFIG_DYNAMIC_DEBUG is enabled.
```

### Intermediate Level

**I1:** Explain the `container_of` macro. How does it work and when do you use it?
```c
/*
 * container_of — given a pointer to a member of a struct,
 * find the pointer to the containing struct.
 *
 * ptr:    pointer to the member
 * type:   type of the containing struct  
 * member: name of the member in the struct
 *
 * How it works:
 *   1. offsetof(type, member) = byte offset of member from struct start
 *   2. (char*)ptr - offset = address of the containing struct
 *   3. Cast to type*
 *
 * Example: get driver private data from a work_struct pointer:
 */

struct my_priv {
    struct work_struct work;    // embedded at some offset
    int data;
    struct device *dev;
};

static void my_work_fn(struct work_struct *work)
{
    // work is a pointer to the work_struct member
    // we need pointer to the containing my_priv struct
    struct my_priv *priv = container_of(work, struct my_priv, work);
    // Now we can access priv->data, priv->dev, etc.
    dev_info(priv->dev, "data = %d\n", priv->data);
}

/*
 * This pattern is ubiquitous in Linux:
 * - Workqueues
 * - Timer callbacks
 * - List traversal (list_entry = container_of)
 * - notifier callbacks
 * - Wait queue callbacks
 */
```

**I2:** What is the difference between `kzalloc` and `vmalloc`? When do you use each?
```
kzalloc(size, GFP_KERNEL):
  - Returns physically contiguous memory
  - Zero-initialized ('z' = zero)
  - Limited: max ~4MB (order-10 pages)
  - Fast: O(1) from slab cache for common sizes
  - Required for DMA buffers (device needs physical contiguity)
  - Use for: per-device structs, small buffers, DMA (with dma_alloc_coherent)

vmalloc(size):
  - Returns virtually contiguous memory (physically scattered)
  - Not zero-initialized (use vzalloc for zeroed)
  - Large: limited only by virtual address space
  - Slower: TLB entries for each physical page
  - Cannot be used for DMA
  - Use for: large buffers, software structures, firmware loading

Rule of thumb:
  size < 128KB → kzalloc (fast, simple)
  size > 128KB and not DMA → vmalloc (no contiguity constraint)
  DMA buffer → dma_alloc_coherent (hardware requirement)
```

### Advanced Level

**A1:** A driver works on your development board but causes a NULL pointer dereference on another engineer's board (same SoC, different board revision). How would you debug this?

```
Step-by-step approach:

1. Get the oops output:
   - Exact address, call trace, registers
   - addr2line -e vmlinux -f <faulting address>

2. Look at the code path:
   - The NULL dereference is often early in probe()
   - What pointer is NULL? (look at which line from addr2line)

3. Compare the DTBs:
   - diff <(fdtdump board_a.dtb | sort) <(fdtdump board_b.dtb | sort)
   - Missing DT property? Wrong value? Missing clock or regulator?

4. Common causes:
   a. Optional resource that board A has but board B doesn't:
      clk = devm_clk_get(dev, "optional-clock");
      // if clk is ERR_PTR, and code later dereferences without IS_ERR check
   
   b. DT node present but different compatible → different probe path
   
   c. Board B has different hardware revision → different PMIC/GPIO layout

5. Fix:
   - Add IS_ERR() check after every resource acquisition
   - Use devm_clk_get_optional() for truly optional clocks
   - Add dev_info() to log all resource acquisition results

Prevention:
   - Always test with KASAN enabled
   - Test with sparse: make C=2
   - Enable CONFIG_DEBUG_NULL_PTRS (ARM64 memory tagging)
```

**A2:** What is KASAN and how do you use it to find bugs?
```
KASAN = Kernel Address SANitizer
Detects:
  - Use-after-free
  - Out-of-bounds memory accesses  
  - Stack-based overflows
  - Global variable overflows
  - Use-before-initialization

Enable:
  CONFIG_KASAN=y
  CONFIG_KASAN_INLINE=y  (or OUTLINE for smaller kernel)
  CONFIG_KASAN_GENERIC=y (or KASAN_SW_TAGS for ARM64)
  CONFIG_SLUB=y (or SLAB)

When a bug is triggered:
  ==================================================================
  BUG: KASAN: use-after-free in ufshcd_clear_cmd+0x88/0x100
  Read of size 4 at addr ffff888012345678 by task kworker/0:1
  ...
  Allocated by task 12345:
    kmalloc called from probe
  Freed by task 12345:
    kfree called from remove

Usage in your workflow:
  - Always run with KASAN for QEMU testing
  - KASAN has ~2× overhead — don't use in production
  - Run your driver's full test suite with KASAN
  - KASAN reports are very actionable — they show allocation + free points
```

---

## C Coding Interview Prep

Common kernel-style C problems:

```c
/* Problem 1: Implement a safe list traversal that handles concurrent deletion */
/* Answer: Use list_for_each_entry_safe */

struct work_item {
    struct list_head list;
    int id;
};

void drain_list(struct list_head *head, spinlock_t *lock)
{
    struct work_item *item, *tmp;
    LIST_HEAD(to_free);

    spin_lock(lock);
    list_splice_init(head, &to_free);  /* atomically move entire list */
    spin_unlock(lock);

    /* Free outside of lock — no blocking allocation with lock held */
    list_for_each_entry_safe(item, tmp, &to_free, list) {
        list_del(&item->list);
        kfree(item);
    }
}

/* Problem 2: Atomic bitfield operations without data races */
#include <linux/bitops.h>

unsigned long flags = 0;
set_bit(0, &flags);             /* atomic set bit 0 */
clear_bit(1, &flags);           /* atomic clear bit 1 */
test_and_set_bit(2, &flags);    /* returns old value, sets bit */
test_bit(0, &flags);            /* read bit (not necessarily atomic) */

/* Problem 3: IOMEM access — common mistake */
/* WRONG: */
uint32_t *ptr = (uint32_t *)0xfe650000;
*ptr = 0x01;  /* undefined behavior — no memory barrier, optimizer may remove */

/* CORRECT: */
void __iomem *base = ioremap(0xfe650000, 0x1000);
writel(0x01, base);      /* writes with memory barrier */
uint32_t val = readl(base);  /* reads with memory barrier */
iounmap(base);
```

---

## Salary Negotiation

### Market Rates (India, 2025)

| Role | Experience | Bangalore CTC |
|------|-----------|---------------|
| Embedded Linux Eng | 5–7yr | ₹18–30 LPA |
| Senior Embedded Linux | 8–12yr | ₹30–55 LPA |
| Principal/Staff Eng | 12yr+ | ₹55–90 LPA |
| AI+Embedded specialist | 5yr+ | ₹25–50 LPA (premium for AI) |
| Consulting (freelance) | 8yr+ | ₹5,000–15,000/hr |

**Your positioning at 11 years with QEMU device modeling + AI:**  
₹35–55 LPA for senior roles, potentially higher given AI specialization.

### Key Negotiation Points

1. **The QEMU device modeling premium:** Very few engineers worldwide have built a complete UFS 4.0 QEMU model. Name it, quantify it ("reduced pre-silicon validation time by 40%").

2. **AI skills command a premium:** Embedded AI is in short supply. Your pre-silicon validation platform puts you 2–3 years ahead of most embedded engineers.

3. **Upstream contributions:** QFPROM in kernel tree + Coreboot patches = verifiable quality signal. Most candidates can't show this.

4. **Never negotiate against yourself:** If asked for expected CTC, give a range with floor at current + 40%. Let them make the first move if possible.

5. **Total compensation:** Base + variable + stock + joining bonus + notice period buyout. Evaluate all components.

---

*Related: [00_Career_Roadmap/](../00_Career_Roadmap/) | [24_Industry_Project_Portfolio/](../24_Industry_Project_Portfolio/) | [20_Freelancing/](../20_Freelancing/)*
