# 33 — Code Diff, Merge & Comparison Tools

> **Why This Matters for You:** In your 11 years you have reviewed patches, maintained 50-deep Coreboot patch-trains, and sent kernel patches to LKML. Every one of those activities depends on confident use of diff/merge tools. This section takes you from zero to expert, with kernel-specific workflows built in.

---

## What You Will Learn (Beginner → Expert)

```
Level 1 (Beginner)  : What is a diff? Reading a unified diff output
Level 2 (Basic)     : diff, patch commands — the Unix foundation
Level 3 (Practical) : git diff, git difftool — daily kernel workflow
Level 4 (Visual)    : meld, vimdiff, kdiff3 — GUI tools
Level 5 (Advanced)  : quilt — managing patch stacks (Coreboot/Yocto)
Level 6 (Expert)    : 3-way merge, conflict resolution strategies
```

---

## Section Structure

```
33_Code_Diff_Merge_Tools/
├── 01_Diff_From_Scratch.md           ← What is a diff, unified format explained
├── 02_Git_Diff_Mastery.md            ← git diff, git show, git log -p deep dive
├── 03_Meld_Visual_Diff.md            ← Meld: install, 2-way, 3-way, directory diff
├── 04_Vimdiff_Nvim.md                ← vimdiff / nvim -d keyboard shortcuts
├── 05_Kdiff3_And_Beyond_Compare.md   ← kdiff3 and commercial alternatives
├── 06_Quilt_Patch_Management.md      ← quilt for Coreboot/Yocto patch stacks
├── 07_Patch_And_Diff_Workflows.md    ← format-patch, apply, am, cherry-pick
├── 08_Conflict_Resolution.md         ← 3-way merge, resolving conflicts properly
└── 09_Kernel_Review_Workflow.md      ← Reviewing upstream patches efficiently
```

---

## Level 1: What Is a Diff? (Start Here)

### The Concept (Explained Like You're New)

A **diff** answers one question: *"What changed between version A and version B of a file?"*

Imagine you have two versions of a driver file:

```
Version A (old)          Version B (new)
─────────────────────    ─────────────────────
int my_probe(...)  {     int my_probe(...)  {
    int ret;                 int ret;
    ret = clk_get();         ret = clk_prepare_get();   ← changed
    if (ret < 0)             if (ret)                   ← changed
        return ret;              return ret;
    return 0;                return 0;
}                        }
```

The diff output (unified format) looks like this:

```diff
--- a/drivers/my_driver.c
+++ b/drivers/my_driver.c
@@ -3,7 +3,7 @@ int my_probe(...) {
     int ret;
-    ret = clk_get();
+    ret = clk_prepare_get();
-    if (ret < 0)
+    if (ret)
         return ret;
     return 0;
 }
```

### Reading the Unified Diff Format

```
--- a/file.c          ← OLD file (minus = removed)
+++ b/file.c          ← NEW file (plus = added)
@@ -3,7 +3,7 @@      ← Hunk header: old_start,old_count new_start,new_count
 int ret;             ← Space = context (unchanged)
-    ret = clk_get(); ← Minus = removed line (was in old)
+    ret = clk_prep();← Plus = added line (new version)
```

**Hunk header decoded:** `@@ -3,7 +3,7 @@`
- `-3,7` = starting at line 3, showing 7 lines from OLD file
- `+3,7` = starting at line 3, showing 7 lines from NEW file

---

## Level 2: The `diff` and `patch` Commands

### `diff` — The Unix Foundation

```bash
# Install
sudo apt install diffutils

# Basic 2-file diff (unified format — always use -u for kernel work)
diff -u old_file.c new_file.c

# Recursive directory diff (compare two BSP versions)
diff -urN old_bsp_dir/ new_bsp_dir/

# Ignore whitespace differences
diff -u --ignore-all-space file1.c file2.c

# Generate a patch file
diff -u old_driver.c new_driver.c > my_fix.patch

# Colorized output (install colordiff first)
sudo apt install colordiff
diff -u file1.c file2.c | colordiff
```

### `patch` — Applying a Diff

