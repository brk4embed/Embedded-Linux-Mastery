# C Multithreading and Design Patterns — Complete Guide

> **Prerequisite:** You know C functions and structs. This guide builds from zero to writing multi-threaded embedded Linux applications and kernel threads.

---

## Part 1: Why Multithreading? (The Problem It Solves)

### Single-threaded limitation

```c
/* Single-threaded: sensor read blocks everything */
while (1) {
    read_uart_data();       /* blocks for 10ms waiting for bytes */
    process_sensor_data();  /* takes 5ms */
    send_to_server();       /* blocks 100ms on network */
    update_display();       /* takes 2ms */
}
/* Total: ~117ms per loop. Can't respond to events in between. */
```

### Multi-threaded solution

```
Thread 1 (UART):     read bytes → put in queue (never blocked)
Thread 2 (Process):  read from queue → process → put in result queue  
Thread 3 (Network):  read result queue → send (can block, doesn't affect others)
Thread 4 (Display):  update UI every 16ms independently
```

All four things happen **simultaneously**, each independent of the others.

---

## Part 2: POSIX Threads (pthreads) — The Foundation

### Thread Creation

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/* Function signature ALL thread functions must match */
void *thread_function(void *arg) {
    int thread_num = *(int *)arg;   /* cast void* back to your type */
    
    printf("Thread %d: running on CPU %d\n", thread_num, sched_getcpu());
    
    /* Do work */
    sleep(1);
    
    /* Return a value (or NULL) */
    int *result = malloc(sizeof(int));
    *result = thread_num * 2;
    return (void *)result;
}

int main(void) {
    pthread_t threads[4];
    int thread_ids[4] = {1, 2, 3, 4};
    
    /* Create 4 threads */
    for (int i = 0; i < 4; i++) {
        int ret = pthread_create(
            &threads[i],          /* thread handle (output) */
            NULL,                 /* attributes (NULL = defaults) */
            thread_function,      /* function to run */
            &thread_ids[i]        /* argument to pass */
        );
        if (ret != 0) {
            fprintf(stderr, "Failed to create thread: %s\n", strerror(ret));
            exit(1);
        }
    }
    
    /* Wait for all threads to complete */
    for (int i = 0; i < 4; i++) {
        void *retval;
        pthread_join(threads[i], &retval);   /* blocks until thread i done */
        
        if (retval) {
            int *result = (int *)retval;
            printf("Thread %d returned: %d\n", i+1, *result);
            free(result);
        }
    }
    
    printf("All threads done\n");
    return 0;
}
```

```bash
gcc -o thread_demo thread_demo.c -lpthread  # -lpthread is REQUIRED
./thread_demo
```

---

## Part 3: Synchronization — Protecting Shared Data

### The Race Condition Problem

```c
/* BROKEN — race condition */
int counter = 0;   /* shared between threads */

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        counter++;   /* NOT atomic: read → increment → write (3 steps) */
        /* Two threads can both read '5', both write '6', losing one increment */
    }
    return NULL;
}

/* Expected: 200000. Actual: random between 100000-200000 each run */
```

### Solution 1: Mutex (Mutual Exclusion Lock)

```c
#include <pthread.h>

/* Global mutex — must be shared by ALL threads that access counter */
pthread_mutex_t counter_mutex = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&counter_mutex);   /* LOCK: only one thread enters */
        counter++;                             /* now this is safe */
        pthread_mutex_unlock(&counter_mutex); /* UNLOCK: next thread can enter */
    }
    return NULL;
}

/* Dynamic init (needed when mutex is in a struct) */
void init_my_struct(struct my_data *d) {
    pthread_mutex_init(&d->lock, NULL);   /* NULL = default attributes */
}

void cleanup_my_struct(struct my_data *d) {
    pthread_mutex_destroy(&d->lock);
}
```

### Solution 2: Condition Variables — Thread Signaling

Mutex protects data. Condition variable lets threads **wait for an event** without busy-polling.

```c
/* Producer-Consumer with mutex + condvar */
#include <pthread.h>
#include <stdbool.h>

#define QUEUE_SIZE 16

