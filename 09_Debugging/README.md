# 09 — Debugging

> "Debugging is the detective work of software engineering. A great debugger finds bugs that others can't even locate." — Master these tools and you'll never be stuck.

---

## Section Structure

```
09_Debugging/
├── 01_Debug_Mindset.md               ← Scientific method for debugging
├── 02_UART_Debug_Workflow.md         ← Serial console, early printk, kgdb
├── 03_Kernel_Oops_Analysis.md        ← Reading crash dumps, addr2line
├── 04_GDB_And_KGDB.md                ← Step-through kernel debugging
├── 05_Ftrace_Guide.md                ← Function tracer, event tracer
├── 06_eBPF_For_Embedded.md           ← BCC, bpftrace for kernel tracing
├── 07_Perf_And_Flamegraphs.md        ← Performance profiling
├── 08_KASAN_UBSAN_Lockdep.md         ← Memory and lock bug detection
├── 09_Crash_Tool.md                  ← Post-mortem analysis with crash
├── 10_JTAG_Debug.md                  ← Hardware debugging with probe
├── 11_Qualcomm_Debug_Tools.md        ← QXDM, CoreSight, ramdump
└── 12_Debug_Scenarios/               ← 15 real-world debug exercises
    ├── Scenario_01_NULL_Dereference.md
    ├── Scenario_02_Boot_Hang.md
    ├── Scenario_03_Driver_Not_Probing.md
    ├── Scenario_04_IRQ_Storm.md
    ├── Scenario_05_Memory_Corruption.md
    ├── Scenario_06_Deadlock.md
    ├── Scenario_07_DMA_Fault.md
    ├── Scenario_08_Kernel_Panic_UFS.md
    └── ...
```

---

## The Debug Mindset

```
The Scientific Method for Debugging:
─────────────────────────────────────
1. OBSERVE:   What exactly happens? What doesn't happen?
              → Get the exact error message, oops, log
              → Note: on which hardware, kernel version, build config

2. HYPOTHESIZE: What could cause this?
              → List 3–5 possible root causes (don't fixate on one)
              → Rank by likelihood

3. EXPERIMENT: Design a test to confirm/deny each hypothesis
              → Change ONE thing at a time
              → Every change is a controlled experiment

4. ANALYZE:   What did the test reveal?
              → Update your hypotheses
              
5. CONCLUDE:  Once one hypothesis is confirmed by evidence, fix it
              → Explain WHY the fix works
              → Document to prevent recurrence

Common traps:
  ✗ Changing multiple things at once (you won't know what fixed it)
  ✗ Assuming you know the root cause without evidence
  ✗ Stopping at "workaround" without understanding root cause
  ✗ Not writing down what you've tried
```

---

## Kernel Oops Analysis

### Reading an Oops

```
A kernel oops (not panic) means a recoverable fault:
─────────────────────────────────────────────────────

[  123.456789] BUG: unable to handle page fault for address: 0000000000000038
[  123.456791] #PF: supervisor write access in kernel mode
[  123.456792] #PF: error_code(0x0002) - not-present page
[  123.456793] PGD 0 P4D 0                          ← page table entries (all zero = NULL)
[  123.456794] Oops: 0002 [#1] PREEMPT SMP
[  123.456795] Modules linked in: hello_char ufshcd_core [...]
[  123.456796] CPU: 2 PID: 1234 Comm: kworker/2:1 Not tainted 6.6.30 #1
[  123.456797] Hardware name: QEMU QEMU Virtual Machine/QEMU ...
[  123.456798] pstate: 60400009 (nZCv daif +PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[  123.456799] pc : ufshcd_send_command+0x88/0x200          ← WHERE it crashed
[  123.456800] lr : ufshcd_queuecommand+0x1ac/0x380         ← called from
[  123.456801] sp : ffffffc012345678
[  123.456802] x0 : ffffff88012abc00    x1 : 0000000000000038   ← x1 = NULL+0x38!
[  123.456803] ...
[  123.456820] Call trace:
[  123.456821]  ufshcd_send_command+0x88/0x200               ← top of stack
[  123.456822]  ufshcd_queuecommand+0x1ac/0x380
[  123.456823]  scsi_dispatch_cmd+0x64/0x180
[  123.456824]  scsi_queue_rq+0x234/0x400
[  123.456825]  blk_mq_dispatch_rq_list+0x18c/0x600
```