```bash
# Apply a patch file to your source
patch -p1 < my_fix.patch

# -p1 means: strip 1 leading path component
# diff output has:  --- a/drivers/my.c
# -p0 looks for:        a/drivers/my.c  (usually wrong)
# -p1 looks for:          drivers/my.c  (correct for kernel)

# Dry run (check without applying)
patch --dry-run -p1 < my_fix.patch

# Reverse a patch (undo it)
patch -R -p1 < my_fix.patch

# Apply to a specific directory
patch -p1 -d /path/to/source < my_fix.patch
```

### Real Example: Patching U-Boot

```bash
# Scenario: You have a fix for a U-Boot board file
cd u-boot/
diff -u board/qualcomm/sc7180/Makefile.bak board/qualcomm/sc7180/Makefile > fix.patch

# Check the patch
cat fix.patch
# --- a/board/qualcomm/sc7180/Makefile
# +++ b/board/qualcomm/sc7180/Makefile
# @@ -5,6 +5,7 @@
# +obj-y += qspi.o

# Apply to a clean checkout
cd /tmp/u-boot-clean/
patch -p1 < /path/to/fix.patch

# Verify it applied
grep -n "qspi.o" board/qualcomm/sc7180/Makefile
```

---

## Level 3: Git Diff Mastery (Daily Workflow)

```bash
# ─── What changed in working tree vs staging ────────────────────
git diff                    # unstaged changes
git diff --staged           # staged changes (what will be committed)
git diff HEAD               # all changes (staged + unstaged)

# ─── Compare specific commits ───────────────────────────────────
git diff HEAD~1             # last commit vs current
git diff v6.8..v6.9         # between two kernel releases
git diff abc123..def456     # between two specific commits

# ─── Show changes to a specific file ───────────────────────────
git diff HEAD drivers/nvmem/qfprom.c
git log -p drivers/nvmem/qfprom.c   # full history with diffs

# ─── Statistics ─────────────────────────────────────────────────
git diff --stat             # files changed + lines added/removed
git diff --shortstat        # one-line summary
git show --stat HEAD        # last commit stats

# ─── Word-level diff (very useful for commit messages/docs) ─────
git diff --word-diff

# ─── Ignore whitespace ──────────────────────────────────────────
git diff -w                 # ignore all whitespace
git diff --ignore-blank-lines

# ─── Color (always on in modern git, but explicit) ───────────────
git diff --color=always | less -R
```

### git difftool — Launch Visual Tool from Git

```bash
# Configure meld as difftool
git config --global diff.tool meld
git config --global difftool.prompt false

# Now use meld for any diff
git difftool HEAD~1
git difftool HEAD~3 HEAD drivers/ufs/

# Configure for mergetool too
git config --global merge.tool meld
git mergetool    # resolve conflicts with meld
```

---

## Level 4: Meld — Best Visual Diff Tool for Linux

### Install

```bash
sudo apt install meld
```

### 2-Way File Comparison

```bash
meld old_driver.c new_driver.c
```

**Meld Interface:**
```
┌─────────────────┬───────────────┬─────────────────┐
│   old_driver.c  │  [arrows/nav] │  new_driver.c   │
│                 │               │                 │
│  int my_probe() │←─────────────→│  int my_probe() │
│    int ret;     │               │    int ret;     │
│  ret = clk_get()│  ══════════   │  ret=clk_prep() │ ← highlighted difference
│  if (ret < 0)   │  ══════════   │  if (ret)       │
└─────────────────┴───────────────┴─────────────────┘
```

**Key shortcuts in Meld:**
```
Ctrl+D      → Next difference
Ctrl+E      → Previous difference
Ctrl+Right  → Copy right (copy current block to right file)
Ctrl+Left   → Copy left (copy current block to left file)
Ctrl+S      → Save
Ctrl+Z      → Undo
```

### 3-Way Merge (Conflict Resolution)

```bash
# 3-way merge: your version, original, their version
meld mine.c original.c theirs.c

# Or let git call it for conflict resolution
git mergetool --tool=meld
```

**3-Way View:**
```
┌──────────┬───────────┬──────────┐
│  MINE    │ ORIGINAL  │  THEIRS  │
│ (local)  │ (base)    │ (remote) │
└──────────┴───────────┴──────────┘
         ↓ merge result ↓
┌─────────────────────────────────┐
│         MERGED OUTPUT           │
└─────────────────────────────────┘
```

### Directory Comparison

```bash
# Compare two kernel versions
meld linux-6.8/ linux-6.9/

# Compare your BSP branch with upstream
meld arch/arm64/boot/dts/qcom/ upstream/arch/arm64/boot/dts/qcom/
```

