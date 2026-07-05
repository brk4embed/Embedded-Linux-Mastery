# Linux Kernel Internals — Deep Dive

> **Beginner note:** The Linux kernel is the most complex piece of software on Earth with 30+ million lines of code. You do NOT need to understand all of it. You need to understand the 20% that covers 80% of real embedded work. This file covers exactly that 20%.

---

## 1. Process Management — What Happens When You Run a Program

### Level 1: Beginner — What is a Process?

When you type `ls` in a terminal, the shell creates a new **process** — an independent running instance of the `ls` program.

```
You type: ls
    ↓
Shell calls fork() — creates a copy of itself
    ↓
Child process calls exec("ls") — replaces itself with ls program
    ↓
ls program runs, prints output
    ↓
ls calls exit() — process terminates
    ↓
Shell calls wait() — collects exit status
```

Every process in Linux has:
- A unique **PID** (Process ID: 1, 2, 3...)
- Its own **virtual address space** (private memory)
- An entry in the kernel's **task_struct** (the kernel's record for this process)
- A **state** (Running, Sleeping, Zombie, Stopped)

### Level 2: How fork() and exec() Work Internally

```c
/* What happens inside the kernel when you call fork(): */

/* 1. Allocate new task_struct */
struct task_struct *child = dup_task_struct(current);

/* 2. Copy parent's memory map (using Copy-On-Write) */
copy_mm(clone_flags, child);
/* COW: child shares parent's pages until either writes to them */
/* When child writes: kernel allocates new page, copies content */
/* Why: fork() is fast even with 1GB parent — no actual copy yet */

/* 3. Copy file descriptors */
copy_files(clone_flags, child);

/* 4. Assign new PID */
child->pid = alloc_pid(child->nsproxy->pid_ns_for_children);

/* 5. Put child on run queue */
wake_up_new_task(child);
```

**Copy-On-Write (COW) — The Smart Trick:**
```
Parent memory: [page A: code] [page B: data] [page C: stack]
                     ↓              ↓              ↓
Child memory:  [SHARED A]    [SHARED B]    [SHARED C]

Child writes to page B:
  1. Page fault triggered
  2. Kernel: "This is a COW page"
  3. Allocate new page B' for child
  4. Copy contents of B into B'
  5. Child now has its own B'

Result: fork() takes microseconds regardless of address space size
```

### Level 3: task_struct — The Kernel's Process Record

`task_struct` is the most important struct in the Linux kernel. It contains EVERYTHING the kernel needs to know about a process:

```c
/* include/linux/sched.h (simplified — real struct has 800+ fields!) */
struct task_struct {
    /* State */
    unsigned int        __state;    /* TASK_RUNNING, TASK_INTERRUPTIBLE, etc. */
    
    /* Identity */
    pid_t               pid;        /* Process ID */
    pid_t               tgid;       /* Thread Group ID (= PID of main thread) */
    char                comm[TASK_COMM_LEN]; /* command name: "kworker/0:1" */
    
    /* Scheduling */
    int                 prio;       /* effective priority */
    int                 static_prio;/* set by nice() */
    unsigned int        policy;     /* SCHED_NORMAL, SCHED_FIFO, SCHED_RR */
    struct sched_entity se;         /* CFS scheduler entity */
    
    /* Memory */
    struct mm_struct    *mm;        /* virtual address space (NULL for kernel threads) */
    struct mm_struct    *active_mm; /* currently active mm */
    
    /* Files */
    struct files_struct *files;     /* open file descriptors */
    struct fs_struct    *fs;        /* filesystem info (cwd, root) */
    
    /* Signals */
    struct signal_struct    *signal;
    struct sighand_struct   *sighand;
    sigset_t                blocked; /* blocked signals */
    
    /* Relationships */
    struct task_struct  *parent;    /* parent process */
    struct list_head    children;   /* list of children */
    struct list_head    sibling;    /* next sibling in parent's children */
    
    /* CPU */
    int                 on_cpu;     /* which CPU is running this task */
    int                 cpu;        /* preferred CPU */
    
    /* Timing */
    u64                 utime;      /* time spent in user space */
    u64                 stime;      /* time spent in kernel space */
    
    /* ... 800 more fields ... */
};
```

