# 26 — Developer Commands

> **Your daily toolkit.** Vim + Linux commands + GDB + kernel debug commands. Master these and you'll be 3× faster than engineers who don't.

---

## Section Structure

```
26_Developer_Commands/
├── 01_Vim_For_Embedded_Engineers.md     ← Vim beginner to expert
├── 02_Linux_Power_Commands.md           ← find, grep, awk, sed, strace, lsof
├── 03_GDB_Mastery.md                    ← GDB for embedded + kernel debugging
├── 04_Kernel_Debug_Commands.md          ← /proc, /sys, dmesg, ftrace, eBPF
├── 05_Serial_Console_Workflow.md        ← minicom, tio, picocom mastery
├── 06_Git_Advanced_Commands.md          ← Git for kernel/Coreboot workflows
├── 07_tmux_Workflow.md                  ← Multi-window development environment
├── 08_SSH_And_Remote_Dev.md             ← Remote development on embedded boards
└── 09_Quick_Reference_Cheatsheets.md    ← All commands on one page
```

---

## 01 — Vim for Embedded Engineers

### Why Vim (Not VS Code) for Embedded Work

```
Scenarios where Vim wins:
✓ SSH into an embedded board with no GUI
✓ Editing files in initramfs or minimal rootfs
✓ QEMU console editing (no mouse)
✓ Fast navigation in 1+ million line kernel source
✓ Remote editing over slow/unreliable connections
✓ Boot scripts, init scripts on minimal systems

VS Code also:
✓ Neovim + clangd LSP = VS Code quality editing, terminal native
✓ See: 27_AI_Dev_Environment/ for full VS Code setup
```

### Vim Survival Commands (Beginner)

```vim
" Navigation
h j k l          ← ↓ ↑ →  (movement)
w W              next word / WORD
b B              prev word / WORD
e E              end of word / WORD
0 ^              start of line / first non-space
$                end of line
gg G             top / bottom of file
Ctrl+d Ctrl+u    half page down/up
Ctrl+f Ctrl+b    full page down/up
:N               go to line N

" Search
/pattern         search forward
?pattern         search backward
n N              next/prev match
*                search word under cursor
# (shift-3)      search word under cursor backward

" Edit
i I              insert before cursor / at line start
a A              append after cursor / at line end
o O              new line below / above and enter insert
x X              delete char forward / backward
dd               delete line
yy               yank (copy) line
p P              paste after / before
u                undo
Ctrl+r           redo
.                repeat last command

" Save/quit
:w               write (save)
:q               quit
:wq or ZZ        save and quit
:q!              quit without saving
:x               save only if changed, then quit

" Visual mode
v                character visual mode
V                line visual mode
Ctrl+v           block visual mode
(select, then):
d                delete selection
y                yank selection
>  <             indent / unindent
```

### Vim Productivity (Intermediate)

```vim
" Splits (essential for driver work)
:vsplit file.c   " vertical split
:split file.c    " horizontal split
Ctrl+w h/j/k/l  " navigate splits
Ctrl+w =         " equalize split sizes
:close           " close current split

" Multiple files
:e file.c        " open file
:ls              " list open buffers
:b2              " switch to buffer 2
:bn :bp          " next/prev buffer
:bd              " close buffer

" Search and replace
:%s/old/new/g    " replace all in file
:%s/old/new/gc   " replace all with confirmation
:1,20s/old/new/g " replace in lines 1-20
" Tip: use \c for case-insensitive, \C for case-sensitive

" Macros (massive time saver)
qa               " start recording macro 'a'
[do stuff]
q                " stop recording
@a               " play macro
5@a              " play macro 5 times

" Ex commands for bulk edits
:argdo %s/old/new/ge | w   " replace in all open files
:bufdo %s/old/new/ge | w   " replace in all buffers
```

### Vim for Kernel Source Navigation (Advanced)

```bash
# Set up ctags for kernel source
cd ~/kernel/linux
ctags -R --extra=+fq --exclude='.git' --c-kinds=+p

# Set up cscope for kernel source
cscope -b -R -q -k
# -k: suppress /usr/include (kernel doesn't use it)
```

