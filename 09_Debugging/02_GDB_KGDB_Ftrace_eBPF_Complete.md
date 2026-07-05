# GDB, KGDB, ftrace, eBPF — Complete Debugging Tools Guide

> **The Debug Mindset:** A great debugger is not someone who knows all the answers. They know how to ask the right questions of the system. These tools help you ask those questions.

---

## Tool Selection Guide

```
Symptom → Tool to Use
─────────────────────────────────────────────────────────────────
User space crash (segfault)           → GDB
Driver crash / kernel oops            → crash + vmlinux
Driver hanging / deadlock             → lockdep + kdb
Which function is called most?        → ftrace function_graph
Where is CPU time spent?              → perf + flamegraph
Memory leak / out-of-bounds           → KASAN + KMEMLEAK
What syscalls does this app make?     → strace
What kernel functions does it call?   → ftrace, eBPF
Boot-time profiling                   → ftrace initcall_debug
Step through kernel code live         → KGDB
Network packet analysis               → eBPF + bpftrace
Lock contention / scheduling          → perf lock / perf sched
```

---

## Part 1: GDB — Debugging User Space from the Beginning

### Level 1: What GDB Is

GDB = GNU Debugger. It lets you **pause a running program** at any point, examine its state (variables, memory, call stack), and step through code line by line.

Think of it as a **time machine** for your program — you can pause it, look around, then let it continue.

### Level 2: Basic GDB Workflow

```bash
# Compile with debug symbols (-g flag is ESSENTIAL)
gcc -g -O0 -o my_program my_program.c
# -g: include debug symbols (variable names, line numbers)
# -O0: disable optimization (makes stepping predictable)

# Start debugging
gdb ./my_program

# Inside GDB:
(gdb) run                        # run the program
(gdb) run arg1 arg2              # run with arguments
(gdb) run < input.txt            # run with stdin redirect

# Set a breakpoint
(gdb) break main                 # break at function start
(gdb) break my_program.c:42     # break at line 42
(gdb) break write_register       # break at function name
(gdb) break *0x4005d0            # break at address

# Control execution
(gdb) continue    # (c) run until next breakpoint
(gdb) next        # (n) execute next line (step over function calls)
(gdb) step        # (s) step INTO function calls
(gdb) finish      # run until current function returns
(gdb) until 50    # run until line 50

# Examine state
(gdb) print variable             # print variable value
(gdb) print *ptr                 # dereference pointer
(gdb) print ptr->field           # struct field
(gdb) display counter            # auto-print counter after every step
(gdb) info locals                # all local variables
(gdb) info args                  # function arguments
(gdb) backtrace                  # (bt) call stack
(gdb) frame 2                    # switch to frame 2 in call stack
(gdb) list                       # show current source
(gdb) list 40,60                 # show lines 40-60

# Memory examination
(gdb) x/10x 0x601000             # 10 hex words at address
(gdb) x/s my_string              # print as string
(gdb) x/i $pc                    # disassemble instruction at PC
(gdb) info registers             # all CPU registers
```

### Level 3: GDB for Kernel/Driver Development (ARM64 + QEMU)

```bash
# Start QEMU with GDB stub enabled
qemu-system-aarch64 \
    -machine virt -cpu cortex-a57 -m 1G \
    -kernel arch/arm64/boot/Image \
    -nographic \
    -s \         # Open GDB server on :1234
    -S           # Pause at startup, wait for GDB

# In another terminal, start GDB
aarch64-linux-gnu-gdb vmlinux   # vmlinux has kernel debug symbols

# Connect to QEMU
(gdb) target remote :1234

# Set ARM64 architecture
(gdb) set architecture aarch64

# Set breakpoint at kernel start
(gdb) hbreak start_kernel         # hardware breakpoint
(gdb) continue

# When breakpoint hits:
(gdb) backtrace
(gdb) info registers
(gdb) p init_task.comm            # print init process name

# Debug a specific driver function
(gdb) hbreak counter_probe
(gdb) continue
# When driver probe is called:
(gdb) p *pdev                     # print platform_device struct
(gdb) p pdev->name                # device name
(gdb) next                        # step through probe()
```

