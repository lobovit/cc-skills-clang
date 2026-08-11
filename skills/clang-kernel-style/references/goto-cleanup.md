# goto Cleanup — Why It Works

## The Problem with Multiple Exit Points

```c
// BAD: multiple returns, easy to leak
int init(struct context *ctx)
{
        ctx->a = allocate_a();
        if (!ctx->a) return -1;

        ctx->b = allocate_b();
        if (!ctx->b) return -1;  // LEAK: ctx->a

        ctx->c = allocate_c();
        if (!ctx->c) return -1;  // LEAK: ctx->a, ctx->b

        return 0;
}
```

Every new resource adds another leak path. reviewers must check every return.

## The goto Solution

```c
// GOOD: single exit, reverse cleanup
int init(struct context *ctx)
{
        int ret;

        ctx->a = allocate_a();
        if (!ctx->a) {
                ret = -1;
                goto err_a;
        }

        ctx->b = allocate_b();
        if (!ctx->b) {
                ret = -1;
                goto err_b;
        }

        ctx->c = allocate_c();
        if (!ctx->c) {
                ret = -1;
                goto err_c;
        }

        return 0;

err_c:
        free(ctx->b);
err_b:
        free(ctx->a);
err_a:
        return ret;
}
```

## Why It's Better

1. **One exit point** — easy to add cleanup without missing paths
2. **Reverse order** — matches initialization order, natural to reason about
3. **No nested ifs** — flat code, easy to scan
4. **Compiler optimizes** — no overhead compared to multiple returns

## Common Patterns

### File/resource cleanup

```c
FILE *f = fopen(path, "r");
if (!f) return -errno;

// ... work ...

fclose(f);  // only one place to close
```

### Multi-resource with shared state

```c
ret = init_a(&ctx->a);
if (ret) goto err_a;

ret = init_b(&ctx->a, &ctx->b);  // needs a
if (ret) goto err_b;

return 0;

err_b:
        cleanup_a(&ctx->a);
err_a:
        return ret;
```

## When NOT to Use

- Simple functions with one resource
- When the cleanup is just `return` (no side effects)
- In C++ where RAII/destructors handle this naturally
