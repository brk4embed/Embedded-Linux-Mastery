# Online Source Code Browsing — U-Boot, Linux Kernel, Coreboot

> **Goal:** Browse 30 million lines of kernel source without downloading it. Find any function, any commit, any driver — in seconds, from your browser.

---

## The Best Online Tools (Ranked by Usefulness for Embedded Work)

| Tool | URL | Best For | Speed |
|------|-----|---------|-------|
| **Elixir Bootlin** | elixir.bootlin.com | Linux, U-Boot, Barebox, GCC | ⭐⭐⭐⭐⭐ |
| **Bootlin Sourcegraph** | sourcegraph.com/github.com/torvalds/linux | Full text search | ⭐⭐⭐⭐ |
| **GitHub** | github.com/torvalds/linux | Git history, blame, PR search | ⭐⭐⭐⭐ |
| **grep.app** | grep.app | Fast regex across all GitHub repos | ⭐⭐⭐⭐ |
| **Bootlin Compiler** | godbolt.org | See ARM64 assembly for your C code | ⭐⭐⭐⭐ |
| **lore.kernel.org** | lore.kernel.org | Kernel mailing list archives | ⭐⭐⭐⭐⭐ |
| **OpenGrok** | Various | Company-internal code hosting | ⭐⭐⭐ |
| **cs.android.com** | cs.android.com | Android kernel + AOSP | ⭐⭐⭐ |

---

## 1. Elixir Bootlin — The Essential Tool

**URL:** https://elixir.bootlin.com

This is the **single most useful tool** for embedded Linux engineers. It provides:
- Cross-references for the ENTIRE Linux kernel, U-Boot, Barebox, LLVM, GCC
- Symbol definitions with one click
- All references (callers, struct users, DT compatible users)
- Every kernel version from 2.6.11 to latest

### Mastering Elixir: The Full Guide

#### Search Types

```
The search box has a dropdown. Use the right type:

"Identifier"  → Exact symbol name (function, variable, struct, macro)
              Example: ufshcd_probe_hba
              Result: Shows definition + all references across codebase

"Free text"   → Full text search in source
              Example: "QFPROM eFuse"
              Result: All lines containing this text

"File"        → Find file by name
              Example: qfprom.c
              Result: All files named qfprom.c across all kernel versions
```

#### Navigating a Symbol

```
1. Go to elixir.bootlin.com/linux/latest/source
2. Select version: "v6.9" from dropdown
3. Type in search: "ufshcd_probe_hba" → Select "Identifier" → Search

Result page shows:
  ┌─ DEFINED ─────────────────────────────────────────────┐
  │ drivers/ufs/core/ufshcd.c, line 7234                  │
  └────────────────────────────────────────────────────────┘
  ┌─ REFERENCED BY ────────────────────────────────────────┐
  │ drivers/ufs/core/ufshcd.c, line 7189 (called from)    │
  │ drivers/ufs/core/ufshcd.c, line 8012 (called from)    │
  └────────────────────────────────────────────────────────┘

Click any reference → goes directly to that file and line.
```

#### Finding DT Compatible Strings (Essential for Driver Work)

```
Search: qcom,qfprom     (Type: Identifier)

Results show:
  Defined in:   arch/arm64/boot/dts/qcom/sc7180.dtsi
  Used in:      drivers/nvmem/qfprom.c  (of_match_table)
                Documentation/devicetree/bindings/...

→ Instantly see which DT binding your driver uses
→ Find all boards that have this compatible string
```

#### Cross-Version Comparison Workflow

```
Scenario: "When was ufshcd_abort() changed between 5.15 and 6.1?"

1. Open two browser tabs:
   Tab 1: elixir.bootlin.com/linux/v5.15/source/drivers/ufs/core/ufshcd.c
   Tab 2: elixir.bootlin.com/linux/v6.1/source/drivers/ufs/core/ufshcd.c

2. Use browser search (Ctrl+F) to find "ufshcd_abort" in each tab

3. Compare the function signatures/logic side by side

→ Faster than git diff between releases for spot-checking
```

