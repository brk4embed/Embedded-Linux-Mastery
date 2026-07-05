# Open Source Contribution Roadmap — Linux Kernel, U-Boot, Coreboot, OP-TEE, QEMU

> **The Most Important Career Investment You Can Make:**  
> One accepted patch to the Linux kernel mainline is worth more on your CV than  
> 10 years of closed-source integration work. This guide takes you from first patch to maintainer.

---

## Why Open Source Contributions?

```
Signal to the world:
  "My code is good enough that Linus Torvalds (or Greg KH, or Simon Glass) reviewed it,
   and they accepted it into the official tree."

Career impact:
  - Job offers from top companies (Google, Meta, Red Hat, Linaro, Arm, Qualcomm)
  - Consulting credibility: "I maintain the UFS driver in mainline Linux"
  - Freelancing premium: 2-3× higher rates for known contributors
  - Visa support for USA/EU: O-1A / EU Blue Card criteria

Ravi's existing assets:
  - QFPROM driver already upstreamed ✓ (huge!)
  - DW-UFS 4.0 QEMU model — pending upstream?
  - Coreboot SC7180/SC7280 work — potential upstream
```

---

## Part 1: Linux Kernel Contributions

### The Kernel Patch Workflow

```
Step 1: Find something to fix/add
Step 2: Make the change
Step 3: Test thoroughly
Step 4: Write a proper commit message
Step 5: Send patch to the right mailing list
Step 6: Respond to review feedback
Step 7: Resubmit until accepted
Step 8: Watch for merge window
```

### Finding Your First Contribution

```bash
# Start here: checkpatch warnings
cd ~/linux
./scripts/checkpatch.pl --strict -f drivers/ufs/core/ufshcd.c
# Will show style issues, potential bugs

# Look for TODO/FIXME comments in your area
grep -rn "FIXME\|TODO\|HACK" drivers/ufs/ | head -20

# Check kernel janitors (easy first bugs)
# https://kernelnewbies.org/KernelJanitors

# Easy categories to start:
# 1. Fix sparse warnings (incorrect __user, __iomem annotations)
# 2. Fix smatch warnings (NULL dereference, unchecked return)
# 3. Convert deprecated APIs (use devm_, use gpiod_get instead of gpio_request)
# 4. Fix checkpatch warnings (style)
# 5. Document existing functions (add kerneldoc)
```

### The Perfect Commit Message

```
# Structure:
subsystem: component: brief description (< 72 chars)

Problem this patch solves (1-3 sentences, no solution yet).

The solution: what changed and why (3-5 sentences).
Explain the approach, not just what lines changed.

Test results:
  - Tested on: [hardware or QEMU setup]
  - Boot tested: yes/no
  - Regression tested: [test suite used]

Fixes: <12-char sha1> ("title of the commit being fixed")
Cc: stable@vger.kernel.org   # if this should go to stable branches
Signed-off-by: Your Name <your@email.com>
```

**Real example:**

```
ufs: core: Fix potential NULL dereference in ufshcd_abort

When ufshcd_abort() is called during error recovery, the lrbp
(Local Reference Block Pointer) may already be NULL if the transfer
was completed concurrently. This leads to a NULL dereference at
lrbp->cmd.

Add a NULL check before accessing lrbp->cmd to prevent the crash.
The race window exists because ufshcd_transfer_req_compl() sets
lrbp->cmd = NULL when completing a transfer, which can happen
before ufshcd_abort() reads it.

Tested on Qualcomm SC7180 with UFS 3.1 device. Triggered by running
'fio --filename=/dev/sda --ioengine=libaio --rw=randrw' under memory
pressure to increase IRQ latency.

Fixes: 7fabb77385b9 ("ufs: core: abort all pending requests when error")
Cc: stable@vger.kernel.org
Signed-off-by: Ravi Kumar Bokka <brk4embed@gmail.com>
```

### Send Patch to Mailing List