### Level 4: Debugging a Crash Dump (Most Common Real-World Use)

```bash
# After kernel crash, collect vmcore
# (Configure CONFIG_KDUMP + kdump service)

# Or use /proc/kcore (live kernel core)
sudo crash vmlinux /proc/kcore

# Inside crash:
crash> bt                     # backtrace of current CPU
crash> bt -a                  # backtrace of ALL CPUs
crash> log                    # kernel message buffer (dmesg)
crash> ps                     # all processes at crash time
crash> ps | grep my_driver    # find your driver's threads
crash> files <pid>            # open files of a process
crash> vm <pid>               # virtual memory of a process

# Analyze the crashing task
crash> bt <pid>               # backtrace of specific task
crash> struct task_struct <addr>  # dump task_struct
crash> rd <addr> 16           # read 16 words from address

# Analyze a specific struct
crash> struct platform_device <addr>
crash> struct counter_priv <addr>

# Find where in the code the crash happened
crash> dis ufshcd_abort       # disassemble function
crash> sym <addr>             # resolve crash address to symbol+offset

# Examine kernel messages around crash
crash> dmesg | grep -i "error\|panic\|BUG"
```

---

## Part 2: KGDB — Live Kernel Debugging

### What is KGDB?

KGDB (Kernel GNU Debugger) lets you use GDB to debug the **live running kernel** — set breakpoints inside kernel functions, step through kernel code, examine kernel data structures.

This is how you debug a driver when it fails intermittently and you can't reproduce it with printk.

### Setup KGDB over UART (Most Common for Embedded)

```bash
# Step 1: Add to kernel config
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_KGDB_KDB=y              # optional: keyboard debugger
CONFIG_FRAME_POINTER=y         # better backtraces

# Step 2: Add to kernel command line (bootargs in DT or U-Boot)
kgdboc=ttyS0,115200 kgdbwait
# kgdbwait = pause kernel at boot and wait for GDB connection
# kgdboc = KGDB over console, using ttyS0 at 115200 baud

# Step 3: Connect GDB from host
# Board's UART0 connected to host /dev/ttyUSB0
aarch64-linux-gnu-gdb vmlinux
(gdb) target remote /dev/ttyUSB0   # serial connection

# Step 4: GDB can now control the kernel!
(gdb) hbreak counter_probe
(gdb) continue
# → Driver probe runs → GDB halts → you can inspect state
```

### KGDB Magic SysRq (Trigger Without Reboot)

```bash
# On the running board, trigger KGDB entry:
echo g > /proc/sysrq-trigger     # enter KGDB immediately

# Or: send SysRq via serial (break + g)
# In minicom/tio: Ctrl+A, then send "SysRq-g"
```

### KDB — Keyboard Debugger (No GDB Needed)

KDB is simpler — it works from the serial console directly, no second machine needed:

```bash
# Boot with: kgdboc=ttyS0,115200 kgdbwait
# OR trigger live: echo g > /proc/sysrq-trigger

# KDB prompt appears:
kdb> help                 # show all commands
kdb> bt                   # backtrace
kdb> ps                   # process list
kdb> md <addr>            # memory dump at address
kdb> rd <register>        # read CPU register
kdb> go                   # continue kernel execution
kdb> bp counter_probe     # set breakpoint
kdb> bl                   # list breakpoints
kdb> bc 1                 # clear breakpoint 1
kdb> ss                   # single step
kdb> lsmod                # list loaded modules
kdb> dmesg                # kernel log
```

---

## Part 3: ftrace — Kernel Function Tracer

### What ftrace Can Do

ftrace lets you trace **which kernel functions are called**, in what order, and how long they take — without modifying the kernel or rebooting.

```bash
# ftrace is in /sys/kernel/debug/tracing/ (mount debugfs first)
sudo mount -t debugfs debugfs /sys/kernel/debug/

cd /sys/kernel/debug/tracing/

# List available tracers
cat available_tracers
# blk  function_graph  wakeup  wakeup_rt  function  nop
```

### Tracer 1: function — Which Functions Are Called?

