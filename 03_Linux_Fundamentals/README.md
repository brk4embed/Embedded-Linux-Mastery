# 03 — Linux Fundamentals

> Before diving into the kernel, master how Linux works from a systems perspective: processes, memory, filesystems, IPC, and the syscall interface.

---

## Section Structure

```
03_Linux_Fundamentals/
├── 01_Process_Model.md               ← fork/exec/wait, task_struct, namespaces
├── 02_Memory_Layout.md               ← Virtual address space, mmap, pages
├── 03_Filesystem_Internals.md        ← VFS, inode, dentry, superblock
├── 04_Syscall_Interface.md           ← How syscalls work, adding a syscall
├── 05_IPC_Mechanisms.md              ← pipes, sockets, signals, shared memory
├── 06_Scheduling_Basics.md           ← CFS, RT scheduler, priorities, cgroups
├── 07_Networking_Stack.md            ← sk_buff, netdev, TCP/IP in the kernel
├── 08_Signals_And_Timers.md          ← signal handling, hrtimer, workqueue timers
├── 09_Namespaces_Cgroups.md          ← Container isolation primitives
└── 10_Interview_Linux_Questions.md   ← 60 Linux fundamentals Q&A
```

---

## Linux System Architecture

```
┌─────────────────────────────────────────────────┐
│                  User Space                       │
│  bash  │  python  │  systemd  │  your-app        │
│        glibc (libc syscall wrappers)              │
└────────────────────┬────────────────────────────-┘
          syscall interface (INT 0x80 / SVC / SYSCALL)
┌────────────────────┴────────────────────────────-┐
│                 Kernel Space                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Process  │  │  Memory  │  │  Filesystem  │   │
│  │ Scheduler│  │  Manager │  │  (VFS)       │   │
│  └──────────┘  └──────────┘  └──────────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │  Device  │  │ Network  │  │  Security    │   │
│  │  Drivers │  │  Stack   │  │  (SELinux)   │   │
│  └──────────┘  └──────────┘  └──────────────┘   │
└────────────────────┬────────────────────────────-┘
                     │
┌────────────────────┴────────────────────────────-┐
│                  Hardware                         │
│  CPU  │  Memory  │  Storage  │  Network  │  GPIO │
└─────────────────────────────────────────────────-┘
```

---

## Key Concepts

### Process vs Thread in Linux
- Both are `task_struct` in the kernel
- `fork()` creates a new task with COW (copy-on-write) address space
- `clone()` with `CLONE_VM` creates a thread sharing the address space
- `pthread_create()` calls `clone()` internally

### Virtual Memory Layout (64-bit ARM)
```
0xFFFF_FFFF_FFFF_FFFF  ─── kernel space top
    ...
0xFFFF_0000_0000_0000  ─── kernel space start
                       ─── (hole: non-canonical)
0x0000_FFFF_FFFF_FFFF  ─── user space top
    stack              ─── grows downward
    mmap area          ─── shared libs, anonymous maps
    heap               ─── grows upward (brk)
    BSS / data / text  ─── ELF segments
0x0000_0000_0040_0000  ─── typical load address
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What is the difference between a process and a thread in Linux? |
| **Basic** | What does `fork()` return in the parent vs child? |
| **Intermediate** | What is Copy-On-Write (COW)? How does it make `fork()` fast? |
| **Intermediate** | What is the VFS layer and why does Linux need it? |
| **Advanced** | What is a page fault? Explain minor vs major fault. |
| **Advanced** | How does the CFS scheduler calculate priority? What is vruntime? |
| **Expert** | Explain the difference between `ioremap` and `mmap` for device memory access. |