```bash
# Configure git-send-email
git config --global sendemail.smtpserver smtp.gmail.com
git config --global sendemail.smtpserverport 587
git config --global sendemail.smtpencryption tls
git config --global sendemail.smtpuser youremail@gmail.com

# Find the right mailing list and maintainer
cd ~/linux
./scripts/get_maintainer.pl drivers/ufs/core/ufshcd.c
# Bart Van Assche <bvanassche@acm.org> (maintainer)
# Martin K. Petersen <martin.petersen@oracle.com> (reviewer)
# linux-scsi@vger.kernel.org (mailing list)
# linux-kernel@vger.kernel.org (always CC this)

# Create patch
git format-patch -1 --subject-prefix="PATCH"
# 0001-ufs-core-Fix-potential-NULL-dereference-in-ufshcd_abort.patch

# Run checkpatch
./scripts/checkpatch.pl 0001-*.patch
# Must be clean before sending

# Send
git send-email \
    --to bvanassche@acm.org \
    --to martin.petersen@oracle.com \
    --cc linux-scsi@vger.kernel.org \
    --cc linux-kernel@vger.kernel.org \
    0001-ufs-core-Fix-potential-NULL-dereference-in-ufshcd_abort.patch
```

### Responding to Review Feedback

```bash
# After review comments arrive:

# If you need to change the patch:
git commit --amend   # edit the commit
# OR for multiple commits:
git rebase -i HEAD~3

# Resend with version number
git format-patch -1 --subject-prefix="PATCH v2"
# Add a version summary at the start of the patch body:
# v2:
#   - Fixed locking as suggested by Bart
#   - Added Reviewed-by tag from Martin

# Track your patch
# https://patchwork.kernel.org/project/linux-scsi/list/
# Search for your name or patch subject
```

### Ravi's Linux Kernel Contribution Targets

Based on your expertise:

| Priority | Area | What to Submit | Effort |
|----------|------|---------------|--------|
| **Now** | UFS (your expertise!) | Bug fixes, cleanup, comments | 1-3 days |
| **Now** | Coreboot RK3588 | Enable Radxa Rock 5B+ in Coreboot | 1 week |
| **3 months** | QEMU UFS model | Upstream your DW-UFS 4.0 QEMU device | 2-4 weeks |
| **6 months** | RK3588 pinctrl | New pin configurations for Rock 5B+ | 1 week |
| **1 year** | UFS subsystem co-maintainer | After multiple accepted patches | Long-term |

---

## Part 2: U-Boot Contributions

### U-Boot Workflow (similar to kernel but different list)

```bash
# Mailing list: u-boot@lists.denx.de
# Web: https://patchwork.ozlabs.org/project/uboot/list/

# Find maintainer
./scripts/get_maintainer.pl board/rockchip/
# Kever Yang <kever.yang@rock-chips.com>
# Simon Glass <sjg@chromium.org>

# Clone U-Boot
git clone https://source.denx.de/u-boot/u-boot.git
cd u-boot

# Easy U-Boot contributions:
# 1. Fix build warnings for your platform
# 2. Update defconfig files
# 3. Add SPDX license headers
# 4. Fix DM (Driver Model) deprecation warnings
# 5. Add board documentation

# Build your board
make rock5b_defconfig
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# Find issues
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
    make KCFLAGS="-W -Wextra" 2>&1 | grep "warning:" | head -20
```

### Specific U-Boot Contributions for Radxa Rock 5B+

```bash
# Add Rock 5B+ to U-Boot (if not fully supported):

# 1. Create board config
cp configs/rock5b_defconfig configs/rock5b_plus_defconfig
# Edit for Rock 5B+ specifics

# 2. Add board files
mkdir board/radxa/rock5b_plus/
# rock5b_plus.c: board_init, board_late_init, dram config

# 3. Add device tree
# arch/arm/dts/rk3588-rock-5b-plus.dts

# 4. Test
# qemu-system-aarch64 -M virt -bios u-boot.bin ...

# 5. Send patch series (multiple related patches)
git format-patch origin/master --subject-prefix="PATCH" -o patches/
git send-email patches/*.patch --to u-boot@lists.denx.de
```