---

## Level 4b: vimdiff — Never Leave the Terminal

```bash
# Open two files side by side
vimdiff old.c new.c

# Or from nvim
nvim -d old.c new.c

# 3-way diff
vimdiff file1.c file2.c file3.c
```

### vimdiff Keyboard Shortcuts

```
]c          → Jump to NEXT difference (c = change)
[c          → Jump to PREVIOUS difference
do          → diff obtain (pull change from other window into current)
dp          → diff put (push change from current window to other)
:diffupdate → refresh diff highlighting after editing
:only       → close all windows except current
zo          → open folded context
zc          → close unchanged context (fold it)
Ctrl+w w    → switch between windows
:wqa        → save all and quit
```

### Real Workflow: Review a Kernel Patch in vimdiff

```bash
# You received a patch to review. Apply to temp copy:
cp drivers/nvmem/qfprom.c /tmp/qfprom_orig.c
git apply patch.patch
vimdiff /tmp/qfprom_orig.c drivers/nvmem/qfprom.c

# Navigate every change with ]c / [c
# Check each modification carefully
```

---

## Level 5: quilt — Patch Stack Management

### What Is quilt? (Critical for Coreboot/Yocto Work)

`quilt` manages a **stack of patches** on top of a source tree. Instead of one big commit, you have:

```
Source base
    ↓
[patch 01] add-uart-support.patch
    ↓
[patch 02] add-ddr-init.patch
    ↓
[patch 03] fix-clock-config.patch
    ↓
[patch 04] add-pmic-support.patch
    ↓
[patch 50] final-board-config.patch   ← your 50-patch train!
```

You can **push** (apply), **pop** (unapply), **refresh** (update), and **fold** patches.

### Install and Basic Usage

```bash
sudo apt install quilt

# Set up quilt config (put in ~/.quiltrc)
cat > ~/.quiltrc << 'EOF'
QUILT_PATCHES=patches
QUILT_NO_DIFF_INDEX=1
QUILT_NO_DIFF_TIMESTAMPS=1
QUILT_REFRESH_ARGS="-p ab --no-timestamps --no-index"
QUILT_DIFF_ARGS="--color=auto"
EOF

# Initialize quilt in a source directory
cd coreboot/
quilt init           # creates 'patches/' directory

# Create a new patch
quilt new 0001-add-sc7280-uart.patch
quilt add src/soc/qualcomm/sc7280/uart.c   # tell quilt to track this file
# ... edit the file ...
quilt refresh         # update the patch with your changes

# Add more files to current patch
quilt add src/soc/qualcomm/sc7280/Makefile.inc

# Check what's in current patch
quilt diff

# Apply all patches
quilt push -a

# Pop all patches (back to clean source)
quilt pop -a

# Show patch stack status
quilt series          # list all patches
quilt applied         # show applied patches
quilt unapplied       # show unapplied patches
quilt top             # show current top patch
```

### Managing Your Coreboot Patch Train

```bash
# Work on patch 23 without losing patches 24-50:
quilt pop -a                    # unapply all 50
quilt push 23                   # apply up to patch 23

# Edit the file
vim src/soc/qualcomm/sc7180/soc.c

# Update patch 23 with your edit
quilt refresh

# Re-apply the rest
quilt push -a

# If later patches conflict:
quilt push                      # apply one at a time
# ... fix conflict ...
quilt refresh                   # refresh the conflicting patch
quilt push                      # continue
```

---

## Level 6: 3-Way Merge — Conflict Resolution

### What Causes a Merge Conflict?

```
          commit A (your change)
         /
base ───┤
         \
          commit B (their change)
```

Both A and B modified the **same lines**. Git cannot automatically merge.

### Conflict Markers Explained

```c
<<<<<<< HEAD           ← YOUR version starts here
    ret = clk_prepare_enable(clk);
    if (ret)
=======                ← dividing line
    ret = clk_get(dev, "core");
    if (ret < 0)
>>>>>>> upstream/main  ← THEIR version ends here
        return ret;
```

### Resolving Strategies

```bash
# Option 1: Keep yours
git checkout --ours drivers/my_driver.c

# Option 2: Keep theirs
git checkout --theirs drivers/my_driver.c

# Option 3: Manual edit (most common for kernel work)
vim drivers/my_driver.c
# Delete the conflict markers, write the correct merged code
git add drivers/my_driver.c
git commit

# Option 4: Use meld for visual 3-way merge
git mergetool --tool=meld
```