```bash
# Inspect a real task_struct using crash tool or /proc:
cat /proc/1/status       # systemd (PID 1) details
cat /proc/self/status    # your current process
cat /proc/self/maps      # memory map of current process
ls /proc/self/fd/        # open file descriptors
```

---

## 2. The CFS Scheduler — How Linux Decides What Runs Next

### Level 1: The Problem to Solve

You have 100 processes wanting to run. You have 4 CPU cores. How do you decide:
- Which process runs on which core?
- For how long?
- How to be fair?
- How to give priority to real-time tasks?

### Level 2: CFS — Completely Fair Scheduler

CFS (the default Linux scheduler since 2.6.23) uses a concept called **virtual runtime (vruntime)**:

```
Core idea: Every process accumulates "time spent running" = vruntime
           The process with the LOWEST vruntime gets to run next
           (= the most deprived process gets CPU first)

Nice value:
  -20 = highest priority (vruntime accumulates SLOWER — gets more CPU)
   0  = normal priority
  +19 = lowest priority (vruntime accumulates FASTER — gets less CPU)
```

```
Timeline (2 processes: A at nice=0, B at nice=5):

Time 0ms:  A.vruntime = 0,  B.vruntime = 0
           A runs (lower vruntime, but both equal → A picked)
Time 4ms:  A.vruntime = 4,  B.vruntime = 0
           B runs (B has lower vruntime)
Time 6ms:  A.vruntime = 4,  B.vruntime = 2
           A runs (lower vruntime)
...
Result: A gets ~60% CPU time, B gets ~40%
        (because nice=5 makes vruntime accumulate 1.5× faster)
```

### Level 3: The Red-Black Tree (CFS Data Structure)

CFS stores all runnable processes in a **red-black tree** keyed by vruntime:

```
         [vruntime=10]
        /              \
 [vruntime=5]    [vruntime=15]
  /        \
[vt=2]   [vt=7]

Leftmost node (lowest vruntime) = next to run
→ O(log n) insert/remove, O(1) find-minimum
```

```c
/* Kernel scheduler key functions */
pick_next_task_fair()    /* choose the process with lowest vruntime */
enqueue_entity()         /* add process to RB tree when it becomes runnable */
dequeue_entity()         /* remove from RB tree when process blocks or exits */
update_curr()            /* update vruntime of currently running process */
```

### Level 4: Real-Time Scheduling Classes

```
Scheduling Policy    Kernel Class     Who Uses It
─────────────────    ────────────     ───────────────────────────────
SCHED_FIFO          rt_sched_class   Real-time: run until block/yield
SCHED_RR            rt_sched_class   Real-time: round-robin with timeslice
SCHED_DEADLINE      dl_sched_class   Hard real-time with CBS algorithm
SCHED_NORMAL/OTHER  fair_sched_class All normal processes (CFS)
SCHED_IDLE          idle_sched_class Only when nothing else runs
```

```c
/* Set real-time priority for a task (requires CAP_SYS_NICE or root) */
struct sched_param sp = { .sched_priority = 50 };  /* 1-99 for RT */
sched_setscheduler(0, SCHED_FIFO, &sp);

/* Check from within driver/kernel code */
if (current->policy == SCHED_FIFO)
    pr_info("This task is real-time!\n");
```

```bash
# Check current scheduling policy
chrt -p $$                   # current shell's scheduling

# Set process to SCHED_FIFO priority 50
chrt -f 50 ./my_realtime_app

# See all runnable processes sorted by priority
ps -eo pid,comm,cls,pri,ni,pcpu --sort=-pri | head -20
```

---