typedef struct {
    int items[QUEUE_SIZE];
    int head, tail, count;
    pthread_mutex_t lock;
    pthread_cond_t  not_empty;   /* signal: item available */
    pthread_cond_t  not_full;    /* signal: space available */
} bounded_queue_t;

void queue_init(bounded_queue_t *q) {
    q->head = q->tail = q->count = 0;
    pthread_mutex_init(&q->lock, NULL);
    pthread_cond_init(&q->not_empty, NULL);
    pthread_cond_init(&q->not_full, NULL);
}

/* Producer: add item (blocks if queue full) */
void queue_push(bounded_queue_t *q, int item) {
    pthread_mutex_lock(&q->lock);
    
    /* Wait while queue is full */
    while (q->count == QUEUE_SIZE) {
        /* IMPORTANT: pthread_cond_wait atomically:
           1. Releases the mutex
           2. Blocks the thread
           3. When signaled: re-acquires mutex, then returns */
        pthread_cond_wait(&q->not_full, &q->lock);
    }
    
    q->items[q->tail] = item;
    q->tail = (q->tail + 1) % QUEUE_SIZE;
    q->count++;
    
    pthread_cond_signal(&q->not_empty);   /* wake up one consumer */
    pthread_mutex_unlock(&q->lock);
}

/* Consumer: get item (blocks if queue empty) */
int queue_pop(bounded_queue_t *q) {
    pthread_mutex_lock(&q->lock);
    
    while (q->count == 0) {
        pthread_cond_wait(&q->not_empty, &q->lock);
    }
    
    int item = q->items[q->head];
    q->head = (q->head + 1) % QUEUE_SIZE;
    q->count--;
    
    pthread_cond_signal(&q->not_full);    /* wake up one producer */
    pthread_mutex_unlock(&q->lock);
    return item;
}
```

### Solution 3: Semaphore

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 5);   /* initial value = 5 (5 resources available) */

/* Acquire resource (blocks if count == 0) */
sem_wait(&sem);    /* decrements counter */
/* use resource */
sem_post(&sem);    /* increments counter, wakes one waiter */

sem_destroy(&sem);

/* Named semaphore (survives between processes) */
sem_t *named = sem_open("/my_semaphore", O_CREAT, 0644, 1);
sem_wait(named);
/* critical section */
sem_post(named);
sem_close(named);
sem_unlink("/my_semaphore");   /* remove when done */
```

---

## Part 4: Kernel Threads (kthread)

### Why Kernel Threads?

User-space pthreads can't run in kernel context. Kernel threads run in kernel space — same as interrupt handlers — and can call any kernel API.

Use kernel threads in drivers when you need:
- Periodic background work (poll hardware every 100ms)
- Process a workqueue of items without blocking the interrupt handler
- Long-running operations that shouldn't happen in interrupt context

```c
#include <linux/kthread.h>
#include <linux/sched.h>
#include <linux/delay.h>

struct my_driver_data {
    struct task_struct *poll_thread;
    bool               stop_thread;
    /* ... */
};

/* Thread function */
static int my_poll_thread(void *data)
{
    struct my_driver_data *drv = data;
    
    pr_info("Poll thread started on CPU %d\n", smp_processor_id());
    
    /* kthread_should_stop() returns true when kthread_stop() is called */
    while (!kthread_should_stop()) {
        /* Do periodic work */
        u32 status = readl(drv->base + REG_STATUS);
        if (status & STATUS_DATA_READY) {
            process_data(drv);
        }
        
        /* Sleep 10ms — don't busy-poll! */
        msleep_interruptible(10);
        /* msleep_interruptible wakes early if signal received */
    }
    
    pr_info("Poll thread exiting\n");
    return 0;
}

/* In probe() — start the thread */
static int my_probe(struct platform_device *pdev)
{
    struct my_driver_data *drv;
    /* ... setup ... */
    
    /* Create and start kernel thread */
    drv->poll_thread = kthread_run(
        my_poll_thread,     /* function */
        drv,                /* argument (passed as void *data) */
        "my-poll-%s",       /* thread name (appears in ps/top) */
        dev_name(&pdev->dev)
    );
    
    if (IS_ERR(drv->poll_thread))
        return PTR_ERR(drv->poll_thread);
    
    return 0;
}

/* In remove() — stop the thread */
static int my_remove(struct platform_device *pdev)
{
    struct my_driver_data *drv = platform_get_drvdata(pdev);
    
    /* Signal thread to stop and wait for it to exit */
    kthread_stop(drv->poll_thread);
    
    return 0;
}
```