```bash
cd /sys/kernel/debug/tracing/

# Enable function tracer
echo function > current_tracer

# Filter to only your driver
echo 'counter_*' > set_ftrace_filter     # trace all counter_* functions
# OR specific functions:
echo 'counter_probe counter_irq_handler' > set_ftrace_filter

# Start tracing
echo 1 > tracing_on

# Do something that triggers your driver
cat /sys/devices/.../count    # trigger sysfs read

# Stop tracing
echo 0 > tracing_on

# Read results
cat trace
# tracer: function
# TASK-PID  CPU#  TIMESTAMP   FUNCTION
# bash-1234 [000] 12345.678:  counter_probe <-platform_drv_probe
# bash-1234 [000] 12345.679:  counter_irq_handler <-__handle_irq_event_percpu
```

### Tracer 2: function_graph — Call Graph with Timing

```bash
echo function_graph > current_tracer
echo 'counter_probe' > set_graph_function   # only trace inside counter_probe

echo 1 > tracing_on
# trigger probe (insmod driver)
echo 0 > tracing_on

cat trace
# function_graph output — shows call tree with timing:
#  0)               |  counter_probe() {
#  0)   0.234 us    |    devm_kzalloc();
#  0)               |    devm_platform_ioremap_resource() {
#  0)   1.456 us    |      __devm_ioremap_resource();
#  0)               |    }
#  0)               |    devm_clk_get() {
#  0)   0.789 us    |      of_clk_get_by_name();
#  0)               |    }
#  0)  12.345 us    |  }   ← total probe() time: 12.345 µs
```

### Tracer 3: events — Specific Kernel Events

```bash
# List all available events
ls events/
# block  drm  ext4  irq  kvm  napi  net  pagefault  sched  ...

# Enable IRQ events
echo 1 > events/irq/irq_handler_entry/enable
echo 1 > events/irq/irq_handler_exit/enable

# Enable scheduling events (who ran when)
echo 1 > events/sched/sched_switch/enable

# Filter by your driver name
echo 'name == "my-counter"' > events/irq/irq_handler_entry/filter

echo 1 > tracing_on
# ... do work ...
echo 0 > tracing_on
cat trace
```

### trace-cmd — Easier ftrace Interface

```bash
sudo apt install trace-cmd

# Record function graph for counter driver
sudo trace-cmd record -p function_graph \
    -g counter_probe \
    -- modprobe counter_driver

# Generate report
sudo trace-cmd report | head -50

# Record IRQ events for 5 seconds
sudo trace-cmd record -e irq:irq_handler_entry -e irq:irq_handler_exit -- sleep 5
sudo trace-cmd report | grep "my-counter"

# Visualize in KernelShark GUI
sudo apt install kernelshark
kernelshark trace.dat
```

---

## Part 4: eBPF — Modern Kernel Observability

### What is eBPF? (Beginner)

eBPF (extended Berkeley Packet Filter) lets you run **small programs inside the kernel** without modifying kernel code or loading a kernel module.

Traditional approach: Add printk → recompile → reboot → test → repeat  
eBPF approach: Write 20-line script → inject into running kernel → get results immediately

### bpftrace — The Easiest eBPF Tool

```bash
sudo apt install bpftrace

# Trace every time kmalloc is called
sudo bpftrace -e 'kprobe:kmalloc { printf("kmalloc size=%d\n", arg0); }'

# Trace calls to your driver's read function
sudo bpftrace -e 'kprobe:counter_probe { 
    printf("counter_probe called by PID %d (%s)\n", pid, comm); 
}'

# Count function calls (like a histogram)
sudo bpftrace -e 'kprobe:kmalloc { @[comm] = count(); } 
                  END { print(@); }' &
sleep 5
kill %1

# Measure function latency
sudo bpftrace -e '
kprobe:counter_probe { @start[tid] = nsecs; }
kretprobe:counter_probe /@start[tid]/ {
    @latency = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}
END { print(@latency); }'
```

### BCC Tools — Pre-Built eBPF Scripts

