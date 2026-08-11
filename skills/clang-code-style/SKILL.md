---
name: clang-code-style
description: "C/Clang code style conventions — line length, brace placement, indentation, variable declarations, control flow clarity, and comment conventions. Use when writing or reviewing C code compiled with Clang, asking about style or clarity, or establishing project coding standards. Not for naming conventions (→ See `lobor/cc-skills-clang@clang-naming` skill) or formatting (→ See `lobor/cc-skills-clang@clang-format` skill)."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🎨"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(clang-tidy:*) Bash(git:*) Agent
---

**Persona:** You are a C engineer who values clarity and explicitness. You write code that reads like prose — every construct earns its place.

> "Clarity is better than cleverness." — adapted from Go Proverbs

When ignoring a rule, add a comment to the code.

# C Code Style

Style rules that require human judgment — linters handle formatting, this skill handles clarity. For naming see `lobor/cc-skills-clang@clang-naming` skill; for formatting see `lobor/cc-skills-clang@clang-format` skill.

## Line Length & Breaking

No rigid line limit, but lines beyond ~100 characters SHOULD be broken. Break at **semantic boundaries**, not arbitrary column counts. Function calls with 4+ arguments MUST use one argument per line:

```c
// Good — each argument on its own line
result = compute_value(
    input_data,
    config->timeout,
    config->retries,
    error_handler
);
```

When a function signature is too long, the real fix is often **fewer parameters** (use a config struct) rather than better line wrapping.

## Brace Placement

C projects conventionally use one of two styles. Pick one and enforce it consistently:

**K&R / Linux kernel style** (common in C):
```c
if (condition) {
    do_something();
} else {
    do_other();
}

while (running) {
    process();
}
```

**Allman style** (common in BSD):
```c
if (condition)
{
    do_something();
}
else
{
    do_other();
}
```

Functions always open on a new line:
```c
int compute_value(int x, int y)
{
    return x + y;
}
```

## Variable Declarations

Declare variables at the **top of their scope** (C89 compatibility) or **at first use** (C99+). Prefer C99+ style for clarity — declare close to first use:

```c
// Good — declare close to use
for (int i = 0; i < n; i++) {
    process(items[i]);
}

// Also acceptable for C89 compatibility
int i;
for (i = 0; i < n; i++) {
    process(items[i]);
}
```

Initialize variables at declaration when possible:
```c
int count = 0;           // explicit zero
char *ptr = NULL;        // explicit null
struct buffer buf = {0}; // zero entire struct
```

## Control Flow

### Reduce Nesting

Handle errors and edge cases first (early return). Keep the happy path at minimal indentation:

```c
// Good — early return keeps happy path flat
int process_data(const uint8_t *data, size_t len)
{
    if (data == NULL || len == 0) {
        return -EINVAL;
    }

    if (!validate_checksum(data, len)) {
        return -EChecksum;
    }

    return transform(data, len);
}
```

### Eliminate Unnecessary else

When the `if` body ends with `return`, `break`, or `continue`, drop the `else`:

```c
// Good — flat, no unnecessary nesting
int result = lookup(key);
if (result < 0) {
    return result;
}

return process(result);

// Bad — unnecessary nesting
int result = lookup(key);
if (result < 0) {
    return result;
} else {
    return process(result);
}
```

## Function Design

- Functions SHOULD be **short and focused** — one function, one job.
- Functions SHOULD have **≤4 parameters**. Beyond that, use a config struct.
- **Parameter order**: outputs/destinations last, then inputs, then context.
- Use `void` for empty parameter lists in C (not K&R style `( )`):

```c
// Good — explicit void
int init_module(void);

// Bad — ambiguous (K&R "accepts anything")
int init_module();
```

## Const Correctness

Mark pointers as `const` when the function does not modify the pointed-to data. This documents intent and lets the compiler catch bugs:

```c
// Good — const documents read-only intent
size_t count_items(const struct item *items, size_t n)
{
    size_t count = 0;
    for (size_t i = 0; i < n; i++) {
        if (items[i].active) count++;
    }
    return count;
}
```

Use `const` for:
- Pointer parameters not modified by the function
- Array parameters not modified
- Read-only struct pointers
- String literals and global config data

## Static & File Scope

Use `static` for functions and variables that are file-internal. This limits visibility and allows the compiler to optimize:

```c
// File-internal helper — not visible outside this translation unit
static int validate_input(const char *data)
{
    /* ... */
}

// File-internal table
static const struct handler handlers[] = {
    { "GET",  handle_get },
    { "POST", handle_post },
};
```

## Enum & Bitmask Conventions

Enums SHOULD have an explicit `NONE`/`UNKNOWN` zero value:

```c
enum log_level {
    LOG_NONE = 0,
    LOG_ERROR,
    LOG_WARN,
    LOG_INFO,
    LOG_DEBUG,
};
```

Bitmasks use shift notation:
```c
enum permissions {
    PERM_NONE  = 0,
    PERM_READ  = (1 << 0),
    PERM_WRITE = (1 << 1),
    PERM_EXEC  = (1 << 2),
};
```

## Error Handling Patterns

C lacks exceptions. Be explicit about error propagation:

```c
// Good — check every return value
int result = allocate_buffer(&buf, size);
if (result != 0) {
    return result;
}

// Bad — unchecked return value
allocate_buffer(&buf, size);  // silently fails
```

Use `goto` for cleanup (not considered bad practice in C):
```c
int process_file(const char *path)
{
    FILE *f = NULL;
    char *buf = NULL;
    int ret = 0;

    f = fopen(path, "r");
    if (f == NULL) {
        ret = -errno;
        goto out;
    }

    buf = malloc(BUF_SIZE);
    if (buf == NULL) {
        ret = -ENOMEM;
        goto out;
    }

    /* ... work ... */

out:
    free(buf);
    if (f) fclose(f);
    return ret;
}
```

## Comment Conventions

- **File-level comments**: purpose, author, license
- **Function comments**: what it does, parameters, return value, thread safety
- **Inline comments**: explain *why*, not *what*

```c
/*
 * compute_similarity - Compute cosine similarity between two vectors.
 * @a: first vector (must be same length as @b)
 * @b: second vector
 * @n: number of elements
 *
 * Returns similarity in range [0.0, 1.0], or -1.0 on error.
 * Thread-safe: yes (no shared state).
 */
double compute_similarity(const double *a, const double *b, size_t n)
{
    /* ... */
}
```

## Cross-References

- → See `lobor/cc-skills-clang@clang-naming` skill for identifier naming conventions
- → See `lobor/cc-skills-clang@clang-format` skill for automated formatting enforcement
- → See `lobor/cc-skills-clang@clang-safety` skill for defensive coding patterns
- → See `lobor/cc-skills-clang@clang-design-patterns` skill for struct-based design patterns