---

## Part 5: Embedded C Design Patterns

### Pattern 1: State Machine

Used in: Protocol handlers, device state management, UI logic.

```c
/* State machine for a UFS command processor */
typedef enum {
    STATE_IDLE,
    STATE_SENDING_UPIU,
    STATE_WAITING_RESPONSE,
    STATE_PROCESSING,
    STATE_ERROR,
} ufs_state_t;

typedef struct {
    ufs_state_t state;
    void *pending_cmd;
    uint32_t error_code;
} ufs_sm_t;

/* State transition table */
typedef void (*state_handler_t)(ufs_sm_t *sm, uint32_t event);

void handle_idle(ufs_sm_t *sm, uint32_t event) {
    switch (event) {
    case EVT_CMD_QUEUED:
        send_upiu(sm->pending_cmd);
        sm->state = STATE_SENDING_UPIU;
        break;
    default:
        break;
    }
}

void handle_waiting_response(ufs_sm_t *sm, uint32_t event) {
    switch (event) {
    case EVT_RESPONSE_RECEIVED:
        sm->state = STATE_PROCESSING;
        break;
    case EVT_TIMEOUT:
        sm->error_code = ERR_TIMEOUT;
        sm->state = STATE_ERROR;
        break;
    default:
        break;
    }
}

static const state_handler_t handlers[] = {
    [STATE_IDLE]             = handle_idle,
    [STATE_SENDING_UPIU]     = handle_sending,
    [STATE_WAITING_RESPONSE] = handle_waiting_response,
    [STATE_PROCESSING]       = handle_processing,
    [STATE_ERROR]            = handle_error,
};

void ufs_sm_event(ufs_sm_t *sm, uint32_t event) {
    if (sm->state < ARRAY_SIZE(handlers) && handlers[sm->state])
        handlers[sm->state](sm, event);
}
```

### Pattern 2: Observer (Callback Registry)

Used in: Subsystem event notification, plugin systems.

```c
/* Event notification system */
#define MAX_LISTENERS 8

typedef void (*event_callback_t)(void *ctx, uint32_t event, void *data);

typedef struct {
    event_callback_t callbacks[MAX_LISTENERS];
    void            *contexts[MAX_LISTENERS];
    int              count;
    pthread_mutex_t  lock;
} event_bus_t;

void event_bus_init(event_bus_t *bus) {
    bus->count = 0;
    pthread_mutex_init(&bus->lock, NULL);
}

int event_bus_subscribe(event_bus_t *bus, event_callback_t cb, void *ctx) {
    pthread_mutex_lock(&bus->lock);
    if (bus->count >= MAX_LISTENERS) {
        pthread_mutex_unlock(&bus->lock);
        return -1;
    }
    bus->callbacks[bus->count] = cb;
    bus->contexts[bus->count] = ctx;
    bus->count++;
    pthread_mutex_unlock(&bus->lock);
    return 0;
}

void event_bus_publish(event_bus_t *bus, uint32_t event, void *data) {
    pthread_mutex_lock(&bus->lock);
    int n = bus->count;
    /* Copy to local to minimize lock hold time */
    event_callback_t cbs[MAX_LISTENERS];
    void *ctxs[MAX_LISTENERS];
    memcpy(cbs, bus->callbacks, n * sizeof(*cbs));
    memcpy(ctxs, bus->contexts, n * sizeof(*ctxs));
    pthread_mutex_unlock(&bus->lock);
    
    /* Call outside lock to prevent deadlock */
    for (int i = 0; i < n; i++)
        cbs[i](ctxs[i], event, data);
}
```

### Pattern 3: Object Pool (Avoid malloc in Fast Paths)

