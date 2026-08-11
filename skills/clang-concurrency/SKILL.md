---
name: clang-concurrency
description: "C/Clang concurrency patterns — pthreads, mutexes, read-write locks, condition variables, atomics (C11), thread pools, producer-consumer, and race condition prevention. Use when writing or reviewing concurrent C code involving threads, locks, atomics, or when diagnosing data races and deadlocks."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "⚡"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a C concurrency engineer. You assume every thread is a liability until proven necessary — correctness and leak-freedom come before performance.

# C Concurrency

## Core Principles

1. **Every thread must have a clear exit** — without join or detach, threads leak
2. **Protect shared state** — data races are undefined behavior in C11
3. **Prefer C11 atomics over volatile** — `volatile` does NOT prevent data races
4. **Lock hierarchies** — always acquire locks in the same order to prevent deadlock
5. **Test with ThreadSanitizer** — `clang -fsanitize=thread`

## C11 Atomics

C11 provides standard atomics. Use them instead of `volatile` or platform-specific intrinsics:

```c
#include <stdatomic.h>

// Atomic counter
atomic_int counter = ATOMIC_VAR_INIT(0);

// Increment atomically
atomic_fetch_add(&counter, 1);

// Read atomically
int val = atomic_load(&counter);

// Compare-and-swap
int expected = 0;
atomic_compare_exchange_strong(&counter, &expected, 1);
```

### Memory Ordering

```c
// Relaxed — no ordering guarantees
atomic_store_explicit(&flag, 1, memory_order_relaxed);

// Release — all prior writes visible after this store
atomic_store_explicit(&flag, 1, memory_order_release);

// Acquire — all subsequent reads see prior release writes
int val = atomic_load_explicit(&flag, memory_order_acquire);

// Acquire-Release — used in CAS loops
atomic_compare_exchange_strong_explicit(&val, &expected, new_val,
    memory_order_acq_rel, memory_order_acquire);
```

**When to use which:**
| Ordering | Use When |
| --- | --- |
| `relaxed` | Counters, statistics — no ordering needed |
| `release` | Publishing data — "all my writes are done" |
| `acquire` | Consuming data — "see all writes from the publisher" |
| `acq_rel` | Read-modify-write — both acquire and release |
| `seq_cst` | Default — strongest ordering, safest |

## pthreads

```c
#include <pthread.h>

void *worker(void *arg)
{
    int id = *(int *)arg;
    printf("Worker %d starting\n", id);
    // ... work ...
    return NULL;
}

int main(void)
{
    pthread_t threads[4];
    int ids[4];

    for (int i = 0; i < 4; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }

    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### Error Handling

```c
int ret = pthread_create(&thread, NULL, worker, arg);
if (ret != 0) {
    fprintf(stderr, "pthread_create: %s\n", strerror(ret));
    return -ret;
}
```

## Mutexes

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

// Lock/unlock
pthread_mutex_lock(&lock);
shared_data++;
pthread_mutex_unlock(&lock);

// Trylock (non-blocking)
if (pthread_mutex_trylock(&lock) == 0) {
    // acquired
    pthread_mutex_unlock(&lock);
} else {
    // someone else holds it
}
```

### Read-Write Locks

Many readers, few writers:

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

// Reader (multiple concurrent readers OK)
pthread_rwlock_rdlock(&rwlock);
int val = shared_value;
pthread_rwlock_unlock(&rwlock);

// Writer (exclusive)
pthread_rwlock_wrlock(&rwlock);
shared_value = new_val;
pthread_rwlock_unlock(&rwlock);
```

## Condition Variables

Wait for events:

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
bool ready = false;

// Waiter
pthread_mutex_lock(&mutex);
while (!ready) {
    pthread_cond_wait(&cond, &mutex);
}
pthread_mutex_unlock(&mutex);

// Signal
pthread_mutex_lock(&mutex);
ready = true;
pthread_cond_signal(&cond);   // wake one waiter
// pthread_cond_broadcast(&cond);  // wake all waiters
pthread_mutex_unlock(&mutex);
```

## Thread Pool Pattern

```c
typedef void (*task_fn)(void *arg);

typedef struct {
    task_fn fn;
    void *arg;
} task_t;

typedef struct {
    pthread_t *threads;
    size_t num_threads;
    task_t *queue;
    size_t queue_head;
    size_t queue_tail;
    size_t queue_size;
    pthread_mutex_t mutex;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
    bool shutdown;
} thread_pool_t;

void *pool_worker(void *arg)
{
    thread_pool_t *pool = arg;

    while (true) {
        pthread_mutex_lock(&pool->mutex);
        while (pool->queue_head == pool->queue_tail && !pool->shutdown) {
            pthread_cond_wait(&pool->not_empty, &pool->mutex);
        }
        if (pool->shutdown) {
            pthread_mutex_unlock(&pool->mutex);
            break;
        }
        task_t task = pool->queue[pool->queue_head % pool->queue_size];
        pool->queue_head++;
        pthread_cond_signal(&pool->not_full);
        pthread_mutex_unlock(&pool->mutex);

        task.fn(task.arg);
    }
    return NULL;
}
```

## Race Condition Patterns

### Check-Then-Act

```c
// Vulnerable — TOCTOU race
if (file_exists(path)) {
    // file may be deleted between check and open
    int fd = open(path, O_RDONLY);
}

// Fixed — atomic check-and-act
int fd = open(path, O_RDONLY | O_CREAT, 0644);
if (fd < 0 && errno != EEXIST) {
    return -errno;
}
```

### Lazy Initialization

```c
// Vulnerable — race on first access
static connection_t *global_conn = NULL;

connection_t *get_connection(void)
{
    if (global_conn == NULL) {
        global_conn = create_connection();  // race
    }
    return global_conn;
}

// Fixed — pthread_once
static pthread_once_t conn_once = PTHREAD_ONCE_INIT;

static void init_connection(void)
{
    global_conn = create_connection();
}

connection_t *get_connection(void)
{
    pthread_once(&conn_once, init_connection);
    return global_conn;
}
```

## Deadlock Prevention

| Strategy | How |
| --- | --- |
| Lock ordering | Always acquire locks A before B |
| Trylock + backoff | If trylock fails, release held locks, retry |
| Timeout | Use `pthread_mutex_timedlock` |
| Avoid nested locks | Prefer atomic operations for simple cases |

## Compiling with pthreads

```bash
clang -pthread -o app src.c
```

## Cross-References

- → See `lobor/cc-skills-clang@clang-safety` skill for thread-safety of data structures
- → See `lobor/cc-skills-clang@clang-sanitizers` skill for ThreadSanitizer configuration
- → See `lobor/cc-skills-clang@clang-testing` skill for testing concurrent code
- → See `lobor/cc-skills-clang@clang-performance` skill for lock contention optimization
