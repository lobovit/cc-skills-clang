---
name: clang-safety
description: "Defensive C/Clang coding to prevent buffer overflows, use-after-free, null pointer dereferences, integer overflow, and other memory corruption bugs. Use when encountering null pointer crashes, buffer overflow risks, integer truncation, resource lifecycle issues (double-free, leak), or defensive copying. Also use when reviewing code for undefined behavior, uninitialized reads, or signed integer overflow."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🛡"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(clang-tidy:*) Bash(git:*) Agent
---

**Persona:** You are a defensive C engineer. You treat every unvalidated assumption about buffer sizes, pointer validity, and numeric ranges as a potential crash or security vulnerability.

# C Safety: Correctness & Defensive Coding

C provides no built-in safety nets. Every buffer access, pointer dereference, and integer operation can fail silently or corrupt memory. Defensive coding is the first line of defense — before sanitizers, before fuzzing, before runtime checks.

## Core Principles

1. **Validate every input** — at function boundaries, before use
2. **Check every return value** — especially `malloc`, file I/O, and system calls
3. **Bound every buffer access** — never trust lengths passed by callers
4. **Initialize everything** — uninitialized memory is undefined behavior
5. **Design for safe failure** — fail closed, not open

## Null Pointer Safety

Null pointer dereference is the most common crash in C:

```c
// Good — check before use
void process(const struct config *cfg)
{
    if (cfg == NULL) {
        return;
    }

    if (cfg->data == NULL || cfg->len == 0) {
        return;
    }

    /* safe to use cfg->data */
}

// Bad — unchecked dereference
void process(const struct config *cfg)
{
    // segfault if cfg is NULL
    printf("%zu items\n", cfg->len);
}
```

**Functions that return NULL on error:**
```c
// Always check malloc, calloc, strdup, fopen, etc.
char *buf = malloc(size);
if (buf == NULL) {
    return -ENOMEM;
}
```

## Buffer Safety

### Bounds Checking

Never access an array without verifying the index is within bounds:

```c
// Good — bounds checked
int get_item(const int *arr, size_t n, size_t index, int *out)
{
    if (arr == NULL || out == NULL) {
        return -EINVAL;
    }
    if (index >= n) {
        return -EINVAL;
    }
    *out = arr[index];
    return 0;
}

// Bad — unchecked index
int get_item(const int *arr, size_t n, size_t index)
{
    return arr[index];  // OOB if index >= n
}
```

### String Safety

Use bounded string functions. Never use `gets()`, `strcpy()` without length, or `sprintf()` without bounds:

```c
// Good — bounded
char dst[64];
snprintf(dst, sizeof(dst), "%s:%d", host, port);
strncat(dst, suffix, sizeof(dst) - strlen(dst) - 1);

// Bad — unbounded
char dst[64];
sprintf(dst, "%s:%d", host, port);  // overflow if host is long
strcpy(dst, src);                   // overflow if src > 64 bytes
```

### memcpy/memmove Safety

Always validate sizes before memory operations:

```c
// Good — validate before copy
if (src_len > dst_capacity) {
    return -E2BIG;
}
memcpy(dst, src, src_len);

// Good — use memmove for overlapping regions
memmove(buf + offset, buf, remaining);
```

## Integer Safety

### Overflow Detection

Signed integer overflow is **undefined behavior** in C. Check before arithmetic:

```c
// Good — check before addition
if (a > INT_MAX - b) {
    return -EOVERFLOW;
}
int sum = a + b;

// Good — check before multiplication
if (a != 0 && b > INT_MAX / a) {
    return -EOVERFLOW;
}
int product = a * b;
```

### Unsigned Wraparound

Unsigned overflow is defined (wraps), but usually still a bug:

```c
// Good — detect underflow
if (a < b) {
    return -EINVAL;  // prevent underflow
}
size_t diff = a - b;
```

### Truncation

Casting between integer sizes can silently truncate:

```c
// Bad — truncation if len > UINT32_MAX
uint32_t small_len = (uint32_t)big_len;

// Good — check before casting
if (big_len > UINT32_MAX) {
    return -E2BIG;
}
uint32_t small_len = (uint32_t)big_len;
```

## Memory Management

### Allocation Patterns

```c
// Always check allocation
void *ptr = calloc(count, elem_size);
if (ptr == NULL) {
    return -ENOMEM;
}

// Detect integer overflow in allocation size
if (count > SIZE_MAX / elem_size) {
    return -EOVERFLOW;
}
void *ptr = calloc(count, elem_size);
```

### Free Safety

`free(NULL)` is safe, but document intent:

```c
// Explicit NULL check before free (documents intent)
if (ptr != NULL) {
    free(ptr);
    ptr = NULL;  // prevent use-after-free
}
```

### Double-Free Prevention

Set pointers to NULL after free:

```c
void cleanup(struct context *ctx)
{
    free(ctx->buffer);
    ctx->buffer = NULL;

    free(ctx->metadata);
    ctx->metadata = NULL;
}
```

## Uninitialized Memory

Uninitialized variables contain garbage — using them is undefined behavior:

```c
// Good — initialized at declaration
int result = 0;
char buf[256] = {0};
struct stat st = {0};

// Bad — uninitialized
int result;         // contains garbage
char buf[256];      // contains stack residue
```

For structs allocated with `malloc`, use `calloc` or `memset`:
```c
struct entry *e = calloc(1, sizeof(*e));
// all fields are zero-initialized
```

## Compiler-Assisted Safety

### Warnings

Enable maximum warnings with Clang:

```bash
clang -Wall -Wextra -Werror -Wpedantic \
      -Wshadow -Wconversion -Wsign-conversion \
      -Wformat=2 -Wnull-dereference \
      -c src.c
```

### Static Analysis

```bash
# Clang's built-in static analyzer
clang --analyze src.c

# clang-tidy
clang-tidy src.c -- -std=c11
```

### Sanitizers for Testing

```bash
# AddressSanitizer — buffer overflows, use-after-free, double-free
clang -fsanitize=address -fno-omit-frame-pointer -g src.c

# UBSan — undefined behavior (signed overflow, null deref, etc.)
clang -fsanitize=undefined -g src.c

# ThreadSanitizer — data races
clang -fsanitize=thread -g src.c
```

→ See `lobor/cc-skills-clang@clang-sanitizers` skill for full sanitizer guide.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Unchecked `malloc` return | Always check for NULL — OOM is real |
| `strcpy` without bounds | Use `snprintf` or `strlcpy` with `sizeof(dst)` |
| Signed overflow assumed defined | Check before arithmetic — signed overflow is UB |
| Uninitialized struct fields | Use `calloc` or `= {0}` initializer |
| Missing NULL check before deref | Validate pointers at function entry |
| `free` without setting to NULL | Set pointer to NULL after free to prevent use-after-free |
| Integer truncation on cast | Check bounds before narrowing casts |
| Buffer declared too small | Use `sizeof(buf)` consistently, never hardcode sizes |
| Ignoring `snprintf` return value | If `snprintf` returns ≥ `sizeof(buf)`, output was truncated |

## Cross-References

- → See `lobor/cc-skills-clang@clang-security` skill for security-relevant safety issues
- → See `lobor/cc-skills-clang@clang-sanitizers` skill for runtime detection
- → See `lobor/cc-skills-clang@clang-testing` skill for testing error paths
- → See `lobor/cc-skills-clang@clang-error-handling` skill for cleanup patterns
