---
name: clang-naming
description: "C/Clang naming conventions — covers functions, variables, macros, constants, enums, typedefs, struct/union tags, file naming, and header guards. Use when writing new C code, reviewing or refactoring, choosing between naming alternatives (init vs initialize, is_ready vs ready, ERR_NOT_FOUND vs NOT_FOUND), debating C naming best practices, or asking about macro naming vs function naming. Also trigger when the user mentions snake_case vs camelCase, ALL_CAPS macros, or Hungarian notation in C."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🏷"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(clang-tidy:*) Bash(git:*) Agent
---

# C Naming Conventions

C favors short, readable names. The standard library and Linux kernel both use `snake_case` for functions and variables, `UPPER_CASE` for macros and constants. Follow these conventions for consistency.

> "Programs must be written for people to read, and only incidentally for machines to execute." — Abelson & Sussman

## Quick Reference

| Element | Convention | Example |
| --- | --- | --- |
| Function | `snake_case` | `compute_hash`, `init_buffer` |
| Variable | `snake_case` | `item_count`, `is_valid` |
| Macro (function-like) | `UPPER_SNAKE_CASE` | `MAX(a, b)`, `ARRAY_SIZE(x)` |
| Macro (constant) | `UPPER_SNAKE_CASE` | `BUF_SIZE`, `MAX_RETRIES` |
| Enum constant | `UPPER_SNAKE_CASE` | `ERR_NOT_FOUND`, `STATE_ACTIVE` |
| Enum type | `snake_case` | `enum log_level` |
| Struct/union tag | `snake_case` | `struct file_entry`, `union raw_value` |
| Typedef | `snake_case` (or `_t` suffix) | `file_entry_t`, `buffer_t` |
| File (source) | `snake_case.c` | `hash_table.c` |
| File (header) | `snake_case.h` | `hash_table.h` |
| Header guard | `PROJECT_PATH_FILE_H` | `MYLIB_HASH_TABLE_H` |
| Static/private function | `snake_case` (same style) | `validate_internal` |
| Global variable | `snake_case` with module prefix | `ht_total_lookups` |
| Parameter | `snake_case` | `max_retries`, `input_data` |

## Functions

Functions use `snake_case`. Names should describe **what** the function does:

```c
// Good — clear verb + noun
int hash_table_insert(hash_table_t *ht, const char *key, void *value);
int buffer_append(buffer_t *buf, const uint8_t *data, size_t len);
bool file_exists(const char *path);

// Bad — vague or abbreviates away meaning
int do_stuff(hash_table_t *ht, char *k, void *v);
int buf_app(buffer_t *b, uint8_t *d, size_t l);
```

**Return convention**:
- `0` on success, negative errno on failure (Linux kernel style)
- Boolean functions: `bool` return type
- Pointer returns: `NULL` on failure

## Variables

Variables use `snake_case`. Prefer descriptive names:

```c
// Good
int active_connection_count;
const char *config_path;

// Bad — cryptic abbreviations
int acc;
const char *cp;
```

**Loop variables** are the exception — short names are fine:
```c
for (int i = 0; i < n; i++) { /* ... */ }
for (size_t j = 0; j < len; j++) { /* ... */ }
```

## Macros

Macros use `UPPER_SNAKE_CASE`:

```c
#define MAX_BUFFER_SIZE  (4096)
#define ARRAY_SIZE(arr)  (sizeof(arr) / sizeof((arr)[0]))
#define UNUSED(x)        ((void)(x))
```

Macro parameters should be parenthesized to prevent precedence bugs:
```c
// Good — parenthesized
#define SQUARE(x) ((x) * (x))

// Bad — unparenthesized
#define SQUARE(x) (x * x)  // SQUARE(1+2) gives 5, not 9
```

## Constants

Constants prefer `enum` over `#define` when possible (type-safe, debugger-visible):

```c
// Good — enum constant (type-safe, debuggable)
enum {
    MAX_RETRIES    = 3,
    BUF_SIZE       = 4096,
    DEFAULT_PORT   = 8080,
};

// Also acceptable for values that must be preprocessor-visible
#define MAX_PATH_LEN 256
```

## Enum Naming

Enum type names use `snake_case`. Enum values use the type name as prefix:

```c
// Good — prefixed values
enum connection_state {
    CONN_STATE_NONE = 0,
    CONN_STATE_CONNECTING,
    CONN_STATE_CONNECTED,
    CONN_STATE_CLOSING,
};

// Bad — unprefixed (collides at file scope)
enum connection_state {
    NONE,       // collides with POSIX NULL-like macros
    CONNECTING,
    CONNECTED,
};
```

Always start with a zero sentinel:
```c
enum error_code {
    ERR_NONE = 0,   // catch uninitialized variables
    ERR_INVALID,
    ERR_TIMEOUT,
};
```

## Typedefs

Typedefs are common in C for convenience. Use `_t` suffix:

```c
typedef struct hash_table hash_table_t;
typedef void (*event_handler_t)(void *ctx, int event);
typedef int (*comparator_t)(const void *a, const void *b);
```

For opaque types, typedef the struct directly:
```c
// In public header — opaque pointer
typedef struct database database_t;

// In private header — actual definition
struct database {
    sqlite3 *db;
    char *path;
};
```

## File Naming

Source files use `snake_case.c`, headers use `snake_case.h`. Name files after the primary type or module they contain:

```c
// Good — named after the module/type
hash_table.c / hash_table.h
event_loop.c / event_loop.h
config_parser.c / config_parser.h

// Bad — generic names
utils.c / utils.h
helpers.c / helpers.h
common.c / common.h
```

## Header Guards

Use `PROJECT_DIR_FILENAME_H` format:

```c
// In mylib/src/hash_table.h
#ifndef MYLIB_SRC_HASH_TABLE_H
#define MYLIB_SRC_HASH_TABLE_H

/* ... */

#endif /* MYLIB_SRC_HASH_TABLE_H */
```

Or use `#pragma once` (non-standard but widely supported by Clang):
```c
#pragma once
```

## Macro vs Function Decision

Use macros when:
- You need type generic behavior in C89/C99 (before `_Generic`)
- The expression must be compile-time evaluable (`sizeof`, array size)
- You need statement expressions (`({ ... })`) for multi-statement macros

Use functions when:
- Type safety matters
- Debuggability matters (macros expand inline, hard to set breakpoints)
- The logic is complex enough to warrant a named stack frame

```c
// Good macro — type-generic min
#define MIN(a, b) ((a) < (b) ? (a) : (b))

// Good function — complex logic
int hash_table_resize(hash_table_t *ht, size_t new_size);
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| `camelCase` functions | Use `snake_case` — consistent with C standard library and POSIX |
| `ALL_CAPS` variables | Reserve for macros and enum constants only |
| `g_` prefix on globals | Use module prefix instead: `ht_count`, `srv_fd` |
| Generic names (`data`, `info`, `temp`) | Be specific: `packet_data`, `connection_info`, `read_temp` |
| `ret` as catch-all return | Name meaningfully: `err`, `bytes_read`, `status` |
| `#define` for typed constants | Use `enum` when the value is compile-time constant |
| Unparenthesized macro params | Always parenthesize: `#define M(x) ((x) * 2)` |

## Cross-References

- → See `lobor/cc-skills-clang@clang-code-style` skill for broader formatting decisions
- → See `lobor/cc-skills-clang@clang-format` skill for automated enforcement
- → See `lobor/cc-skills-clang@clang-safety` skill for naming patterns that prevent bugs
