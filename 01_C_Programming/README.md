# 01 — C Programming for Embedded Linux

> Mastering C is non-negotiable for kernel and driver development. This section takes you from C basics to kernel-style C idioms used in production Linux code.

---

## Section Structure

```
01_C_Programming/
├── 01_C_Fundamentals.md              ← Pointers, arrays, structs, unions, bitfields
├── 02_Memory_Management.md           ← Stack vs heap, malloc vs kmalloc
├── 03_Pointers_Deep_Dive.md          ← Function pointers, pointer arithmetic, void*
├── 04_Bitwise_Operations.md          ← Masks, shifts, register manipulation
├── 05_Volatile_Restrict_Inline.md    ← Qualifiers critical for embedded/kernel work
├── 06_Kernel_C_Style.md              ← Linux CodingStyle, checkpatch, sparse
├── 07_Common_Patterns.md             ← container_of, list_head, RCU wrappers
├── 08_Data_Structures_In_C.md        ← Linked lists, trees, hash tables
├── 09_Debugging_C_Code.md            ← Valgrind, AddressSanitizer, static analysis
└── 10_Interview_C_Questions.md       ← 50 kernel-style C interview questions
```

---

## Why C Mastery Is Critical

The Linux kernel is 99.9% C. Device drivers are C. Bootloaders are C. If your C is weak, every kernel bug takes 10× longer to debug.

Key areas kernel developers must master:
- **Pointer arithmetic** — walking device register maps, DMA scatter-gather lists
- **Bitfield manipulation** — reading SoC registers, parsing DT properties
- **volatile** — memory-mapped I/O, must-not-be-optimized registers
- **`__attribute__` extensions** — GCC packed structs, alignment, likely/unlikely
- **container_of** — the magic macro that makes the entire Linux driver model work

---

## Quick Reference: Kernel-Critical C Patterns

### container_of — The Kernel's Most Important Macro

```c
#define container_of(ptr, type, member) ({                \
    const typeof(((type *)0)->member) *__mptr = (ptr);    \
    (type *)((char *)__mptr - offsetof(type, member)); })
```

**Usage:**
```c
struct my_device {
    int id;
    struct platform_device *pdev;
    struct list_head list;      ← embedded in larger struct
};

/* When you have a pointer to list_head, get back to my_device: */
struct my_device *dev = container_of(list_ptr, struct my_device, list);
```

---

### Bitfield Register Access

```c
/* SoC register bit definitions */
#define REG_CTRL        0x00
#define REG_CTRL_ENABLE BIT(0)
#define REG_CTRL_RESET  BIT(1)
#define REG_CTRL_MODE   GENMASK(5, 4)   /* bits [5:4] */

/* Read-modify-write pattern */
u32 val = readl(base + REG_CTRL);
val &= ~REG_CTRL_MODE;                  /* clear mode bits */
val |= FIELD_PREP(REG_CTRL_MODE, 2);    /* set mode = 2 */
writel(val, base + REG_CTRL);
```

---

### volatile for MMIO

```c
/* WRONG — compiler may optimize away repeated reads */
uint32_t *reg = (uint32_t *)MMIO_BASE;
while (*reg & STATUS_BUSY) ;

/* RIGHT — volatile prevents optimization */
volatile uint32_t *reg = (volatile uint32_t *)MMIO_BASE;
while (*reg & STATUS_BUSY) ;

/* KERNEL WAY — use readl/writel which include memory barriers */
while (readl(base + STATUS_REG) & STATUS_BUSY) ;
```

---

### Function Pointers (Driver Model Foundation)

```c
struct file_operations {
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    int     (*open)(struct inode *, struct file *);
    int     (*release)(struct inode *, struct file *);
};

/* Implementing the ops */
static const struct file_operations my_fops = {
    .owner   = THIS_MODULE,
    .read    = my_read,
    .write   = my_write,
    .open    = my_open,
    .release = my_release,
};
```

---

## Interview Questions (C for Kernel Developers)

| Level | Question |
|-------|----------|
| **Basic** | What is the difference between `const int *p` and `int * const p`? |
| **Basic** | What does `volatile` do and when must you use it in embedded code? |
| **Intermediate** | Explain `container_of`. How does it work without knowing the type at compile time? |
| **Intermediate** | What is the difference between `kmalloc` and `vmalloc`? When to use each? |
| **Advanced** | What is the Linux `__iomem` annotation and what does `sparse` check for? |
| **Advanced** | Explain `FIELD_PREP` and `FIELD_GET`. Why are they safer than manual bit shifts? |
| **Expert** | What is the strict aliasing rule and why does Linux use `-fno-strict-aliasing`? |
| **Expert** | How does `READ_ONCE`/`WRITE_ONCE` differ from `volatile`? When does the kernel use each? |

---

## Resources

- [Linux kernel coding style](https://www.kernel.org/doc/html/latest/process/coding-style.html)
- [Kernel Newbies C basics](https://kernelnewbies.org/FAQ/C)
- Book: *Linux Device Drivers* (LDD3) — Chapter 1-3 for C patterns
- Tool: `scripts/checkpatch.pl` — enforces CodingStyle on your patches
