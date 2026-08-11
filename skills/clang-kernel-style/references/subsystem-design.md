# Subsystem Design — Clear API Boundaries

## The Principle

Every module has a public API (what others can use) and private implementation (what changes freely). This separation allows refactoring internals without breaking callers.

## Structure

```
module/
├── module.h       # Public API — what other modules include
├── module.c       # Implementation — may contain static helpers
└── module_internal.h  # Optional: shared between .c files in the module
```

## Public API Design

```c
// module.h — public interface
#ifndef MODULE_H
#define MODULE_H

// Opaque pointer — callers cannot touch internals
struct module_ctx;

// Lifecycle
struct module_ctx *module_create(void);
void module_destroy(struct module_ctx *ctx);

// Operations
int module_process(struct module_ctx *ctx, const void *data, size_t len);
int module_flush(struct module_ctx *ctx);

#endif
```

```c
// module.c — implementation
#include "module.h"

struct module_ctx {
        int fd;
        char *buffer;
        size_t capacity;
        // callers never see these
};

struct module_ctx *module_create(void)
{
        struct module_ctx *ctx = calloc(1, sizeof(*ctx));
        if (!ctx) return NULL;

        ctx->fd = -1;
        ctx->capacity = 4096;
        ctx->buffer = malloc(ctx->capacity);
        if (!ctx->buffer) {
                free(ctx);
                return NULL;
        }
        return ctx;
}

// static helper — not in header, not exported
static int ensure_capacity(struct module_ctx *ctx, size_t needed)
{
        if (ctx->capacity >= needed) return 0;
        // ... realloc logic ...
        return 0;
}
```

## Naming Convention

Pattern: `subsystem_verb_noun()`

| Good | Bad |
| --- | --- |
| `netdev_register()` | `register_device_network()` |
| `cache_invalidate()` | `invalidate_the_cache()` |
| `pool_alloc()` | `alloc_from_pool()` |

## Why It Matters

1. **Opaque pointers** — callers cannot corrupt internal state
2. **Stable ABI** — internals can change without recompiling callers
3. **Testability** — public API is the test surface
4. **Boundary enforcement** — `static` functions cannot leak across modules
