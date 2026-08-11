---
name: clang-error-handling
description: "C/Clang error handling best practices — return codes, errno propagation, goto-cleanup patterns, error context, error messages, and resource cleanup. Apply when creating, propagating, or handling errors in C code. Covers Linux kernel style (negative errno), POSIX style, and structured error handling with error contexts."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "⚠"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

**Persona:** You are a C reliability engineer. You treat every error as an event that must either be handled or propagated — silent failures and leaked resources are equally unacceptable.

# C Error Handling

C lacks exceptions. Error handling relies on return values, `errno`, and disciplined cleanup. Done well, it produces robust code. Done poorly, it leaks resources and silently corrupts state.

## Core Principles

1. **Every return value MUST be checked** — never ignore errors
2. **Clean up on every exit path** — use `goto cleanup` or structured lifetimes
3. **Propagate error context** — callers need to know what failed and why
4. **Use negative errno for library functions** — consistent with Linux kernel/POSIX
5. **Never silently swallow errors** — log or propagate, never both

## Return Value Conventions

### Linux Kernel Style (negative errno)

The standard for C libraries and kernel code:

```c
int process_data(const uint8_t *data, size_t len)
{
    if (data == NULL) {
        return -EINVAL;
    }

    if (len > MAX_SIZE) {
        return -E2BIG;
    }

    /* success */
    return 0;
}
```

Common errno values: `EINVAL` (bad argument), `ENOMEM` (out of memory), `EIO` (I/O error), `ENOENT` (not found), `EACCES` (permission denied), `EAGAIN` (try again).

### Boolean Success

For simple pass/fail where errno isn't useful:
```c
bool validate_config(const config_t *cfg);
```

### Pointer Returns

`NULL` on failure:
```c
hash_table_t *hash_table_create(size_t initial_size);
```

## The goto Cleanup Pattern

The idiomatic C way to handle errors without leaking resources. **This is not bad practice** — it's the standard pattern in Linux, SQLite, and most mature C codebases:

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

    /* ... do work with f and buf ... */

out:
    free(buf);
    if (f) {
        fclose(f);
    }
    return ret;
}
```

**Rules for goto cleanup:**
- Always jump to the same label (usually `out`, `cleanup`, or `exit`)
- Free/destroy in reverse order of acquisition
- Check pointers before freeing (freeing NULL is safe but explicit checks document intent)
- Initialize all cleanup variables to their "empty" state at function start

## Error Context

When propagating errors, add context so callers know what failed:

```c
// Good — context tells the story
int ret = parse_header(buf, len, &hdr);
if (ret != 0) {
    fprintf(stderr, "failed to parse header: %s\n", strerror(-ret));
    return ret;
}

// Bad — no context
parse_header(buf, len, &hdr);
```

For library code, use a structured error type:
```c
typedef struct {
    int code;           // negative errno
    const char *file;   // source file
    int line;           // source line
    char msg[128];      // human-readable context
} error_t;

#define ERR_SET(err, c, fmt, ...) do { \
    (err)->code = (c);                \
    (err)->file = __FILE__;           \
    (err)->line = __LINE__;           \
    snprintf((err)->msg, sizeof((err)->msg), fmt, ##__VA_ARGS__); \
} while (0)
```

## errno Handling

`errno` is thread-local but fragile — it can be overwritten by any function call:

```c
// Bad — errno overwritten by fprintf
FILE *f = fopen(path, "r");
if (f == NULL) {
    fprintf(stderr, "open failed\n");  // may overwrite errno
    return -errno;                      // wrong errno!
}

// Good — save errno immediately
FILE *f = fopen(path, "r");
if (f == NULL) {
    int saved_errno = errno;
    fprintf(stderr, "open failed: %s\n", strerror(saved_errno));
    return -saved_errno;
}
```

## Resource Cleanup Patterns

### RAII-like with goto

```c
int complex_operation(void)
{
    resource_a_t *a = NULL;
    resource_b_t *b = NULL;
    resource_c_t *c = NULL;
    int ret = 0;

    a = create_a();
    if (a == NULL) { ret = -ENOMEM; goto out_c; }

    b = create_b(a);
    if (b == NULL) { ret = -EIO; goto out_b; }

    c = create_c(b);
    if (c == NULL) { ret = -ENOMEM; goto out_a; }

    /* ... work ... */

out_a:
    destroy_b(b);
out_b:
    destroy_a(a);
out_c:
    return ret;
}
```

### Deferred Cleanup (C11 `_Cleanup` or compiler extensions)

```c
// Using __attribute__((cleanup)) (GCC/Clang extension)
static void auto_free(void *p)
{
    free(*(void **)p);
}

#define AUTO_FREE __attribute__((cleanup(auto_free)))

void process(void)
{
    AUTO_FREE char *buf = malloc(1024);
    AUTO_FREE FILE *f = fopen("data.txt", "r");

    // buf and f are automatically freed when they go out of scope
}
```

## Error Reporting

For user-facing code, translate internal errors to user-friendly messages:

```c
// Library: returns negative errno
int ret = database_connect(db, url);

// Application: translates to user message
if (ret == -EACCES) {
    fprintf(stderr, "Error: permission denied connecting to %s\n", url);
} else if (ret == -ENOENT) {
    fprintf(stderr, "Error: database not found at %s\n", url);
} else if (ret != 0) {
    fprintf(stderr, "Error: connection failed (%s)\n", strerror(-ret));
}
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Ignoring `malloc` return | Always check for NULL — out-of-memory is real |
| Ignoring `fclose` return | Flush errors are real — check and propagate |
| `errno` after intervening call | Save `errno` immediately after the failing call |
| `goto` across variable declarations | In C99+, declarations with initializers prevent this — initialize at top |
| Logging AND returning | Pick one: either log and handle, or propagate to caller — not both |
| Calling `exit()` in library code | Libraries should return errors; only `main()` should call `exit()` |
| Missing cleanup on error paths | Audit every `return` — ensure all acquired resources are released |

## Cross-References

- → See `lobor/cc-skills-clang@clang-safety` skill for nil pointer handling and defensive coding
- → See `lobor/cc-skills-clang@clang-testing` skill for testing error paths
- → See `lobor/cc-skills-clang@clang-security` skill for error information leakage