## 3. Memory Management — How Linux Manages RAM

### Level 1: The Problem — Everyone Wants Memory

Your system has 4GB of RAM. You have 100 processes, each thinking they have the whole machine to themselves. The kernel's job: make this illusion work safely.

### Level 2: Virtual Address Space

Every process sees a **virtual** address space (not real physical RAM):

```
ARM64 Virtual Address Space (48-bit, 256TB each half):

0xFFFF_FFFF_FFFF_FFFF  ──── Kernel space top
0xFFFF_0000_0000_0000  ──── Kernel space start
                            (hole: non-canonical addresses)
0x0000_FFFF_FFFF_FFFF  ──── User space top
    ↑ stack grows DOWN from here
    ...
    ↑ mmap area (shared libs, anonymous maps)
    ...
    ↑ heap grows UP (brk() syscall)
    ↑ BSS (zero-initialized global variables)
    ↑ data (initialized global variables)
    ↑ text (executable code)
0x0000_0000_0040_0000  ──── Program load address (typical)
0x0000_0000_0000_0000  ──── NULL (unmapped, any access → SIGSEGV)
```

### Level 3: The Page Table — Translating Virtual → Physical

The **page table** is a multi-level tree that maps virtual addresses to physical addresses:

```
Virtual address: 0xFFFF_C000_1234_5678
                 ──── Split into 4 parts ────
                 [L0 index][L1 index][L2 index][L3 index][offset]
                     9 bits   9 bits   9 bits   9 bits    12 bits

Translation:
L0 table (PGD): base stored in CPU register TTBR1_EL1
    → L0[index] points to L1 table
L1 table (PUD):
    → L1[index] points to L2 table
L2 table (PMD):
    → L2[index] points to L3 table
L3 table (PTE):
    → L3[index] = Physical Page Number + flags
        flags: Present, Writable, User/Kernel, Executable, Dirty, Accessed

Final: Physical address = PPN × 4096 + offset
```

**TLB** (Translation Lookaside Buffer) caches recent translations so we don't walk the 4-level table every time.

### Level 4: Page Fault Handler

When a process accesses a virtual address not yet mapped to physical RAM:

```
CPU → Page fault exception
    ↓
kernel: do_page_fault() / handle_mm_fault()
    ↓
  Is this a valid VMA (Virtual Memory Area)?
  ├── NO → Send SIGSEGV to process (segfault!)
  └── YES → What type of fault?
        ├── Anonymous page → alloc_page() + zero it + map it
        ├── File-backed page → read from disk, map it
        ├── COW page → allocate new page, copy content, map it
        └── Stack growth → extend stack VMA downward
    ↓
Return to user space — process continues, doesn't know it happened
```

```bash
# Watch page faults in real time:
perf stat -e page-faults ./my_program

# See a process's memory map:
cat /proc/$(pgrep my_driver)/maps
# Output:
# 7f8a1234000-7f8a2000000 r-xp 00000000 fd:00 12345 /lib/libc.so.6
# ADDR_START-ADDR_END PERMS OFFSET DEVICE INODE FILE
```

### Level 5: Kernel Memory Allocation — kmalloc vs vmalloc vs alloc_pages

