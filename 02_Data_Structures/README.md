# 02 — Data Structures & Algorithms for Kernel Work

> The kernel uses its own data structure implementations optimized for lock-free, cache-friendly, and interrupt-safe operation.

---

## Section Structure

```
02_Data_Structures/
├── 01_Kernel_Linked_Lists.md         ← list_head, hlist, doubly-linked traversal
├── 02_Rbtree.md                      ← Red-black trees: timer, CFS scheduler
├── 03_Kfifo.md                       ← Lock-free ring buffer for driver I/O
├── 04_Radix_Tree_XArray.md           ← Page cache, inode mapping
├── 05_Hash_Tables.md                 ← jhash, hashtable.h macros
├── 06_Wait_Queues.md                 ← wait_queue_head_t, wake_up patterns
├── 07_Completion.md                  ← struct completion, init/wait/complete
├── 08_Work_Queues.md                 ← Deferred work: workqueue, delayed_work
├── 09_RCU_Concepts.md                ← Read-Copy-Update — the kernel's secret weapon
└── 10_Interview_DSA_Questions.md     ← 40 kernel DSA interview questions
```

---

## Kernel Data Structures Quick Reference

### list_head — Intrusive Doubly Linked List

```c
struct my_item {
    int data;
    struct list_head node;   /* embed the list node */
};

/* Initialize */
LIST_HEAD(my_list);          /* static: struct list_head my_list = LIST_HEAD_INIT(my_list) */

/* Add */
list_add(&item->node, &my_list);       /* add at head */
list_add_tail(&item->node, &my_list);  /* add at tail */

/* Traverse */
struct my_item *pos;
list_for_each_entry(pos, &my_list, node) {
    printk("data: %d\n", pos->data);
}

/* Safe traverse (allows deletion during iteration) */
struct my_item *tmp;
list_for_each_entry_safe(pos, tmp, &my_list, node) {
    list_del(&pos->node);
    kfree(pos);
}
```

---

### kfifo — Lock-Free Ring Buffer

```c
#include <linux/kfifo.h>

DEFINE_KFIFO(my_fifo, u8, 64);    /* 64-byte ring buffer */

/* Producer (e.g., IRQ handler) */
kfifo_put(&my_fifo, byte);

/* Consumer (e.g., read syscall) */
u8 byte;
if (kfifo_get(&my_fifo, &byte)) {
    /* process byte */
}

/* Copy to/from user space */
kfifo_to_user(&my_fifo, ubuf, count, &copied);
kfifo_from_user(&my_fifo, ubuf, count, &copied);
```

---

### RCU — Read-Copy-Update

```c
/* RCU-protected pointer */
struct config *rcu_cfg __rcu;

/* Reader — no lock, just RCU read-side critical section */
rcu_read_lock();
struct config *cfg = rcu_dereference(rcu_cfg);
int val = cfg->threshold;   /* safe to read */
rcu_read_unlock();

/* Writer — update by replace, not modify */
struct config *new_cfg = kmalloc(sizeof(*new_cfg), GFP_KERNEL);
*new_cfg = *old_cfg;
new_cfg->threshold = 42;

struct config *old = rcu_dereference_protected(rcu_cfg, lockdep_is_held(&update_lock));
rcu_assign_pointer(rcu_cfg, new_cfg);
synchronize_rcu();   /* wait for all readers to finish with old */
kfree(old);
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What is `list_head`? How is it different from a traditional linked list node? |
| **Intermediate** | What is `kfifo` and why is it safe between one producer and one consumer without locks? |
| **Advanced** | Explain RCU. When can you use it and what is `synchronize_rcu`? |
| **Expert** | Compare RCU vs spinlock vs mutex for a read-heavy shared data structure in an interrupt context. |
