---
name: clang-kernel-style
description: "Linux kernel coding style for C projects — tab indentation, goto cleanup, subsystem boundaries, reference counting. Use when adopting kernel-grade discipline for struct layout, error handling, or API design in any C codebase."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🐧"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

# Clang Kernel Style

Linux kernel coding style adapted for general C projects. Not about writing kernel code — about adopting kernel-grade discipline.

## Formatting

- **Indentation:** tabs, 80-column limit
- **Braces:** opening brace on same line for functions/control flow, next line for functions only if preferred
- **Spaces:** around operators, after keywords (`if`, `for`, `while`), not after function names
- **Naming:** `lower_snake_case` for functions/variables, `UPPER_SNAKE` for macros, `struct_name` for types

```c
// Kernel style
static int device_open(struct inode *inode, struct file *file)
{
        struct device_data *dev;

        dev = container_of(inode->i_cdev, struct device_data, cdev);
        if (!dev)
                return -ENODEV;

        file->private_data = dev;
        return 0;
}
```

## goto Cleanup

Single exit point. Cleanup in reverse initialization order. Easy to audit, hard to leak.

```c
int complex_init(struct context *ctx)
{
        int ret;

        ctx->buffer = malloc(BUF_SIZE);
        if (!ctx->buffer)
                return -ENOMEM;

        ctx->log = fopen("trace.log", "w");
        if (!ctx->log) {
                ret = -errno;
                goto err_log;
        }

        ret = start_worker(ctx);
        if (ret)
                goto err_worker;

        return 0;

err_worker:
        fclose(ctx->log);
err_log:
        free(ctx->buffer);
        return ret;
}
```

**Why it matters:** [Details](references/goto-cleanup.md)

## Subsystem Boundaries

Define clear API surfaces. Internal helpers are `static`. Public functions follow `subsystem_verb_noun()` naming.

```c
// --- public API (header) ---
int netdev_register(struct net_device *dev);
void netdev_unregister(struct net_device *dev);

// --- internal (source, static) ---
static int internal_setup(struct net_device *dev);
static void internal_teardown(struct net_device *dev);
```

**Why it matters:** [Details](references/subsystem-design.md)

## Reference Counting

Manual refcounting for shared objects. Get/put pattern with release callback.

```c
struct shared_obj {
    int refcount;
    void *data;
};

void obj_get(struct shared_obj *obj)
{
        if (obj)
                obj->refcount++;
}

void obj_put(struct shared_obj *obj)
{
        if (obj && --obj->refcount == 0) {
                free(obj->data);
                free(obj);
        }
}
```

## Git Workflow

**Commit messages:** `subsystem: short summary (50 chars)`

```
parser: handle empty input gracefully

The parser previously crashed when fed a zero-length buffer.
Add a length check at entry and return -EINVAL.

Fixes: a1b2c3d4 ("parser: initial implementation")
Signed-off-by: Name <email>
```

**Useful commands:**

```bash
git add -p                    # Stage by hunk, not file
git bisect start              # Find breaking commit
git blame -w -C file.c        # Ignore whitespace, track moves
```

## Mental Model

1. What are the data structures? Design these first.
2. What's the cache behavior? Sequential > random access.
3. What's the common case? Optimize for it.
4. Can I review this in 5 minutes? Small patches, clear code.
5. What breaks if this is wrong? Systems code must be reliable.
