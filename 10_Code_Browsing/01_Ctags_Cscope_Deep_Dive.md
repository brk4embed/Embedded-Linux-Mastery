# ctags, cscope & Global — Deep Dive Code Navigation

> **Your Situation:** You've been navigating Linux kernel code and Coreboot for 11 years. But if you're using only grep or clicking through GitHub, you're leaving 80% of productivity on the table. This file takes you from zero to expert on terminal-based code navigation.

---

## What Problem These Tools Solve

**The challenge:** The Linux kernel has 30+ million lines of code across 70,000+ files. When you're debugging a UFS driver crash, you need to:

1. Find where `ufshcd_probe_hba()` is defined
2. Find all places that call it
3. Find all structs that have a `hba` member
4. Find who implements the `scsi_host_ops` for UFS
5. Do all this in **seconds**, not minutes

**The tools:**
- `ctags` — builds an index of symbols (functions, structs, macros) → navigate with Vim/Emacs
- `cscope` — builds a cross-reference database → find callers, definitions, inclusions
- `gtags` (GNU Global) — like ctags + cscope combined, faster, incremental update
- `LSP` (Language Server Protocol) — modern IDE-style navigation (clangd for C)

---

## Tool Comparison

| Tool | Install | What It Finds | Speed | Vim Integration |
|------|---------|--------------|-------|-----------------|
| ctags | apt install universal-ctags | Definitions | Fast | Built-in |
| cscope | apt install cscope | Definitions + callers + text | Medium | Built-in |
| gtags | apt install global | Definitions + callers + refs | Fast + incremental | Via plugin |
| clangd/LSP | apt install clangd | Everything (semantic) | Slow first build | Via coc.nvim/nvim-lsp |

**Recommendation for kernel work:**
- **ctags + cscope**: Works with any editor, no compilation needed, proven 30 years
- **gtags**: Better for large codebases, incremental updates
- **clangd**: Best accuracy but needs `compile_commands.json`

---

## 1. ctags — From Scratch to Expert

### Install

```bash
# Install universal-ctags (better than exuberant-ctags)
sudo apt install universal-ctags

# Verify
ctags --version
# Universal Ctags 6.0.0, ...
```

### Generate Tags for the Linux Kernel

```bash
cd /path/to/linux/

# Method 1: Kernel's built-in make target (BEST for kernel)
make tags                 # generates ./tags (takes ~2 minutes)
make TAGS                 # for Emacs format

# Method 2: Manual ctags (for partial trees or custom config)
ctags -R --languages=C,C++ --exclude=.git --exclude=Documentation \
         --exclude="*.o" --exclude="*.cmd" .

# Method 3: Targeted (only drivers, much faster)
ctags -R drivers/ include/ arch/arm64/

# Check tag file size
wc -l tags
# 3,500,000 lines — that's 3.5 million symbols indexed!
```

### .ctags Configuration File (Put in ~/.ctags or ~/.ctags.d/)

```bash
mkdir -p ~/.ctags.d/
cat > ~/.ctags.d/kernel.ctags << 'EOF'
# Exclude generated files
--exclude=*.o
--exclude=*.ko
--exclude=*.cmd
--exclude=*.d
--exclude=.git
--exclude=Documentation
--exclude=scripts

# Add kernel-specific patterns
--langdef=kconfig
--langmap=kconfig:Kconfig
--regex-kconfig=/^(menu)?config[ \t]+([A-Z0-9_]+)/\2/c,config/

# C-specific: add struct/enum/typedef as separate tags
--c-kinds=+px
--fields=+aS
--extra=+q
EOF
```

### Vim Integration (No Plugins Needed)

ctags works with Vim out-of-the-box:

```vim
" In vim, navigate with these:
Ctrl+]        " Jump to definition of word under cursor
Ctrl+T        " Jump back (pop the tag stack)
Ctrl+W Ctrl+] " Open definition in split window
:tag func_name  " Jump to a named tag
:tselect       " Show all matches for tag (when ambiguous)
:tnext         " Next match
:tprev         " Previous match
:tags          " Show tag stack (your navigation history)

" Preview tag without leaving current file:
Ctrl+W }      " Open tag in preview window
Ctrl+W z      " Close preview window
```

### Practical Navigation Exercise (Linux Kernel)

```bash
cd ~/linux/
make tags

# Open a driver file in vim
vim drivers/ufs/core/ufshcd.c
```

Now in Vim:
1. Place cursor on `ufshcd_probe_hba` → press `Ctrl+]` → jumps to definition
2. Press `Ctrl+T` → back to where you were
3. Place cursor on `struct ufs_hba` → `Ctrl+]` → jumps to struct definition
4. `:tselect ufshcd_abort` → shows all matches (may find multiple declarations)
5. Press `Ctrl+W ]` → open in split window (keep both files visible)