**Analysis:**
1. `address: 0000000000000038` → accessing NULL + 0x38 → NULL pointer dereference
2. `pc : ufshcd_send_command+0x88` → fault is inside `ufshcd_send_command`, 0x88 bytes from start
3. `x1 : 0000000000000038` → register x1 was 0x38 (not NULL itself, but accessing struct at NULL)

**Next step:** `addr2line -e vmlinux -f $(grep ufshcd_send_command /proc/kallsyms | head -1 | awk '{print $1}' | xargs printf '%#x')`

### Decoding with addr2line

```bash
# Method 1: addr2line with vmlinux
addr2line -e vmlinux -f ffffffc0xxxxxxxx

# Method 2: scripts/decode_stacktrace.sh (preferred)
cat oops.txt | ./scripts/decode_stacktrace.sh vmlinux

# Method 3: From running system
echo "ffffffc0xxxxxxxx" | scripts/faddr2line vmlinux ufshcd_send_command+0x88

# Method 4: objdump for context
objdump -d vmlinux | grep -A20 "<ufshcd_send_command>"
```

---

## ftrace — Practical Usage

```bash
# Setup
cd /sys/kernel/debug/tracing

# === FUNCTION TRACER === 
echo function > current_tracer
echo ufshcd_probe > set_ftrace_filter    # only trace this function
echo 1 > tracing_on
# ... trigger probe ...
echo 0 > tracing_on
cat trace
# Output:
# kworker/0:2-456  [000] ....  10.123456: ufshcd_probe <-platform_drv_probe

# === FUNCTION GRAPH (with timing) ===
echo function_graph > current_tracer
echo ufshcd_probe > set_graph_function
echo 1 > tracing_on
# ... trigger probe ...
echo 0 > tracing_on
cat trace
# Output shows call tree:
# 0)               |  ufshcd_probe() {
# 0)   1.234 us    |    devm_kzalloc();
# 0) + 15.678 us   |    devm_ioremap_resource();
# 0) ! 285.123 ms  |    ufshcd_init();   ← ! = > 100ms

# === TRACE EVENTS ===
# Find UFS-related events
cat available_events | grep -i ufs
# Find block I/O events
cat available_events | grep block

# Enable specific events
echo block:block_rq_issue > set_event
echo block:block_rq_complete >> set_event
echo 1 > tracing_on
# ... run some I/O ...
echo 0 > tracing_on
cat trace

# === TRACE MARKERS (add your own) ===
echo "my_event_start" > trace_marker
# ... your operation ...
echo "my_event_end" > trace_marker
# Now trace shows timing between your markers

# === KPROBE EVENTS (trace any kernel function) ===
# Add probe at ufshcd_send_command, capture arg
echo 'p:myprobe ufshcd_send_command hba=%x0 lrb=%x1' > kprobe_events
echo 'myprobe' > set_event
echo 1 > tracing_on
# ... run I/O ...
cat trace | grep myprobe
```

---

## KASAN: Memory Bug Detection

```bash
# Kernel config (enable in .config):
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y  # or KASAN_SW_TAGS on ARM64 for lower overhead
CONFIG_KASAN_INLINE=y
CONFIG_KASAN_STACK=y

# KASAN catches:
# 1. Use-after-free
# 2. Out-of-bounds access
# 3. Stack-based buffer overflow
# 4. Global variable overflow
# 5. Use of uninitialized memory (KMSAN — separate tool)

# Example bug KASAN catches:
static int buggy_driver_probe(struct platform_device *pdev)
{
    char *buf = kmalloc(16, GFP_KERNEL);
    buf[20] = 'X';  // ← KASAN: OOB write at offset 20, size 16
    kfree(buf);
    buf[0] = 'Y';   // ← KASAN: use-after-free
    return 0;
}

# KASAN output looks like:
# ==================================================================
# BUG: KASAN: slab-out-of-bounds in buggy_driver_probe+0x48/0x80
# Write of size 1 at addr ffff888012345600 by task kworker/0:1
#
# Allocated by task 123:
#  kmalloc called at buggy_driver_probe+0x24
# ==================================================================
```

