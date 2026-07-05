# 12 — Open Source Contribution

> Contributing to Linux, Coreboot, and ChromeOS upstream is the fastest way to become a recognized embedded Linux engineer. This section documents the exact workflow.

---

## Section Structure

```
12_Open_Source/
├── 01_Why_Contribute.md              ← Career impact, visibility, credibility
├── 02_Linux_Kernel_Workflow.md       ← Mailing list, get_maintainer.pl, patch series
├── 03_Coreboot_Gerrit_Workflow.md    ← Gerrit review, CQ+2, ChromeOS integration
├── 04_First_Patch_Guide.md           ← Step-by-step: your first upstream patch
├── 05_Patch_Writing_Standards.md     ← commit message, cover letter, v2 revisions
├── 06_Code_Review_Etiquette.md       ← Responding to maintainer feedback
├── 07_Becoming_A_Maintainer.md       ← MAINTAINERS file, subsystem ownership
└── 08_Your_Contribution_Portfolio.md ← QFPROM driver, Coreboot SC7180/SC7280 docs
```

---

## Your Upstream Contributions (Case Studies)

### 1. QFPROM eFuse Linux Driver
- **File:** `drivers/nvmem/qfprom.c`
- **Subsystem:** NVMEM
- **Maintainer:** Srinivas Kandagatla
- **Mailing list:** linux-arm-msm@vger.kernel.org + linux-kernel@vger.kernel.org

### 2. Coreboot SC7180/SC7280 (50-patch train)
- **Repository:** review.coreboot.org (Gerrit)
- **Patches:** SoC init, DDR, UART, QSPI, GPIO, PMIC, Depthcharge integration
- **Upstream status:** Merged to coreboot master

---

## Linux Kernel Patch Workflow

```bash
# 1. Find maintainers for your file
scripts/get_maintainer.pl drivers/nvmem/qfprom.c

# 2. Create patch
git format-patch -1 --cover-letter -o patches/

# 3. Check patch quality
scripts/checkpatch.pl patches/*.patch

# 4. Send patch
git send-email \
  --to=linux-nvmem@lists.linux.dev \
  --to=linux-arm-msm@vger.kernel.org \
  --cc=linux-kernel@vger.kernel.org \
  patches/*.patch

# 5. Monitor lore.kernel.org for review feedback
# 6. Address feedback, re-send as v2:
git format-patch -1 -v2 -o patches/
git send-email patches/v2-*.patch
```

---

## Perfect Commit Message Formula

```
subsystem: component: Short description (max 72 chars)

Problem: [What was wrong / what was missing]

Solution: [What you did and why]

[Optional: Before/after comparison, benchmark numbers]

Signed-off-by: Your Name <your@email.com>
Reviewed-by: Reviewer Name <reviewer@email.com>  [after review]
```

**Your real example (QFPROM):**
```
nvmem: qfprom: Add Qualcomm SC7180 and SC7280 support

Add QFPROM eFuse nvmem driver support for Qualcomm SC7180 (Snapdragon 7c)
and SC7280 (Snapdragon 7c Gen 2) SoCs.

The QFPROM controller on these SoCs provides read access to eFuse cells
for serial number, calibration data, and secure boot fuses.

Signed-off-by: Ravi Kumar Bokka <brk4embed@gmail.com>
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What is `get_maintainer.pl` and why do you use it before sending a patch? |
| **Intermediate** | What is `checkpatch.pl`? What are the most common issues it catches? |
| **Advanced** | How do you handle reviewer feedback that you disagree with? |
| **Expert** | Describe your process for a multi-patch series. How do you write the cover letter? |