### Tag Navigation in Practice

```bash
# From shell: find what file defines a symbol
grep -m1 "ufshcd_probe_hba" tags
# ufshcd_probe_hba	drivers/ufs/core/ufshcd.c	/^static int ufshcd_probe_hba/;"	f

# Vim: search for pattern in tags
:tag /^ufshcd_.*abort/    " search tags matching regex
```

---

## 2. cscope — Find Callers and References

### Why cscope is Essential

ctags only finds **definitions**. cscope also finds:
- Every place a function is **called**
- Every file that **includes** a header
- Every place a symbol is **assigned**
- Text string search in source

```bash
# Install
sudo apt install cscope

# Generate cscope database for kernel
cd ~/linux/

# Method 1: Kernel's make target
make cscope

# Method 2: Manual
find . -name "*.c" -o -name "*.h" | grep -v ".git" > cscope.files
cscope -b -q -k -i cscope.files
# -b = build database and exit
# -q = faster queries
# -k = use kernel headers (don't include /usr/include)
# Output: cscope.out, cscope.out.in, cscope.out.po

# Check database size
ls -lh cscope.out*
# cscope.out  500MB  (full kernel)
```

### Vim Integration (Built-in)

```vim
" Connect vim to cscope database (put in .vimrc):
if has("cscope")
    set csprg=/usr/bin/cscope
    set csto=0          " search cscope first, then ctags
    set cst             " use :cstag instead of :tag
    cs add cscope.out   " add database
    set csverb          " show messages
endif

" Keyboard shortcuts for cscope:
" (add to .vimrc)
nmap <C-\>s :cs find s <C-R>=expand("<cword>")<CR><CR>  " symbol
nmap <C-\>g :cs find g <C-R>=expand("<cword>")<CR><CR>  " definition
nmap <C-\>c :cs find c <C-R>=expand("<cword>")<CR><CR>  " callers
nmap <C-\>t :cs find t <C-R>=expand("<cword>")<CR><CR>  " text string
nmap <C-\>f :cs find f <C-R>=expand("<cfile>")<CR><CR>  " file
nmap <C-\>i :cs find i <C-R>=expand("<cfile>")<CR><CR>  " includes
```

### cscope Queries Explained

```
Query type:
  s = find all uses of symbol (variable, function name)
  g = find global definition
  c = find functions calling this function
  t = find text string
  e = find regex pattern  
  f = find file by name
  i = find files including this header
  d = find functions called by this function

Example session in vim:
  1. Cursor on "ufshcd_probe_hba"
  2. Ctrl+\ c → shows ALL callers:
       ufshcd_init() calls it
       ufshcd_reset_and_restore() calls it
  3. Select one → jump there
  4. Ctrl+\ d → shows all functions ufshcd_probe_hba CALLS
```

### cscope Interactive Mode (Without Vim)

```bash
# Launch cscope TUI
cscope -d           # -d = don't rebuild, use existing database

# Navigate:
# Tab     = switch between search types and results
# Enter   = select
# Ctrl+D  = exit

# Non-interactive (script/automation):
cscope -d -L1 ufshcd_probe_hba    # find callers of ufshcd_probe_hba
cscope -d -L2 ufshcd_probe_hba    # find definition
```

---

## 3. GNU Global (gtags) — Best of Both Worlds

### Why gtags Beats ctags + cscope for Big Projects

```
ctags:  defines only, no callers, full rebuild every time
cscope: callers + defines, but slow full rebuild
gtags:  callers + defines + refs, INCREMENTAL update, fastest queries
```

```bash
# Install
sudo apt install global

# Generate database (Linux kernel ~5 minutes first time)
cd ~/linux/
gtags                    # creates GTAGS, GRTAGS, GPATH

# Update incrementally (very fast — only processes changed files)
global -u

# Query examples:
global ufshcd_probe_hba          # find definition
global -r ufshcd_probe_hba       # find all references (callers)
global -s ufshcd_probe_hba       # find symbol (not definitions)
global -e "ufshcd_.*error"       # regex search

# File-level:
global -f drivers/ufs/core/ufshcd.c   # all tags in this file
global -P "qfprom"                     # find files matching pattern
```

### Vim Integration via vim-gutentags (Auto-updates tags)

```vim
" Install vim-plug first, then add to .vimrc:
Plug 'ludovicchabant/vim-gutentags'

" Config:
let g:gutentags_modules = ['ctags', 'gtags_cscope']
let g:gutentags_cache_dir = expand('~/.cache/tags')
let g:gutentags_project_root = ['.git', 'Makefile']

" Now tags auto-update when you save files!
```

---

## 4. Practical Setup: Your Linux Kernel Dev Environment