---

## Lockdep: Deadlock Detection

```bash
# Enable:
CONFIG_LOCKDEP=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_LOCKDEP=y
CONFIG_LOCK_STAT=y  # performance stats

# Lockdep tracks lock acquisition ordering.
# If ever lock A is acquired while holding lock B,
# and lock B is acquired while holding lock A → DEADLOCK WARNING

# Common deadlock pattern lockdep catches:
static DEFINE_MUTEX(lock_a);
static DEFINE_MUTEX(lock_b);

void thread1(void) {
    mutex_lock(&lock_a);  // acquires A
    mutex_lock(&lock_b);  // then acquires B
    mutex_unlock(&lock_b);
    mutex_unlock(&lock_a);
}

void thread2(void) {
    mutex_lock(&lock_b);  // acquires B
    mutex_lock(&lock_a);  // then tries to acquire A  ← DEADLOCK if thread1 is running
    mutex_unlock(&lock_a);
    mutex_unlock(&lock_b);
}

# Lockdep output:
# ======================================================
# WARNING: possible circular locking dependency detected
# ......
# thread2 is trying to acquire lock:
#   lock_a at thread2+0x48
# 
# but task is already holding lock:
#   lock_b at thread2+0x24
# 
# which lock already depends on the new lock.
# chain: lock_a → lock_b → lock_a  ← cycle!
```

---

## Debug Scenarios (Sample)

### Scenario: Driver Doesn't Probe

```
Symptoms:
- Device node present in DT
- Module loaded
- No dev_info() from probe()
- No /dev entry, no sysfs entry

Debug steps:
1. Check DT node:
   cat /proc/device-tree/[path-to-your-node]/compatible
   # Compare with driver's of_device_id .compatible string

2. Check DT status:
   cat /proc/device-tree/[path-to-your-node]/status
   # Must be: "okay"  (if missing, defaults to disabled in some drivers)

3. Check driver is loaded:
   lsmod | grep your_driver
   
4. Check platform bus:
   ls /sys/bus/platform/devices/ | grep [your-reg-address]
   # If device not listed → DT parsing issue
   
5. Check driver bind:
   ls /sys/bus/platform/drivers/your_driver/
   # If driver listed but not bound to device → compatible mismatch
   
6. Force bind (for testing):
   echo "your-device-address" > /sys/bus/platform/drivers/your_driver/bind

7. Enable dynamic debug:
   echo "module your_driver +p" > /sys/kernel/debug/dynamic_debug/control
   dmesg -w  # watch for probe debug output

Root cause 90% of the time:
   - compatible string mismatch (typo, case sensitivity)
   - status != "okay"
   - Missing required DT property → probe returns error before dev_info
```

### Scenario: Kernel Panic on Boot

```
Symptoms:
- Boot hangs with "Kernel panic - not syncing: ..."
- System appears stuck (no UART output after panic message)

Common causes:
1. "VFS: Unable to mount root fs" → wrong root= in bootargs, or rootfs corrupt
2. "attempted to kill init!" → init crashed on first instruction (corrupt rootfs)  
3. "Oops: ... in init process" → kernel module/driver crash before init runs
4. "not syncing: no working init found!" → init binary missing or wrong path

Debug steps:
1. Add to kernel cmdline: debug initcall_debug
   → Shows every initcall, you'll see the last successful one

2. Add: ignore_loglevel
   → Shows all dmesg output (some get suppressed during panic)

3. For rootfs issues:
   root=/dev/mmcblk0p2    → check partition number
   rootfstype=ext4        → explicit fstype
   rootwait               → wait for storage to appear
   init=/bin/sh           → bypass init, drop to shell

4. For module crash:
   module_blacklist=bad_module  → skip specific module
   initrd=minimal_initrd.gz     → use minimal initrd without bad driver
```

---

*Related: [06_Linux_Kernel/](../06_Linux_Kernel/) | [26_Developer_Commands/](../26_Developer_Commands/) | [32_Complete_System_Design/04_Complete_Boot_Flow_Visualization/02_Boot_Stage_Debug_Guide.md](../32_Complete_System_Design/04_Complete_Boot_Flow_Visualization/02_Boot_Stage_Debug_Guide.md)*