```bash
sudo apt install bpfcc-tools python3-bpfcc

# opensnoop — trace all open() calls
sudo opensnoop-bpfcc

# execsnoop — trace all exec() calls
sudo execsnoop-bpfcc

# ext4slower — trace slow ext4 operations
sudo ext4slower-bpfcc 10    # operations > 10ms

# biolatency — block I/O latency histogram
sudo biolatency-bpfcc -d sda -T 10   # 10 second histogram for sda

# tcpconnect — trace TCP connections
sudo tcpconnect-bpfcc

# funclatency — measure any kernel function latency
sudo funclatency-bpfcc counter_probe
# Tracing counter_probe()... Ctrl+C to end.
#    value           : count   distribution
#        0 -> 1      : 0      |                  |
#        2 -> 3      : 0      |                  |
#        4 -> 7      : 1      |**                |
#        8 -> 15     : 5      |**************    |
#       16 -> 31     : 3      |*******           |

# profile — CPU profiling (like perf but eBPF)
sudo profile-bpfcc -af 30    # 30 seconds, user+kernel stacks
```

### Write Your Own eBPF Program (Python + BCC)

```python
#!/usr/bin/env python3
# trace_driver_io.py — trace read/write to your driver

from bcc import BPF

# eBPF program (runs inside kernel)
bpf_program = """
#include <uapi/linux/ptrace.h>
#include <linux/fs.h>

struct data_t {
    u32 pid;
    u64 count;
    char comm[16];
    char type[8];
};

BPF_PERF_OUTPUT(events);

int trace_read(struct pt_regs *ctx, struct file *file,
               char __user *buf, size_t count, loff_t *ppos) {
    struct data_t data = {};
    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.count = count;
    bpf_get_current_comm(&data.comm, sizeof(data.comm));
    __builtin_memcpy(&data.type, "READ", 5);
    events.perf_submit(ctx, &data, sizeof(data));
    return 0;
}

int trace_write(struct pt_regs *ctx, struct file *file,
                const char __user *buf, size_t count, loff_t *ppos) {
    struct data_t data = {};
    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.count = count;
    bpf_get_current_comm(&data.comm, sizeof(data.comm));
    __builtin_memcpy(&data.type, "WRITE", 6);
    events.perf_submit(ctx, &data, sizeof(data));
    return 0;
}
"""

b = BPF(text=bpf_program)

# Attach to chardev_read and chardev_write functions
b.attach_kprobe(event="chardev_read",  fn_name="trace_read")
b.attach_kprobe(event="chardev_write", fn_name="trace_write")

def print_event(cpu, data, size):
    event = b["events"].event(data)
    print(f"PID={event.pid} COMM={event.comm.decode()} "
          f"TYPE={event.type.decode()} COUNT={event.count}")

b["events"].open_perf_buffer(print_event)

print("Tracing /dev/mydevice access... Ctrl+C to stop")
while True:
    b.perf_buffer_poll()
```

```bash
sudo python3 trace_driver_io.py
# Now in another terminal:
echo "test" > /dev/mydevice
cat /dev/mydevice
# Output:
# PID=1234 COMM=bash TYPE=WRITE COUNT=5
# PID=1234 COMM=cat  TYPE=READ  COUNT=5
```

---

## Part 5: perf — Performance Analysis

```bash
# CPU performance counter statistics
sudo perf stat ./my_program
# Performance counter stats for './my_program':
#    1,234,567      cycles                 #  1.234 GHz
#      456,789      instructions           #  0.37  insn per cycle
#       12,345      cache-misses           #  2.34% of all cache refs
#       98,765      branch-misses          #  1.23% of all branches

# Profile: where is CPU time spent?
sudo perf record -g ./my_program
sudo perf report --stdio | head -30

# Profile the kernel (while running your driver)
sudo perf record -ag -e cycles -c 1000 -- sleep 10
sudo perf report --stdio | grep -A3 "counter_"

# System-wide call frequency
sudo perf top -ag            # live interactive view

# Specific hardware events
sudo perf stat -e cache-misses,cache-references,instructions \
    --repeat 5 \
    ./my_benchmark

# Generate flamegraph
sudo perf record -ag -F 99 -- sleep 30
sudo perf script | \
    ~/FlameGraph/stackcollapse-perf.pl | \
    ~/FlameGraph/flamegraph.pl > flame.svg
```