### One-Time Setup Script

```bash
#!/bin/bash
# setup_kernel_nav.sh — Run once per kernel tree

KERNEL_DIR="${1:-$PWD}"
cd "$KERNEL_DIR" || exit 1

echo "=== Building navigation databases for kernel tree ==="

# 1. ctags (vim :tag, Ctrl+])
echo "[1/3] Building ctags..."
make tags 2>/dev/null || ctags -R --languages=C --exclude=.git .
echo "  Done: $(wc -l < tags) symbols"

# 2. cscope (vim :cs find c = callers)
echo "[2/3] Building cscope..."
make cscope 2>/dev/null || {
    find . -name "*.c" -o -name "*.h" | grep -v ".git" > cscope.files
    cscope -b -q -k -i cscope.files
}
echo "  Done: $(ls -lh cscope.out | awk '{print $5}') database"

# 3. gtags (global -r = references/callers)
echo "[3/3] Building gtags..."
gtags 2>/dev/null
echo "  Done"

echo ""
echo "=== Navigation ready! ==="
echo "In vim:"
echo "  Ctrl+]       = jump to definition"
echo "  Ctrl+\\ c     = show all callers"
echo "  Ctrl+\\ d     = show all callees"
echo "  Ctrl+T       = jump back"
```

```bash
chmod +x setup_kernel_nav.sh
cd ~/linux/
./setup_kernel_nav.sh
```

### .vimrc for Kernel Development

```vim
" === .vimrc for kernel/embedded development ===

" ctags: search upward for tags file
set tags=./tags,tags;/

" cscope: auto-load cscope.out
if has("cscope")
    set csprg=/usr/bin/cscope
    set csto=0
    set cst
    set nocsverb
    if filereadable("cscope.out")
        cs add cscope.out
    endif
    set csverb
endif

" cscope shortcuts (Ctrl+\ + letter)
nmap <C-\>s :cs find s <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>g :cs find g <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>c :cs find c <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>t :cs find t <C-R>=expand("<cword>")<CR><CR>
nmap <C-\>f :cs find f <C-R>=expand("<cfile>")<CR><CR>
nmap <C-\>i :cs find i <C-R>=expand("<cfile>")<CR><CR>
nmap <C-\>d :cs find d <C-R>=expand("<cword>")<CR><CR>

" Highlight current word matches
set hlsearch
nnoremap <F3> :set hlsearch!<CR>

" Line numbers
set number
set relativenumber

" Show function prototype at bottom when navigating
set ruler
```

---

## 5. Practical Exercises

### Exercise 1: Navigate the UFS Driver (30 min)

```bash
cd ~/linux/
make tags cscope

vim drivers/ufs/core/ufshcd.c
```

1. Find `ufshcd_probe_hba` — press `Ctrl+]` → see definition
2. `Ctrl+T` back, then `Ctrl+\ c` on same word → see all callers
3. Navigate to `struct ufs_hba` definition → `Ctrl+]` on struct name
4. `Ctrl+\ i` on `"ufshcd.h"` → see all files that include it
5. Press `:tags` → see your navigation history

### Exercise 2: Find Your QFPROM Code Paths (20 min)

```bash
vim drivers/nvmem/qfprom.c

# 1. Find qfprom_read function
# 2. Find who calls qfprom_read (Ctrl+\ c)
# 3. Trace back through the nvmem subsystem
# 4. Find struct nvmem_config — where is it defined?
# 5. Find all drivers using nvmem_register()
```

### Exercise 3: Compare ctags vs clangd Navigation (15 min)

```bash
# ctags navigation (offline, no compilation needed):
cd ~/linux/ && make tags
vim drivers/nvmem/qfprom.c
# Ctrl+] on any symbol

# clangd navigation (semantic, needs compile_commands.json):
cd ~/linux/
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
bear -- make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
# Now clangd can navigate with full type information
```

---

## Quick Reference Card

```
TASK                           TOOL        KEY/COMMAND
────────────────────────────   ─────────   ──────────────────────────
Jump to definition             ctags/vim   Ctrl+]
Jump back                      ctags/vim   Ctrl+T
Open def in split              ctags/vim   Ctrl+W ]
Search tags with regex         ctags/vim   :tag /pattern/
Find all callers               cscope/vim  Ctrl+\ c
Find all definitions           cscope/vim  Ctrl+\ g
Find symbol uses               cscope/vim  Ctrl+\ s
Find files including header    cscope/vim  Ctrl+\ i
Find all references (gtags)    gtags CLI   global -r function_name
Find in specific file          gtags CLI   global -f filename.c
Rebuild database               gtags CLI   global -u (incremental)
Generate kernel tags           make        make tags
Generate kernel cscope         make        make cscope
```