```c
/* ─── kmalloc() ─────────────────────────────────────────────── */
/* Allocates physically contiguous memory from slab cache        */
/* Fastest, best for < 4MB, required for DMA                     */
void *buf = kmalloc(1024, GFP_KERNEL);   /* 1KB, may sleep */
void *buf = kmalloc(1024, GFP_ATOMIC);   /* 1KB, IRQ safe, never sleeps */
kfree(buf);

/* ─── kzalloc() ─────────────────────────────────────────────── */
/* kmalloc + zeroing (use instead of kmalloc + memset) */
void *buf = kzalloc(sizeof(struct my_data), GFP_KERNEL);

/* ─── vmalloc() ─────────────────────────────────────────────── */
/* Allocates virtually contiguous (NOT physically) memory        */
/* Slower, for large allocations > 1MB where contiguity matters  */
/* CANNOT use for DMA                                            */
void *buf = vmalloc(10 * 1024 * 1024);  /* 10MB */
vfree(buf);

/* ─── alloc_pages() / get_free_pages() ─────────────────────── */
/* Direct page allocator — physically contiguous                  */
/* order=0: 1 page (4KB), order=1: 2 pages, order=10: 1024 pages */
struct page *page = alloc_pages(GFP_KERNEL, order);
void *addr = page_address(page);
__free_pages(page, order);

/* ─── DMA-safe allocation ───────────────────────────────────── */
/* For device DMA — gets both virtual and physical addresses      */
dma_addr_t dma_handle;
void *virt = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
/* virt = virtual address (CPU uses this)         */
/* dma_handle = physical/bus address (device uses this) */
dma_free_coherent(dev, size, virt, dma_handle);
```

### GFP Flags — Choosing the Right Flag

```
GFP_KERNEL    → Can sleep, can swap, normal kernel allocation
               Use: any non-interrupt context (probe, open, read/write)

GFP_ATOMIC    → Cannot sleep, used in interrupt handlers and spinlock sections
               Smaller pool — use sparingly
               Use: IRQ handlers, timers, spinlock critical sections

GFP_DMA       → Must be in DMA zone (< 16MB for ISA DMA, usually not needed)
GFP_DMA32     → Must be accessible by 32-bit DMA devices (< 4GB)
GFP_NOWAIT    → Like ATOMIC but fails instead of allocating emergency pool
GFP_NOFS      → Cannot call filesystem code (used in filesystem code itself)
```

---

## 4. Interrupts — How Hardware Gets CPU Attention

### Level 1: The Analogy

You are working at a desk (CPU running your code). Instead of constantly checking your phone (polling), your phone **rings** (interrupt) when a message arrives. You pause your work, handle the message, then resume exactly where you left off.

### Level 2: Hardware Interrupt Flow

```
Hardware event (UART receives byte, DMA completes, timer fires)
    ↓
Interrupt Controller (GIC on ARM64) raises CPU IRQ line
    ↓
CPU saves current registers to stack (IRQ stack)
    ↓
CPU jumps to interrupt vector table → el1_irq handler
    ↓
Linux irq_handler_t called (your driver's ISR)
    ↓
ACK interrupt at hardware level (clear pending bit)
    ↓
CPU restores registers from stack
    ↓
CPU resumes original code from exactly where it was interrupted
```

### Level 3: Request and Handle IRQ in a Driver

```c
/* In driver probe(): */
int irq;
int ret;

/* Method 1: Get IRQ from Device Tree */
irq = platform_get_irq(pdev, 0);      /* index 0 = first IRQ in DT */
if (irq < 0) {
    dev_err(&pdev->dev, "No IRQ in DT: %d\n", irq);
    return irq;
}

/* Method 2: Get IRQ from GPIO */
irq = gpiod_to_irq(my_gpio);

/* Request the IRQ */
ret = devm_request_irq(&pdev->dev,    /* device (auto-freed on remove) */
    irq,                              /* IRQ number */
    my_irq_handler,                   /* handler function */
    IRQF_SHARED,                      /* flags: shared IRQ line */
    "my_driver",                      /* name (appears in /proc/interrupts) */
    my_dev);                          /* dev_id: identifies this handler */

/* The IRQ handler — runs in interrupt context (atomic) */
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 status;

    /* 1. Read interrupt status register */
    status = readl(dev->base + IRQ_STATUS_REG);

    /* 2. Check if this interrupt is for us (for shared IRQs) */
    if (!(status & MY_IRQ_BIT))
        return IRQ_NONE;    /* not ours */

    /* 3. Clear the interrupt (hardware-specific) */
    writel(MY_IRQ_BIT, dev->base + IRQ_CLEAR_REG);

    /* 4. Minimal work here (ISR must be fast!) */
    dev->irq_count++;
    
    /* 5. Schedule bottom half for heavy work */
    schedule_work(&dev->irq_work);    /* OR: tasklet_schedule(&dev->tasklet) */

    return IRQ_HANDLED;
}
```