```vim
" ctags navigation (in Vim)
Ctrl+]           " jump to definition of word under cursor
Ctrl+t           " jump back
:tag symbol      " jump to symbol
:ts symbol       " list all matching tags
:tp :tn          " prev/next tag match

" cscope navigation
:cs find s symbol   " find all uses of symbol
:cs find d symbol   " find functions called by symbol  
:cs find c symbol   " find functions that call symbol
:cs find t text     " find text
:cs find e egrep    " egrep pattern
:cs find i file     " find files #including this file

" Quickfix list (great for search results)
:copen           " open quickfix window
:cn :cp          " next/prev quickfix item
:cclose          " close quickfix

" Example workflow: find all calls to ufshcd_probe
:cs find c ufshcd_probe
" → opens quickfix list showing all call sites
:copen
" → navigate with Enter
```

### Vim Config for Embedded Development

```vim
" ~/.vimrc — Optimized for kernel/embedded development
set nocompatible
filetype plugin indent on
syntax on

" Display
set number          " line numbers
set relativenumber  " relative line numbers (for fast navigation)
set cursorline      " highlight current line
set scrolloff=5     " keep 5 lines above/below cursor visible
set colorcolumn=80  " show column 80 guide (kernel style limit)
set ruler           " show cursor position in statusline

" Indentation (kernel uses tabs)
set noexpandtab     " use real tabs (kernel coding style)
set tabstop=8       " tab = 8 spaces (kernel standard)
set shiftwidth=8    " indent = 8 spaces
set softtabstop=8

" Search
set incsearch       " search as you type
set hlsearch        " highlight matches
set ignorecase      " case insensitive
set smartcase       " but case-sensitive if uppercase in pattern

" Performance
set lazyredraw      " don't redraw during macros
set hidden          " allow hidden buffers without saving
set history=1000    " remember more commands

" File navigation
set wildmenu        " enhanced command completion
set wildmode=longest,list  " tab completion like bash

" ctags
set tags=./tags,tags;/  " find tags file in current or parent dirs

" Quick save
nnoremap <leader>w :w<CR>
" Navigate splits with Ctrl+movement
nnoremap <C-h> <C-w>h
nnoremap <C-j> <C-w>j
nnoremap <C-k> <C-w>k
nnoremap <C-l> <C-w>l

" Clear search highlight
nnoremap <Esc><Esc> :nohlsearch<CR>

" Make working on kernel files with long lines bearable
set nowrap
nnoremap j gj
nnoremap k gk
```

---

## 02 — Linux Power Commands

### Essential File Operations

```bash
# Find files
find . -name "*.c" -type f                      # find C files
find . -name "Kconfig" -path "*/drivers/*"      # Kconfig in drivers
find . -mtime -1 -name "*.c"                    # C files modified in last day
find . -size +100k -name "*.c"                  # C files > 100KB
find . -name "*.c" -exec grep -l "ufshcd" {} \; # C files containing ufshcd

# Grep with context
grep -r "ufshcd_probe" drivers/scsi/            # recursive grep
grep -rn "devm_kzalloc" drivers/ufs/ | head -20 # with line numbers
grep -rI "CONFIG_UFS" arch/arm64/               # only text files
grep -A5 -B5 "ufshcd_probe" drivers/ufs/ufshcd.c  # 5 lines before/after

# Advanced grep for kernel work
# Find all functions that return -ENOMEM
grep -rn "return -ENOMEM" drivers/ufs/
# Find all DT compatible strings
grep -rn "\.compatible\s*=" drivers/ufs/
# Find all MODULE_DEVICE_TABLE entries
grep -rn "MODULE_DEVICE_TABLE" drivers/ | grep "of"

# Text processing
awk '{print $1, $3}' file.txt               # print columns 1 and 3
awk '/error/{print NR": "$0}' dmesg.log     # print error lines with numbers
sed 's/old/new/g' file.c                    # replace in stream
sed -n '100,200p' large_file.c              # print lines 100-200
sort -k3 -n file.txt                        # sort by 3rd column numerically
uniq -c sorted.txt                          # count unique lines

# Process info
ps aux | grep kworker                       # find kernel worker threads
ps -eLf | grep kworker | wc -l             # count kernel threads
cat /proc/PID/maps                          # memory map of process
cat /proc/PID/status                        # process status
cat /proc/PID/cmdline | tr '\0' ' '        # command line

# System info
cat /proc/cpuinfo | grep -E "model|cpu MHz"
cat /proc/meminfo | grep -E "MemTotal|MemFree|Cached"
cat /proc/interrupts | head -30             # IRQ statistics
cat /proc/iomem | grep -v "  "            # IO memory map (top level)
cat /proc/ioports                           # IO port map (x86 only)
lsmod | sort -k3 -n                         # list modules sorted by use count
```