### U-Boot on Elixir

```
URL: https://elixir.bootlin.com/u-boot/latest/source

Same navigation works for U-Boot:
- Find board init: search "board_init" (Identifier)
- Find ENV commands: search "U_BOOT_CMD" (Identifier)
- Find your platform: browse arch/arm/mach-snapdragon/
- Find DT parsing: search "ofnode_get_property" (Identifier)
```

---

## 2. Sourcegraph — Full-Text Code Intelligence

**URL:** https://sourcegraph.com/github.com/torvalds/linux

Better than GitHub for:
- Go-to-definition across files (like an IDE)
- Find all references globally
- Search with complex queries

```
Search syntax examples:

repo:torvalds/linux ufshcd_abort lang:c
→ Find ufshcd_abort in C files only

repo:torvalds/linux file:qfprom.c select_rows
→ Search in a specific file

repo:torvalds/linux "qcom,sc7180-qfprom" lang:c
→ Find all C files using this DT compatible

repo:torvalds/linux type:symbol ufshcd_host_params
→ Find symbol definitions specifically

repo:torvalds/linux -file:Documentation/ qfprom
→ Exclude Documentation directory from search
```

---

## 3. grep.app — Fastest Multi-Repo Search

**URL:** https://grep.app

Searches ALL public GitHub repositories simultaneously. Extremely fast.

```
Use cases:
  "ufshci_4_0_intr_status"    → Find who uses this register across all drivers
  "QCOM_SCM_SVC_FUSE"         → Find all callers of this Qualcomm SMC service
  "depthcharge"               → Find all Coreboot/Chromebook references
  lang:c "ufs_hba_variant_ops" → C-only search for UFS variant ops

Great for:
  - "Has anyone else solved this problem in another driver?"
  - "What's the standard pattern for X in the kernel?"
  - Finding example implementations before writing your own
```

---

## 4. GitHub — Git History and Blame

**URL:** https://github.com/torvalds/linux

GitHub is essential for understanding **why** code exists, not just what it does.

### Finding the Commit That Introduced a Bug

```
Method 1: GitHub Blame
  1. Navigate to: github.com/torvalds/linux/blob/master/drivers/nvmem/qfprom.c
  2. Click "Blame" button (top right)
  3. See every line annotated with its commit hash and author
  4. Click any commit hash → see the full change and commit message

Method 2: GitHub Search by commit message
  URL: github.com/torvalds/linux/commits/master/drivers/nvmem/qfprom.c
  → Shows all commits affecting qfprom.c with their messages
  → Click any commit → see full diff

Method 3: GitHub Search across commit history
  In search box: "qfprom" + select "Commits" filter
  → Find all commits mentioning qfprom
```

### Searching Issues / PRs for Known Bugs

```
github.com/torvalds/linux/issues?q=ufshcd+suspend+resume
→ Find if someone already reported and fixed your bug

Note: Linux kernel uses email (lore.kernel.org), not GitHub issues.
      But: U-Boot, Coreboot, Yocto use GitHub issues heavily.
```

---

## 5. lore.kernel.org — Kernel Mailing List Search

**URL:** https://lore.kernel.org

This is where ALL Linux kernel patch review, bug reports, and discussions happen. Your QFPROM patches are here.

```
Search your own patches:
  https://lore.kernel.org/lkml/?q=Ravi+Kumar+Bokka
  → Shows all your upstream contributions

Search for patches about a topic:
  https://lore.kernel.org/linux-nvmem/?q=qfprom
  → All patches touching NVMEM/QFPROM subsystem

Search for fix for a specific issue:
  https://lore.kernel.org/?q=ufshcd+abort+null+pointer
  → Find if someone already sent a fix

Follow a subsystem:
  https://lore.kernel.org/linux-nvmem/  → NVMEM subsystem
  https://lore.kernel.org/linux-ufs/    → UFS subsystem
  https://lore.kernel.org/linux-arm-msm/ → Qualcomm ARM platform
```