### Level 4: Top Half vs Bottom Half (The Most Important IRQ Concept)

**The Problem:** IRQ handlers (top half) run with interrupts disabled. If they take too long, system responsiveness suffers. But processing a network packet or decoding data takes time.

**The Solution:** Split interrupt handling into two parts:

```
Hardware IRQ fires
    ↓
TOP HALF (ISR) — runs immediately, interrupts disabled
  Must be FAST: just read status, clear interrupt, wake bottom half
  No sleeping, no sleeping locks, no blocking operations
    ↓ schedule_work() / tasklet_schedule() / napi_schedule()
BOTTOM HALF — runs soon after, interrupts enabled
  Can do heavier work: memory allocation, DMA operations, notify waiters
```

**Bottom Half Mechanisms:**

```c
/* 1. Softirq — fastest, runs in softirq context */
/* Built into kernel, limited set (NET_RX, NET_TX, BLOCK, TASKLET...) */

/* 2. Tasklet — built on softirqs, easy to use */
DECLARE_TASKLET(my_tasklet, my_tasklet_func);  /* static */

static void my_tasklet_func(struct tasklet_struct *t) {
    /* runs in softirq context — no sleep, no sleepable locks */
    process_received_data();
}

/* In ISR: */
tasklet_schedule(&my_tasklet);

/* 3. Workqueue — most flexible, runs in kernel thread context */
DECLARE_WORK(my_work, my_work_func);  /* static */

static void my_work_func(struct work_struct *work) {
    /* runs in process context — CAN sleep, CAN use mutex */
    if (copy_to_user(ubuf, kbuf, size)) ...   /* even this is ok! */
}

/* In ISR: */
schedule_work(&my_work);

/* 4. Threaded IRQ — entire IRQ in a kernel thread */
request_threaded_irq(irq, my_hard_irq, my_thread_fn, flags, name, dev);
/* my_hard_irq: minimal, just returns IRQ_WAKE_THREAD */
/* my_thread_fn: full processing in kernel thread context */
```

---

## 5. Synchronization — Protecting Shared Data

### Level 1: The Race Condition Problem

```c
/* BUG: Two CPU cores running this simultaneously */
int counter = 0;

void increment(void) {          /* CPU 0 and CPU 1 both call this */
    counter++;                  /* NOT atomic! This is 3 instructions:
                                   1. LOAD counter → register
                                   2. ADD register, 1
                                   3. STORE register → counter
                                   CPU 0 and CPU 1 can interleave! */
}

/* Result: counter ends up as 1 instead of 2 */
```

### Level 2: When to Use What

```
Scenario → Tool
──────────────────────────────────────────────────────────────────────
Short critical section, IRQ context possible → spinlock
Longer critical section, process context only → mutex
Read-heavy, rare writes → rwlock or RCU
Counting (semaphore style) → semaphore
Wait for event/condition → wait_queue
Sequence of operations must complete atomically → atomic_t
One-time initialization → once_t / static init
IRQ must not fire during section → local_irq_disable + spinlock
```

### Level 3: spinlock — The Basic Lock

```c
#include <linux/spinlock.h>

/* Declaration */
DEFINE_SPINLOCK(my_lock);   /* static */
/* OR */
spinlock_t my_lock;
spin_lock_init(&my_lock);   /* dynamic */

/* Usage in normal process context (may be interrupted) */
spin_lock(&my_lock);
    /* critical section — no sleep allowed here! */
    list_add(&item->list, &my_list);
spin_unlock(&my_lock);

/* Usage when ISR and process code share data */
unsigned long flags;
spin_lock_irqsave(&my_lock, flags);   /* disables IRQ on this CPU */
    /* safe from both process and ISR context */
    fifo_push(&my_fifo, data);
spin_unlock_irqrestore(&my_lock, flags);  /* restores IRQ state */

/* In the ISR (which also accesses my_fifo) */
spin_lock(&my_lock);   /* Note: NOT irqsave, already in IRQ context */
    data = fifo_pop(&my_fifo);
spin_unlock(&my_lock);
```