### System Debugging Commands

```bash
# strace — trace system calls
strace ls -la 2>&1 | head -30               # basic strace
strace -e trace=open,read,write ls          # trace specific syscalls
strace -p PID                               # attach to running process
strace -f ./my_program                      # follow forks
strace -c ./my_program                      # summary with counts/times

# ltrace — trace library calls
ltrace ./my_program 2>&1 | head -20

# lsof — list open files (great for driver debugging)
lsof /dev/ttyS0                             # who has this device open
lsof -p PID                                 # all files opened by process
lsof -i :22                                 # process using TCP port 22

# ldd — shared library dependencies
ldd /usr/bin/python3
ldd /usr/bin/ssh

# objdump / readelf
objdump -d hello.o | head -30               # disassemble
readelf -h vmlinux                          # ELF header
readelf -S vmlinux | grep -E "text|data|bss"  # sections
nm vmlinux | grep " T " | grep ufshcd      # kernel symbols
nm --size-sort vmlinux | tail -20           # largest symbols

# addr2line — convert address to source line
addr2line -e vmlinux -f 0xffffff8008abc123  # kernel Oops address

# Hardware probing
lspci -vvv                                   # verbose PCI info
lspci -k                                     # PCI devices with kernel driver
lsusb -v                                     # USB devices verbose
lsscsi                                       # SCSI/SATA/UFS devices
udevadm info /dev/mmcblk0                   # device information
udevadm monitor                              # watch udev events live

# Kernel module info
modinfo ufshcd                               # module information
modprobe -v ufshcd                           # load module verbose
rmmod -f ufshcd                              # force remove module
```

---

## 03 — GDB Mastery for Embedded

### Basic GDB Workflow

```bash
# Compile with debug info
gcc -g -O0 -o myapp myapp.c

# Start GDB
gdb ./myapp

# GDB commands:
(gdb) run arg1 arg2         # run program
(gdb) break main            # breakpoint at function
(gdb) break file.c:42       # breakpoint at line 42
(gdb) break *0x400abc       # breakpoint at address
(gdb) info breakpoints      # list breakpoints
(gdb) delete 2              # delete breakpoint 2
(gdb) continue              # continue execution
(gdb) next                  # step over (C level)
(gdb) step                  # step into (C level)
(gdb) nexti                 # step over (instruction level)
(gdb) stepi                 # step into (instruction level)
(gdb) finish                # run until function returns
(gdb) until 100             # run until line 100
```

### GDB for Embedded (Cross-Debugging with QEMU)

```bash
# Terminal 1: Start QEMU with GDB server
qemu-system-aarch64 \
  -M virt -cpu cortex-a76 -m 2G \
  -kernel Image -initrd initramfs.cpio.gz \
  -append "console=ttyAMA0 nokaslr" \
  -nographic \
  -S -gdb tcp::1234
# -S: start CPU stopped (wait for GDB)
# -gdb tcp::1234: listen for GDB on port 1234

# Terminal 2: Connect GDB
gdb-multiarch vmlinux
(gdb) target remote :1234
(gdb) set architecture aarch64
(gdb) break start_kernel
(gdb) continue
(gdb) # Kernel will stop at start_kernel()

# Useful GDB commands for kernel debugging:
(gdb) lx-symbols              # load kernel module symbols (lx-gdb.py)
(gdb) lx-ps                   # list processes
(gdb) lx-dmesg                # print kernel log
(gdb) info registers          # all registers
(gdb) info registers pc sp x0  # specific registers

# ARM64 register display
(gdb) info registers pc
(gdb) x/10i $pc              # disassemble 10 instructions at PC
(gdb) x/20x $sp              # dump stack
(gdb) x/s 0xffffff8008abc123 # read string at kernel address
```