---

## Part 3: Coreboot Contributions

### Coreboot Workflow (uses Gerrit)

```bash
# Setup Gerrit account at https://review.coreboot.org
# Configure git hooks for Gerrit
git clone https://review.coreboot.org/coreboot.git
cd coreboot
./util/gitconfig/gitconfig.sh   # sets up git hooks for Change-Id

# Configure your identity
git config user.email "brk4embed@gmail.com"
git config user.name "Ravi Kumar Bokka"

# Ravi's focus: SC7180/SC7280 + RK3588 Coreboot support
ls src/mainboard/
# google/      # most Chromebooks use Google mainboards
# radxa/       # may not exist yet — opportunity!
# rockchip/    # SoC-level support

# Find SC7180-related files
find . -name "*.c" -o -name "*.h" | xargs grep -l "sc7180" 2>/dev/null | head -10
```

### Adding a New Mainboard to Coreboot

```bash
# Coreboot mainboard structure:
src/mainboard/radxa/rock5b/
├── Kconfig          # build configuration
├── Makefile.inc     # build rules
├── board.c          # board-specific init
├── devicetree.cb    # device tree (Coreboot format)
└── romstage.c       # early init (before RAM)

# devicetree.cb example for Rock 5B+:
chip soc/rockchip/rk3588
    device domain 0 on
        device ref uart2 on end    # debug UART
        device ref i2c0 on         # PMIC
            chip drivers/i2c/rk808
                register "bus"  = "0"
                device i2c 0x20 on end
            end
        end
        device ref emmc on end
        device ref sd on end
        device ref usb3_0 on end
    end
end

# Submit to Gerrit
git add -A
git commit   # Gerrit requires a Change-Id (added by git hook)
git push origin HEAD:refs/for/master   # submit for review
```

---

## Part 4: QEMU Contributions

### Upstream Your DW-UFS 4.0 QEMU Model

This is Ravi's highest-value open source contribution opportunity.

```bash
# QEMU mailing list: qemu-devel@nongnu.org
# Subsystem: qemu-block@nongnu.org (storage)
# Maintainer: Markus Armbruster <armbru@redhat.com>

# Your DW-UFS 4.0 model structure:
hw/ufs/
├── ufs.c          # main device model
├── ufs.h          # data structures
├── ufs-lu.c       # logical unit emulation
└── meson.build    # build integration

# Check existing UFS emulation in QEMU
ls hw/ufs/    # may already exist partially

# Requirements for upstream:
# 1. Follow QEMU coding style (hw/ufs/ufs.c already does this?)
# 2. Add tests in tests/unit/ or tests/qtest/
# 3. Add documentation in docs/
# 4. Migration support (for live migration)
# 5. QAPI schema for -device ufs options

# Migration support (required for upstream):
static const VMStateDescription vmstate_ufs = {
    .name = "ufs",
    .version_id = 1,
    .minimum_version_id = 1,
    .fields = (VMStateField[]) {
        VMSTATE_UINT32(regs.hcs, UfsHc),
        VMSTATE_UINT32(regs.is, UfsHc),
        /* ... all state that must survive live migration ... */
        VMSTATE_END_OF_LIST()
    }
};
```

---

## Part 5: OP-TEE Contributions

```bash
# OP-TEE GitHub: https://github.com/OP-TEE/optee_os
# Mailing list: op-tee@lists.trustedfirmware.org

# Add RK3588 platform support to OP-TEE OS
# See existing Rockchip support:
ls core/arch/arm/plat-rockchip/

# Key files to add:
core/arch/arm/plat-rockchip/
├── rk3588_NOTICE       # copyright + license
├── main.c              # platform init
├── conf.mk             # build config for RK3588
└── platform_config.h  # memory map, uart, etc.

# Build OP-TEE for RK3588
make PLATFORM=rockchip-rk3588 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     CFG_ARM64_core=y \
     all
```