### Kernel Merge Best Practice

For kernel patches, the correct approach is almost always **manual edit** + **test**:

```bash
# 1. Understand BOTH changes (what was each trying to do?)
git log --oneline HEAD...MERGE_HEAD -- drivers/my_driver.c

# 2. Read both changes in context
git show HEAD -- drivers/my_driver.c
git show MERGE_HEAD -- drivers/my_driver.c

# 3. Write the merged version that accomplishes BOTH goals
vim drivers/my_driver.c    # resolve manually

# 4. Build and test
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- drivers/my_driver.o

# 5. Mark resolved and commit
git add drivers/my_driver.c
git commit
```

---

## Level 7: Reviewing Kernel Patches Efficiently

```bash
# Reviewing a patch series from LKML (your QFPROM workflow):

# 1. Download patch from lore.kernel.org
b4 am https://lore.kernel.org/linux-nvmem/xxx/       # b4 tool
# OR
wget https://lore.kernel.org/.../.../mbox -O patch.mbox

# 2. Apply to your tree
git am patch.mbox

# 3. Review each commit
git log --oneline -10           # see what was applied
git show HEAD                   # review latest commit
git show HEAD --stat            # stats only

# 4. Check style
scripts/checkpatch.pl --git HEAD

# 5. Build test
make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

# 6. Send reviewed-by (after you're satisfied)
git format-patch origin..HEAD
# Add "Reviewed-by: Ravi Kumar Bokka <brk4embed@gmail.com>" to patch

# Install b4 tool (very useful for patch management)
pip install b4
```

---

## Practical Exercises

### Exercise 1: Create Your First Patch (15 minutes)
```bash
# 1. Clone a simple project
git clone https://github.com/torvalds/linux linux-exercise
cd linux-exercise

# 2. Create a branch
git checkout -b my-fix

# 3. Make a small, safe change (add a comment)
vim drivers/nvmem/qfprom.c
# Add a comment explaining something

# 4. Generate the patch
git add drivers/nvmem/qfprom.c
git commit -m "nvmem: qfprom: Add comment explaining register layout"
git format-patch HEAD~1

# 5. View your patch
cat 0001-nvmem-qfprom-*.patch

# 6. Check the diff
git diff HEAD~1
```

### Exercise 2: Meld vs vimdiff Comparison (20 minutes)
```bash
# 1. Create two versions of a file
cp drivers/nvmem/qfprom.c /tmp/qfprom_v1.c
cp drivers/nvmem/qfprom.c /tmp/qfprom_v2.c
# Edit v2: change a few lines manually

# 2. Compare with meld
meld /tmp/qfprom_v1.c /tmp/qfprom_v2.c

# 3. Compare same files with vimdiff
vimdiff /tmp/qfprom_v1.c /tmp/qfprom_v2.c

# 4. Practice navigating: ]c ]c ]c [c [c
# 5. Practice: copy a change from right to left: dp
```

### Exercise 3: quilt Patch Stack (30 minutes)
```bash
git clone https://review.coreboot.org/coreboot
cd coreboot
quilt init
quilt new 0001-my-test.patch
quilt add src/mainboard/google/Makefile.inc
echo "# test comment" >> src/mainboard/google/Makefile.inc
quilt refresh
quilt diff
quilt pop
quilt push
```

---

## Quick Reference Card

```
TOOL          BEST USE                         COMMAND
──────────    ────────────────────────────     ──────────────────────────────
diff -u       Generate a patch file            diff -u old.c new.c > fix.patch
patch -p1     Apply a patch                    patch -p1 < fix.patch
git diff      See uncommitted changes          git diff / git diff --staged
git difftool  Launch visual tool               git difftool HEAD~1
meld          Visual 2-way or 3-way compare    meld file1.c file2.c
vimdiff       In-terminal visual compare       vimdiff file1.c file2.c
kdiff3        3-way merge with auto-resolve    kdiff3 base.c mine.c theirs.c
quilt         Manage ordered patch stacks      quilt push/pop/refresh/series
b4 am         Apply LKML patches               b4 am <lore-url>
git mergetool Resolve conflicts visually       git mergetool --tool=meld
```