### GDB Print Techniques

```
(gdb) print variable          # print variable value
(gdb) print *ptr              # print what pointer points to
(gdb) print ptr->field        # print struct field
(gdb) print/x addr            # print in hex
(gdb) print/d addr            # print in decimal
(gdb) print/t val             # print in binary
(gdb) print/a addr            # print as address

# Kernel struct inspection
(gdb) print *(struct task_struct *)0xffffff8012345678
(gdb) print ((struct ufshcd_lrb *)0xffffff8009abc)->cmd->cmnd[0]

# Display (watch) — auto-print on each stop
(gdb) display i
(gdb) display *buf@10         # display 10 elements of array
(gdb) info display
(gdb) undisplay 1

# Memory examination
(gdb) x/10xb 0xffffff80...   # 10 bytes in hex
(gdb) x/10xw 0xffffff80...   # 10 words (4-byte) in hex
(gdb) x/10xg 0xffffff80...   # 10 giant (8-byte) in hex
(gdb) x/10i  0xffffff80...   # 10 instructions (disasm)
(gdb) x/10s  0xffffff80...   # 10 strings

# Watchpoints
(gdb) watch variable          # stop when variable is written
(gdb) rwatch variable         # stop when read
(gdb) awatch variable         # stop when read or written
```

### KGDB for In-Kernel Debugging

```bash
# Kernel config required:
# CONFIG_KGDB=y
# CONFIG_KGDB_SERIAL_CONSOLE=y
# CONFIG_DEBUG_INFO=y
# CONFIG_FRAME_POINTER=y

# Kernel command line:
kgdboc=ttyS0,115200 kgdbwait

# GDB connection from host:
gdb vmlinux
(gdb) target remote /dev/ttyUSB0
# OR if using QEMU:
(gdb) target remote :4321
```

---

## 04 — Kernel Debug Commands

### /proc and /sys Deep Dive

```bash
# Kernel messages
dmesg                           # all kernel messages
dmesg -w                        # watch (follow) kernel messages
dmesg -T                        # with human-readable timestamps
dmesg -l err,warn               # only errors and warnings
dmesg | grep -E "error|fail|BUG" # filter for problems
dmesg --clear                   # clear ring buffer

# Kernel version and build info
uname -a                        # kernel version + arch
cat /proc/version               # detailed build info
cat /proc/sys/kernel/osrelease  # release string

# Driver debugging
cat /sys/kernel/debug/dynamic_debug/control | grep ufshcd
echo "module ufshcd +p" > /sys/kernel/debug/dynamic_debug/control
# +p = enable pr_debug() messages for ufshcd module

# Probe debugging
udevadm test /sys/bus/platform/devices/fe3c0000.ufs
# Simulates what udev would do for a device

# I2C debugging
i2cdetect -y 0              # scan I2C bus 0
i2cdump -y 0 0x68           # dump all registers of device at 0x68
i2cget -y 0 0x68 0x00       # read register 0x00
i2cset -y 0 0x68 0x00 0x42  # write 0x42 to register 0x00

# GPIO debugging
gpiodetect                  # list all GPIO controllers
gpioinfo gpiochip0          # list all pins of gpiochip0
gpioget gpiochip0 5         # read GPIO line 5
gpioset gpiochip0 5=1       # set GPIO line 5 high

# Power management
cat /sys/kernel/debug/pm_debug/suspend_stats
cat /sys/kernel/debug/wakeup_sources
cat /sys/power/wakeup_count
echo mem > /sys/power/state  # suspend to RAM
```

### ftrace — The Kernel Function Tracer