```c
/* Pre-allocated pool of command objects — no malloc in hot path */
#define POOL_SIZE 32

typedef struct {
    /* command data */
    uint32_t opcode;
    uint8_t  lun;
    uint64_t lba;
    uint32_t transfer_length;
    bool     in_use;
} ufs_cmd_t;

typedef struct {
    ufs_cmd_t   cmds[POOL_SIZE];
    pthread_mutex_t lock;
} cmd_pool_t;

ufs_cmd_t *pool_alloc(cmd_pool_t *pool) {
    pthread_mutex_lock(&pool->lock);
    for (int i = 0; i < POOL_SIZE; i++) {
        if (!pool->cmds[i].in_use) {
            pool->cmds[i].in_use = true;
            pthread_mutex_unlock(&pool->lock);
            memset(&pool->cmds[i], 0, sizeof(ufs_cmd_t));
            pool->cmds[i].in_use = true;
            return &pool->cmds[i];
        }
    }
    pthread_mutex_unlock(&pool->lock);
    return NULL;   /* pool exhausted */
}

void pool_free(cmd_pool_t *pool, ufs_cmd_t *cmd) {
    pthread_mutex_lock(&pool->lock);
    cmd->in_use = false;
    pthread_mutex_unlock(&pool->lock);
}
```

---

## Part 6: Common Multithreading Bugs

### Bug 1: Deadlock

```c
/* DEADLOCK: Thread A holds L1, waits L2. Thread B holds L2, waits L1. */
pthread_mutex_t L1, L2;

/* Thread A */
pthread_mutex_lock(&L1);
pthread_mutex_lock(&L2);   /* waits forever if Thread B holds L2 */

/* Thread B */
pthread_mutex_lock(&L2);
pthread_mutex_lock(&L1);   /* waits forever if Thread A holds L1 */

/* FIX: Always lock in the same ORDER in all threads */
/* Thread A AND Thread B: always lock L1 first, then L2 */
pthread_mutex_lock(&L1);
pthread_mutex_lock(&L2);
```

### Bug 2: Priority Inversion

```c
/* Low-priority task holds mutex needed by high-priority task */
/* High-priority task blocks → misses deadline */

/* FIX: Use priority-inheriting mutex */
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
/* Low-priority task temporarily inherits high-priority's priority while holding mutex */
pthread_mutex_init(&my_mutex, &attr);
```

### Bug 3: Spurious Wakeups

```c
/* WRONG: if() — thread can wake spuriously (OS implementation detail) */
pthread_mutex_lock(&lock);
if (count == 0)                          /* BUG: use while, not if */
    pthread_cond_wait(&cond, &lock);
/* Do work — but count might still be 0 if spurious wakeup! */

/* RIGHT: while() — re-check condition after every wakeup */
pthread_mutex_lock(&lock);
while (count == 0)                       /* CORRECT */
    pthread_cond_wait(&cond, &lock);
/* Now guaranteed: count > 0 */
```

---

## Interview Questions — Multithreading

| Level | Question | Key Answer |
|-------|----------|-----------|
| **Beginner** | What is a race condition? Give an example | Two threads read-modify-write same variable without protection |
| **Beginner** | What is a mutex? When do you use it? | Binary lock protecting shared data; use whenever multiple threads access shared state |
| **Intermediate** | What is the difference between mutex and semaphore? | Mutex: owned by one thread (same thread must unlock); semaphore: signaling mechanism, any thread can post |
| **Intermediate** | What is a condition variable? Why use `while` not `if`? | Allows threads to wait for a condition; `while` handles spurious wakeups |
| **Intermediate** | What is a deadlock? How do you prevent it? | Circular wait for locks; prevention: always lock in same order |
| **Advanced** | What is priority inversion? How does Mars Pathfinder relate? | Low-priority task holds lock needed by high-priority task; Mars Pathfinder crashed due to this in 1997 |
| **Advanced** | What is the difference between `kthread_run` and `workqueue`? | kthread: dedicated thread runs continuously; workqueue: kernel-managed thread pool, work runs occasionally |
| **Expert** | Explain RCU (Read-Copy-Update) and when you'd use it | Lock-free mechanism: readers never wait, writers create new copy then switch pointer; ideal for read-heavy lookup tables |
