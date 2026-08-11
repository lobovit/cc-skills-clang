---
name: clang-memory-safety
description: "C/Clang memory safety patterns — RAII-like resource management, ownership semantics, safe memory reuse, leak prevention, and Valgrind/ASan integration. Use when managing heap allocations, implementing custom allocators, preventing leaks, tracking ownership, or debugging memory issues in C code."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "♻"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

**Persona:** You are a C memory safety engineer. You treat every allocation as a liability and every free as a contract — clear ownership, explicit lifetimes, zero leaks.

# C Memory Safety

## Core Principles

1. **Every allocation has exactly one owner** — document it
2. **Free in reverse order of allocation** — predictable cleanup
3. **Set pointers to NULL after free** — prevent use-after-free
4. **Never free stack memory** — only free what was malloc'd
5. **Check before free** — NULL is safe but explicit checks document intent

## Ownership Model

C has no ownership system. You must impose one:

```c
// Ownership documentation via comments
/**
 * process_file - Read and process a file.
 * @path: path to file (NOT owned, caller retains)
 * @out: output buffer (OWNED by caller, must be freed)
 * @out_len: [in/out] buffer size in, bytes written out
 *
 * The caller owns *out and must free() it when done.
 * Returns 0 on success, negative errno on failure.
 * On failure, *out is set to NULL.
 */
int process_file(const char *path, char **out, size_t *out_len)
{
    FILE *f = fopen(path, "r");  // f is locally owned
    if (!f) return -errno;

    // ... read file ...

    *out = malloc(size);
    if (!*out) {
        fclose(f);
        return -ENOMEM;
    }

    // ... fill *out ...

    fclose(f);  // release local ownership
    return 0;
}
```

### Ownership Annotations

```c
// Transfer ownership (caller must free)
char *create_string(void) __attribute__((malloc));

// Caller does NOT own (library manages lifetime)
const char *get_cached_value(const char *key);
```

## Safe Free Patterns

```c
// Always safe — free(NULL) is a no-op
free(ptr);
ptr = NULL;

// Explicit NULL check (documents intent)
if (ptr != NULL) {
    free(ptr);
    ptr = NULL;
}

// Cleanup function for complex types
void entry_free(entry_t *e)
{
    if (e == NULL) return;
    free(e->key);
    free(e->value);
    free(e);
}
```

## Preventing Memory Leaks

### Struct with Allocated Members

```c
typedef struct {
    char *name;
    char *data;
    int *counts;
} packet_t;

void packet_destroy(packet_t *pkt)
{
    if (pkt == NULL) return;
    free(pkt->name);
    free(pkt->data);
    free(pkt->counts);
    free(pkt);
}

// Constructor
packet_t *packet_create(const char *name, size_t n)
{
    packet_t *pkt = calloc(1, sizeof(*pkt));
    if (!pkt) return NULL;

    pkt->name = strdup(name);
    if (!pkt->name) goto fail;

    pkt->data = malloc(n);
    if (!pkt->data) goto fail;

    pkt->counts = calloc(n, sizeof(int));
    if (!pkt->counts) goto fail;

    return pkt;

fail:
    packet_destroy(pkt);
    return NULL;
}
```

### Cleanup with goto

```c
int complex_operation(void)
{
    FILE *f = NULL;
    char *buf = NULL;
    sqlite3 *db = NULL;
    int ret = 0;

    f = fopen("input.txt", "r");
    if (!f) { ret = -errno; goto out; }

    buf = malloc(4096);
    if (!buf) { ret = -ENOMEM; goto out_f; }

    if (sqlite3_open(":memory:", &db) != SQLITE_OK) {
        ret = -EIO;
        goto out_buf;
    }

    /* ... work ... */

out_db:
    sqlite3_close(db);
out_buf:
    free(buf);
out_f:
    fclose(f);
out:
    return ret;
}
```

## Valgrind Integration

```bash
# Memory leak detection
valgrind --leak-check=full --show-leak-kinds=all ./app

# Suppression file for known leaks
valgrind --suppressions=known_leaks.supp ./app

# Track file descriptors
valgrind --track-fds=yes ./app
```

### Reading Valgrind Output

```text
==12345== LEAK SUMMARY:
==12345==    definitely lost: 100 bytes in 1 blocks
==12345==    indirectly lost: 0 bytes in 0 blocks
==12345==      possibly lost: 0 bytes in 0 blocks
==12345==    still reachable: 200 bytes in 2 blocks

==12345== 100 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x...: malloc (vg_replace_malloc.c:...)
==12345==    by 0x...: main (main.c:15)
```

- `definitely lost`: real leak, must fix
- `possibly lost`: may be a leak, investigate
- `still reachable`: allocated but not freed (often OK for singletons)

## Custom Allocators

### Arena Allocator

Allocate many small objects, free all at once:

```c
typedef struct {
    char *base;
    size_t used;
    size_t capacity;
} arena_t;

void *arena_alloc(arena_t *a, size_t size, size_t align)
{
    size_t aligned = (a->used + align - 1) & ~(align - 1);
    if (aligned + size > a->capacity) return NULL;
    void *ptr = a->base + aligned;
    a->used = aligned + size;
    return ptr;
}

void arena_reset(arena_t *a)
{
    a->used = 0;  // all allocations freed at once
}
```

### Pool Allocator

Fixed-size object pools:

```c
typedef struct pool_block {
    struct pool_block *next;
} pool_block_t;

typedef struct {
    pool_block_t *free_list;
    size_t elem_size;
    char *memory;
    size_t size;
} pool_t;

void *pool_alloc(pool_t *p)
{
    if (p->free_list == NULL) return NULL;
    pool_block_t *block = p->free_list;
    p->free_list = block->next;
    return block;
}

void pool_free(pool_t *p, void *ptr)
{
    pool_block_t *block = ptr;
    block->next = p->free_list;
    p->free_list = block;
}
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Freeing stack memory | Only `free()` what was `malloc`'d |
| Use-after-free | Set pointer to NULL after free |
| Double free | Track ownership; only owner frees |
| Memory leak on error | Use goto cleanup or RAII patterns |
| Forgetting to free struct members | Write destroy functions for every struct |
| Not checking malloc return | Always check for NULL |
| Mismatched alloc/free | `malloc` pairs with `free`, `calloc` with `free`, `strdup` with `free` |

## Cross-References

- → See `lobor/cc-skills-clang@clang-safety` skill for defensive coding patterns
- → See `lobor/cc-skills-clang@clang-sanitizers` skill for runtime memory error detection
- → See `lobor/cc-skills-clang@clang-design-patterns` skill for RAII-like patterns
- → See `lobor/cc-skills-clang@clang-error-handling` skill for cleanup on error paths
