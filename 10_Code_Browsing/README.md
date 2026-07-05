# 10 — Code Browsing & Navigation

> Professional kernel developers spend 60% of their time reading code. Master the tools to navigate 30 million lines of Linux kernel source efficiently.

---

## Section Structure

```
10_Code_Browsing/
├── 01_VSCode_Kernel_Setup.md         ← clangd, compile_commands.json, extensions
├── 02_Neovim_For_Kernel.md           ← nvim + LSP + telescope for kernel work
├── 03_Elixir_Bootlin.md              ← Online cross-reference: elixir.bootlin.com
├── 04_Sourcegraph.md                 ← Cloud code search, symbol cross-reference
├── 05_Cscope_Ctags.md                ← Traditional tools, still useful in terminals
├── 06_Git_Log_Navigation.md          ← git log, git blame, git bisect for kernel
├── 07_Kernel_Source_Map.md           ← What lives where in the Linux source tree
└── 08_Code_Reading_Strategy.md       ← How to understand an unfamiliar subsystem
```

---

## Why This Matters

When debugging a UFS driver crash, you need to:
1. Find where `ufshcd_probe_hba()` is called from
2. Trace through 5 layers of abstraction in < 2 minutes
3. Find the git commit that introduced a regression
4. Cross-reference the DT binding with the driver

Without good tooling, this takes hours. With the right setup, minutes.

---

## VS Code + clangd Setup (Quick Reference)

```bash
# Generate compile_commands.json for kernel build
cd linux/
make defconfig
bear -- make -j$(nproc)    # intercept build commands
# OR
scripts/clang-tools/gen_compile_commands.py  # kernel 5.13+

# .vscode/settings.json
{
  "clangd.arguments": [
    "--background-index",
    "--clang-tidy",
    "--header-insertion=iwyu",
    "--compile-commands-dir=${workspaceFolder}"
  ],
  "C_Cpp.intelliSenseEngine": "disabled"
}
```

---

## Elixir Bootlin — The Essential Online Tool

**URL:** https://elixir.bootlin.com/linux/latest/source

| Task | How |
|------|-----|
| Find a function definition | Search box → select "Function" |
| Find all callers of a function | Click function name → "Referenced by" |
| Find DT compatible string | Search `qcom,sc7180` in "Identifier" |
| Browse a specific kernel version | Use version dropdown |
| Cross-reference with Git log | Click line number → "git blame" |

---

## Linux Kernel Source Map

```
linux/
├── arch/           ← Architecture-specific code (arm64, x86, riscv...)
│   └── arm64/
│       ├── boot/dts/   ← ALL device trees for ARM64
│       ├── kernel/     ← Entry, exceptions, syscalls
│       └── mm/         ← Architecture memory management
├── drivers/        ← THE BIGGEST DIRECTORY — all device drivers
│   ├── ufs/        ← UFS host controller drivers
│   ├── nvmem/      ← NVMEM subsystem (your QFPROM driver lives here)
│   ├── clk/        ← Clock framework drivers
│   ├── regulator/  ← Regulator/PMIC drivers
│   └── ...
├── fs/             ← Filesystems (ext4, btrfs, proc, sysfs...)
├── include/        ← Kernel headers (linux/, asm/, uapi/)
├── kernel/         ← Core kernel (scheduler, locking, timers...)
├── mm/             ← Memory management (buddy, slab, vmalloc...)
├── net/            ← Networking stack
└── Documentation/  ← Device tree bindings, ABI docs, subsystem docs
```

---

## Git Bisect — Find the Regression Commit

```bash
git bisect start
git bisect bad                    # current HEAD is broken
git bisect good v6.1              # v6.1 was working

# Git checks out midpoint — test it:
make -j$(nproc) && boot_test.sh
git bisect good   # OR:
git bisect bad

# Repeat until Git finds the exact commit:
# "abcdef1234 is the first bad commit"

git bisect reset                  # restore HEAD
```