```bash
# Mount debugfs (usually auto-mounted)
mount -t debugfs none /sys/kernel/debug

cd /sys/kernel/debug/tracing

# List available tracers
cat available_tracers
# nop blk mmiotrace function function_graph

# Function tracer — trace every kernel function call
echo function > current_tracer
echo 1 > tracing_on
cat trace | head -50
echo 0 > tracing_on

# Function graph — shows call tree with timing
echo function_graph > current_tracer
echo ufshcd_probe > set_graph_function     # only trace ufshcd_probe subtree
echo 1 > tracing_on
# ... trigger your driver operation ...
echo 0 > tracing_on
cat trace | head -100

# Filter by function name
echo ufshcd_* > set_ftrace_filter         # trace all ufshcd functions
echo > set_ftrace_filter                  # clear filter (trace all)

# Trace specific events
cat available_events | grep scsi
echo scsi:scsi_dispatch_cmd_start > set_event
echo 1 > tracing_on
# ... run SCSI command ...
echo 0 > tracing_on
cat trace

# Useful for UFS debugging:
echo ufs:* > set_event
echo 1 > tracing_on

# Ring buffer size (increase for longer traces)
echo 65536 > buffer_size_kb

# Clear trace buffer
echo > trace
```

### perf — Performance Analysis

```bash
# Install
sudo apt install linux-tools-$(uname -r)

# Count hardware events during a command
perf stat ls
perf stat -e cache-misses,cache-references,instructions,cycles ls

# Profile (sample) — find hot functions
perf record -g ./my_program        # with call graph
perf record -g -a sleep 10         # system-wide for 10 seconds
perf report                         # view interactive report
perf report --stdio                 # non-interactive

# Profile kernel
sudo perf record -g -a -e cycles:k sleep 5
sudo perf report

# flamegraphs (install FlameGraph)
git clone https://github.com/brendangregg/FlameGraph
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg

# Useful: profile UFS I/O
perf record -e block:block_rq_issue,block:block_rq_complete -ag sleep 10
perf report

# perf trace — lightweight strace alternative
perf trace ls
perf trace -p PID                   # trace running process
```

---

## 09 — Quick Reference Cheatsheet

```
══════════════════ KERNEL OOPS ANALYSIS ══════════════════

1. Get the oops output from dmesg
2. Find the faulting instruction address (PC value)
3. addr2line -e vmlinux -f 0xFAULT_ADDR
4. Or: scripts/decode_stacktrace.sh vmlinux < oops.txt

══════════════════ DRIVER NOT PROBING CHECKLIST ══════════

□ Is the driver module loaded? (lsmod)
□ Is the DT node present? (cat /proc/device-tree/PATH/compatible)
□ Does compatible string match? (grep in driver source)
□ Is the DT node enabled? (status = "okay" in DTS)
□ Check dmesg for probe errors
□ Enable dynamic debug: echo "module NAME +p" > dynamic_debug/control

══════════════════ GDB QUICK START ══════════════════════

# QEMU + kernel debug:
qemu-system-aarch64 [options] -S -gdb tcp::1234
gdb-multiarch vmlinux
> target remote :1234
> break start_kernel
> continue

══════════════════ FTRACE QUICK START ═══════════════════

cd /sys/kernel/debug/tracing
echo function > current_tracer
echo "my_function" > set_ftrace_filter
echo 1 > tracing_on
[do something]
echo 0 > tracing_on
cat trace

══════════════════ VIM QUICK REFERENCE ══════════════════

Navigation: gg/G=top/bot, Ctrl+d/u=half pg, /=search
Edit:       dd=del line, yy=yank, p=paste, u=undo
Tags:       Ctrl+]=jump def, Ctrl+t=back
Cscope:     :cs find s SYMBOL (find all uses)
Split:      :vsplit file, Ctrl+w h/j/k/l=navigate
```

---

*Next section: [27_AI_Dev_Environment/](../27_AI_Dev_Environment/)*  
*Related: [09_Debugging/](../09_Debugging/) | [10_Code_Browsing/](../10_Code_Browsing/)*