---

## Part 6: The Open Source Reputation Building Timeline

### Month 1-3: Learn the Process

```
Week 1: Set up development environment
  ✓ Subscribe to linux-kernel, linux-scsi, linux-rockchip mailing lists
  ✓ Register on lore.kernel.org, patchwork.kernel.org
  ✓ Set up git send-email with your email
  ✓ Read: Documentation/process/submitting-patches.rst

Week 2: Study existing patches in your area
  ✓ Browse linux-scsi mailing list archives
  ✓ Find patches by Bart Van Assche (UFS maintainer) — study their style
  ✓ Read 10 accepted patches in drivers/ufs/

Week 3-4: Send first patch
  ✓ Fix one checkpatch warning in drivers/ufs/
  ✓ Send as [PATCH] with [RFC] if unsure
  ✓ Respond promptly to any feedback
```

### Month 4-6: Build Momentum

```
Goal: 3-5 accepted patches in Linux kernel
  Focus: drivers/ufs/ (your expertise)
  Types: bug fixes, cleanups, documentation
  
Goal: 1-2 accepted patches in QEMU
  Focus: hw/ufs/ (your DW-UFS model)
  
Result: Your name in the kernel changelog
  git log --oneline drivers/ufs/ | grep "Ravi"
```

### Month 7-12: First Major Contribution

```
Goal: Add Rock 5B+ to U-Boot or Coreboot
  This is a multi-patch series (10-20 patches)
  Requires coordination with SoC maintainers
  
Result: 
  "Ravi Kumar Bokka added Radxa Rock 5B+ support to U-Boot 2024.10"
  This is on your LinkedIn, your website, everywhere.
```

### Year 2+: Maintainer Path

```
After 20+ accepted patches in one subsystem:
  Request to be added as reviewer/co-maintainer
  
Email to maintainer:
  "Hi [Name], I've been contributing to [subsystem] for about 18 months.
   I've had [N] patches accepted. I'd like to help review patches to
   reduce your load. Would you consider adding me as a reviewer?"
  
Maintainers always need help reviewing. Being asked to review is the
first step to becoming a maintainer.
```

---

## Checklist: Before Sending Any Patch

```bash
# Run all checks:

# 1. Checkpatch
./scripts/checkpatch.pl --strict 0001-*.patch
# Must have NO errors. Warnings: fix as many as possible.

# 2. Sparse (static analysis)
make C=1 drivers/ufs/core/ufshcd.o

# 3. Build with all warnings
make KCFLAGS="-W -Wextra" drivers/ufs/

# 4. Boot test (at minimum)
# Test on real hardware OR QEMU

# 5. Commit message check
git log --oneline -1
# Correct format: "subsystem: description"

# 6. Signed-off-by
git log -1 | grep "Signed-off-by"
# Must be present with your real name and email

# 7. No trailing whitespace
grep -n " $" your_file.c   # must be empty

# 8. SPDX header in new files
head -3 new_file.c
# // SPDX-License-Identifier: GPL-2.0
```

---

## Interview Q&A: Open Source

| Level | Question | Answer |
|-------|----------|--------|
| **Beginner** | What is a signed-off-by? | Developer Certificate of Origin — certifies you have right to contribute |
| **Beginner** | What is Patchwork? | Web tool to track patch status on mailing lists |
| **Intermediate** | What is the difference between `git send-email` and GitHub PR? | Kernel uses email; companies may use GitHub PRs; Coreboot uses Gerrit |
| **Intermediate** | How many patches before you can call yourself a kernel contributor? | Even 1 accepted patch makes you a contributor; 10+ shows commitment |
| **Advanced** | What is the merge window? How does it affect patch submission? | 2-week window after each release where new features merge; bug fixes any time |
| **Expert** | How do you become a subsystem maintainer? | Multiple contributions, invitation from current maintainer, MAINTAINERS file entry |