---

## Part 6: KASAN + UBSAN — Catch Memory Bugs Automatically

### Enable in Kernel Config

```bash
# These make bugs CRASH IMMEDIATELY with a detailed message
# instead of silent corruption that manifests later

CONFIG_KASAN=y                    # Kernel Address SANitizer
CONFIG_KASAN_INLINE=y             # inline instrumentation (faster)
CONFIG_UBSAN=y                    # Undefined Behavior SANitizer
CONFIG_UBSAN_SANITIZE_ALL=y
CONFIG_DEBUG_LIST=y               # detect list corruption
CONFIG_DEBUG_SG=y                 # detect scatter-gather corruption
```

### KASAN in Action

```c
/* This driver code has a bug: use-after-free */
static int __init buggy_init(void)
{
    char *buf = kmalloc(64, GFP_KERNEL);
    kfree(buf);
    buf[0] = 'x';   /* BUG: use after free! */
    return 0;
}
```

```
# KASAN output (catches it immediately):
BUG: KASAN: use-after-free in buggy_init+0x3c/0x50
Write of size 1 at addr ffff888012345678 by task insmod/1234

CPU: 0 PID: 1234 Comm: insmod
Hardware name: QEMU ARM64 virt machine
Call trace:
 dump_stack
 print_address_description
 kasan_report
 buggy_init
 do_one_initcall
 ...

Allocated by task 1234:
 kmalloc
 buggy_init
 ...

Freed by task 1234:
 kfree
 buggy_init
 ...
```

KASAN tells you **exactly**:
- What kind of bug (use-after-free, out-of-bounds)
- The address accessed
- Where it was allocated (with stack trace)
- Where it was freed (with stack trace)
- Where the bad access happened (with stack trace)

---

## Part 7: lockdep — Find Deadlocks Before They Happen

```bash
CONFIG_DEBUG_LOCKDEP=y
CONFIG_PROVE_LOCKING=y
CONFIG_LOCK_STAT=y
```

```c
/* lockdep catches this potential deadlock at first occurrence: */
void func_A(void) {
    spin_lock(&lock_X);
    spin_lock(&lock_Y);   /* lock order: X → Y */
    spin_unlock(&lock_Y);
    spin_unlock(&lock_X);
}

void func_B(void) {
    spin_lock(&lock_Y);
    spin_lock(&lock_X);   /* lock order: Y → X ← OPPOSITE! */
    spin_unlock(&lock_X);
    spin_unlock(&lock_Y);
}
/* If CPU0 runs func_A and CPU1 runs func_B simultaneously: DEADLOCK */
```

```
# lockdep output (caught before deadlock actually happens):
WARNING: possible circular locking dependency detected
insmod/1234 is trying to acquire lock:
  (&lock_X) at func_B+0x28

but task is already holding lock:
  (&lock_Y) at func_B+0x10

which lock already depends on the new lock.
The existing dependency chain:
  lock_X → lock_Y (from func_A)

the cycle would be:
  lock_Y → lock_X → lock_Y
```

---

## Interview Questions — Debugging

| Level | Question |
|-------|----------|
| **Beginner** | How do you see kernel messages? What is dmesg? |
| **Beginner** | What is a kernel oops? What information does it contain? |
| **Intermediate** | How would you debug a driver that fails to probe? What do you check first? |
| **Intermediate** | What is ftrace and how do you enable it? Give a concrete example. |
| **Advanced** | Explain KASAN. What kinds of bugs does it catch? How does it work internally? |
| **Advanced** | What is lockdep? How does it detect a potential deadlock without it actually occurring? |
| **Advanced** | How do you use GDB to debug the Linux kernel over QEMU? What flags does QEMU need? |
| **Expert** | Write a bpftrace one-liner to measure the latency distribution of all VFS read() calls. |
| **Expert** | How would you debug a kernel crash that only happens once per week on production hardware? |