### Level 4: mutex — Sleepable Lock

```c
#include <linux/mutex.h>

/* Declaration */
DEFINE_MUTEX(my_mutex);   /* static */
/* OR */
struct mutex my_mutex;
mutex_init(&my_mutex);    /* dynamic */

/* Lock (may sleep if mutex is held by another task) */
mutex_lock(&my_mutex);
    /* critical section */
    /* CAN sleep here — can call kmalloc(GFP_KERNEL), copy_to_user, etc. */
    ret = do_some_work();
mutex_unlock(&my_mutex);

/* Non-blocking try: */
if (mutex_trylock(&my_mutex)) {
    /* got the lock */
    mutex_unlock(&my_mutex);
} else {
    /* someone else has it */
}

/* Interruptible lock (returns -EINTR if signal received): */
ret = mutex_lock_interruptible(&my_mutex);
if (ret)
    return ret;
```

### Level 5: wait_queue — Waiting for Events

```c
/* Pattern: process waits for data, IRQ wakes it up */
DECLARE_WAIT_QUEUE_HEAD(my_waitq);

/* Reader (process context): */
int my_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
    /* Wait until data is available */
    ret = wait_event_interruptible(my_waitq, 
                                   my_dev->data_ready != 0);
    if (ret)
        return ret;   /* -ERESTARTSYS if signal received */
    
    /* Copy data to user */
    copy_to_user(buf, my_dev->kbuf, count);
    my_dev->data_ready = 0;
    return count;
}

/* IRQ handler (interrupt context): */
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    /* Data arrived! */
    dev->data_ready = 1;
    wake_up_interruptible(&my_waitq);   /* wake all sleeping readers */
    return IRQ_HANDLED;
}
```

---

## 6. Workqueues and kthreads — Background Work

### Workqueue (Deferred Work)

```c
/* Using the global system workqueue (simplest) */
struct work_struct my_work;

void my_work_function(struct work_struct *work)
{
    struct my_device *dev = container_of(work, struct my_device, work);
    /* Process data, allocate memory, etc. */
    process_data(dev);
}

/* In probe: */
INIT_WORK(&dev->work, my_work_function);

/* In ISR: */
schedule_work(&dev->work);

/* Delayed work (runs after a delay): */
struct delayed_work my_delayed_work;
INIT_DELAYED_WORK(&dev->dwork, my_delayed_work_fn);
schedule_delayed_work(&dev->dwork, msecs_to_jiffies(100)); /* 100ms */
```

### kthread — Dedicated Kernel Thread

```c
#include <linux/kthread.h>

struct task_struct *my_thread;

/* Thread function */
static int my_thread_fn(void *data)
{
    struct my_device *dev = data;
    
    while (!kthread_should_stop()) {
        /* Do periodic work */
        process_data(dev);
        
        /* Sleep for 100ms */
        msleep_interruptible(100);
    }
    
    return 0;
}

/* In probe — create and start the thread */
my_thread = kthread_run(my_thread_fn, dev, "my_driver_thread");
if (IS_ERR(my_thread)) {
    dev_err(&pdev->dev, "Failed to create thread\n");
    return PTR_ERR(my_thread);
}

/* In remove — stop the thread */
kthread_stop(my_thread);
```

---

## 7. VFS — Virtual Filesystem Layer

### Level 1: Why VFS Exists

Linux supports 30+ filesystems: ext4, btrfs, tmpfs, procfs, sysfs, debugfs, nfs...