---

## 6. Compiler Explorer (godbolt.org) — See ARM64 Assembly

**URL:** https://godbolt.org

This tool lets you see the **ARM64 assembly output** for any C code you write. Essential for:
- Understanding how the compiler handles `volatile`, `barrier()`, `READ_ONCE`
- Verifying your MMIO access code is correct
- Learning about instruction selection

```
1. Go to godbolt.org
2. Select language: C
3. Select compiler: arm64-linux-gnu gcc 12.2  (or aarch64-linux-gnu)
4. Add flags: -O2 -march=armv8-a

Paste your code:
#include <stdint.h>

static inline uint32_t readl(volatile uint32_t *addr) {
    uint32_t val;
    asm volatile("ldr %w0, [%1]" : "=r"(val) : "r"(addr) : "memory");
    return val;
}

void my_driver_init(volatile uint32_t *base) {
    uint32_t val = readl(base + 0x10);
    val |= (1 << 3);
    *(volatile uint32_t *)(base + 0x10) = val;
}

→ See exact ARM64 instructions: ldr, orr, str
→ Verify barriers are present
→ Check alignment and access size
```

---

## 7. Coreboot Source Browsing

**URL:** https://review.coreboot.org/plugins/gitiles/coreboot/+/refs/heads/main

```
Browse Coreboot source (your 50-patch area):
  → src/soc/qualcomm/sc7180/
  → src/soc/qualcomm/sc7280/
  → src/mainboard/google/ (Chromebook boards)

Your patches history:
  https://review.coreboot.org/q/ravi+kumar+bokka

Search in Coreboot source:
  Use GitHub mirror: github.com/coreboot/coreboot
  Or: grep.app with "repo:coreboot/coreboot"
```

---

## 8. Bookmarks List — Embedded Engineer's Browser Toolbar

Copy these into your browser bookmarks toolbar:

```
Folder: Kernel Nav
  Linux Elixir     → https://elixir.bootlin.com/linux/latest/source
  U-Boot Elixir    → https://elixir.bootlin.com/u-boot/latest/source
  Kernel GitHub    → https://github.com/torvalds/linux
  Kernel Lore      → https://lore.kernel.org
  Linux NVMEM      → https://lore.kernel.org/linux-nvmem/
  Linux ARM-MSM    → https://lore.kernel.org/linux-arm-msm/
  UFS Subsystem    → https://lore.kernel.org/linux-scsi/

Folder: Search Tools
  grep.app         → https://grep.app
  Sourcegraph      → https://sourcegraph.com/github.com/torvalds/linux
  Compiler Expl.   → https://godbolt.org

Folder: Your Work
  Your Coreboot    → https://review.coreboot.org/q/ravi+kumar+bokka
  Your LKML        → https://lore.kernel.org/lkml/?q=Ravi+Kumar+Bokka
  Your Chromium    → https://chromium-review.googlesource.com/q/ravi+kumar+bokka
  Your GitHub      → https://github.com/brk4embed/Embedded-Linux-Mastery
```

---

## 9. Practical Workflow: Debug a UFS Issue Without Local Source

```
Scenario: You see this in dmesg:
  ufshcd 4804000.ufs: ufshcd_abort: tag 0 task not found

Step 1: Find the function on Elixir
  → elixir.bootlin.com → search "ufshcd_abort" → Identifier
  → See definition + all callers

Step 2: Read the function logic on Elixir
  → Click on the definition link
  → Read without downloading anything

Step 3: Check git history on GitHub
  → github.com/torvalds/linux/blame/master/drivers/ufs/core/ufshcd.c
  → Find who last modified ufshcd_abort + why

Step 4: Search for known fixes on lore
  → lore.kernel.org/?q=ufshcd_abort+tag
  → Find if someone already reported and patched this

Step 5: Cross-reference with grep.app
  → grep.app: "ufshcd_abort" lang:c
  → See how other UFS drivers handle abort

Total time: 10 minutes without touching your local machine
```