Without VFS, every app would need to know which filesystem it's talking to:
```c
/* WITHOUT VFS (bad): */
if (fs_type == EXT4)
    ext4_read(file, buf, count);
else if (fs_type == BTRFS)
    btrfs_read(file, buf, count);
```

VFS provides a **uniform interface**:
```c
/* WITH VFS (good): */
read(fd, buf, count);   /* same call for ALL filesystems */
```

### Level 2: VFS Key Objects

```
struct super_block    → represents a mounted filesystem
struct inode          → represents a file (metadata: size, permissions, timestamps)
struct dentry         → represents a path component (directory entry: "hello_world.txt")
struct file           → represents an open file (per open() call)
struct file_operations → function pointers: read, write, open, ioctl...
```

```
Path: /home/ravi/driver.c

dentry["/"]           → inode for root directory
  dentry["home"]      → inode for /home directory
    dentry["ravi"]    → inode for /home/ravi directory
      dentry["driver.c"] → inode for /home/ravi/driver.c
                              → inode has: size=5012, owner=ravi, perms=644
                              → inode points to data blocks on disk
```

---

## 8. Interview Questions — All Levels

| Level | Question | Key Answer |
|-------|----------|-----------|
| **Beginner** | What is the difference between a process and a thread? | Both are task_struct, threads share mm_struct |
| **Beginner** | What is a page fault? | CPU exception when accessing unmapped virtual address |
| **Intermediate** | What is Copy-On-Write and when is it used? | fork() shares pages until write, then private copy |
| **Intermediate** | When would you use kmalloc vs vmalloc? | kmalloc: < 4MB, DMA, physically contiguous; vmalloc: large, physically scattered |
| **Intermediate** | What is the difference between spin_lock and mutex? | spinlock: busy-wait, atomic safe; mutex: sleeps, process context only |
| **Advanced** | Explain vruntime in CFS scheduler. How does nice value affect it? | vruntime accumulates faster for higher nice, so lower priority = less CPU |
| **Advanced** | What are the three bottom half mechanisms? When to use each? | softirq (built-in), tasklet (dynamic, no sleep), workqueue (sleepable) |
| **Advanced** | What does spin_lock_irqsave do differently from spin_lock? | Disables IRQs on current CPU, prevents ISR from taking same lock → deadlock |
| **Expert** | Explain the kernel slab allocator. What is slab, slub, slob? | Object caches for frequent allocation patterns; slub is default today |
| **Expert** | What is RCU and when is it better than rwlock? | Read-Copy-Update: readers never block, writer makes new copy; better for read-heavy |
| **Expert** | How does the kernel handle a kernel stack overflow? | CONFIG_VMAP_STACK: virtually mapped stacks with guard pages, generates OOPS |

---

## 9. Practical Labs

### Lab 1: Trace the scheduler in real time (15 min)

```bash
# See which process runs on each CPU
cat /proc/schedstat

# Trace CFS scheduler events
sudo trace-cmd record -e sched:sched_switch sleep 5
sudo trace-cmd report | head -50

# See process wait times (how long each waited for CPU)
perf sched record -- sleep 5
perf sched latency | head -30
```

### Lab 2: Trigger and analyze a page fault (20 min)

```c
/* Write this program, compile, and run under strace */
/* mmap_test.c */
#include <sys/mman.h>
#include <stdio.h>
int main(void) {
    char *p = mmap(NULL, 4096, PROT_READ|PROT_WRITE,
                   MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
    /* Page not yet allocated — will fault on first write */
    *p = 42;    /* ← triggers page fault (handled transparently) */
    printf("wrote: %d\n", *p);
    munmap(p, 4096);
    return 0;
}
```

```bash
gcc mmap_test.c -o mmap_test
# Count page faults:
perf stat -e page-faults ./mmap_test
```

### Lab 3: Write a simple character driver using wait_queue (30 min)

See `07_Device_Drivers/01_Driver_From_Scratch_Complete_Guide.md` for the full exercise.
