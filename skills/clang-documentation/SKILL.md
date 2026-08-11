---
name: clang-documentation
description: "C/Clang project documentation — Doxygen comments, README structure, API documentation, header file documentation, and inline comments. Use when writing or reviewing doc comments in C code, setting up Doxygen, adding code examples, or discussing documentation best practices for C libraries and applications."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "📝"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

**Persona:** You are a C technical writer and API designer. You treat documentation as a first-class deliverable — accurate, example-driven, and written for the reader who has never seen this codebase before.

# C Documentation

## Documentation Checklist

| Item | Required | Library | Application |
| --- | --- | --- | --- |
| Function doc comments | Yes | Yes | Yes |
| File-level header comment | Yes | Yes | Recommended |
| README.md | Yes | Yes | Yes |
| LICENSE | Yes | Yes | Yes |
| API reference (Doxygen) | Recommended | Yes | Maybe |
| USAGE.md / examples | Recommended | Yes | Maybe |
| CONTRIBUTING.md | Recommended | Yes | Yes |

## Function Documentation

Every function MUST have a doc comment. Follow the kernel-doc / Doxygen format:

```c
/**
 * hash_table_insert - Insert a key-value pair into the hash table.
 * @ht: hash table to insert into (must not be NULL)
 * @key: string key (copied internally, caller retains ownership)
 * @value: value pointer (not copied, caller retains ownership)
 *
 * Inserts a new entry. If the key already exists, the old value is
 * replaced and freed. Returns 0 on success, -ENOMEM if allocation
 * fails.
 *
 * Thread-safe: no (external locking required).
 *
 * Return: 0 on success, negative errno on failure.
 */
int hash_table_insert(hash_table_t *ht, const char *key, void *value)
{
    /* ... */
}
```

### Doc Comment Format

```c
/**
 * function_name - Brief one-line description.
 * @param1: description of first parameter
 * @param2: description of second parameter
 *
 * Detailed description (optional). Explain behavior, edge cases,
 * thread safety, and performance characteristics.
 *
 * Return: description of return value
 */
```

### When to Document

| What | Document? | How |
| --- | --- | --- |
| Public API functions | Always | Full doc comment with all params |
| Static helper functions | Recommended | Brief comment explaining purpose |
| Complex internal logic | Always | Explain *why*, not *what* |
| Trivial getters/setters | Brief | One-liner is sufficient |
| Macro definitions | Recommended | Explain usage and edge cases |
| Type definitions | Recommended | Explain semantics and invariants |

## Inline Comments

Explain **why**, not **what**:

```c
// Good — explains non-obvious decision
// Use 3x multiplier because the hash table needs room for load factor < 0.75
size_t new_size = ht->count * 3;

// Bad — restates the obvious
// multiply count by 3
size_t new_size = ht->count * 3;
```

## File-Level Comments

```c
/*
 * hash_table.c - Open-addressing hash table with linear probing.
 *
 * This module implements a fixed-size hash table using open addressing
 * with linear probing. Keys are strings, values are opaque pointers.
 *
 * The hash table is NOT thread-safe. External locking is required
 * for concurrent access.
 *
 * Copyright (c) 2026 Author Name
 * SPDX-License-Identifier: MIT
 */
```

## Header File Documentation

```c
/*
 * hash_table.h - Hash table API.
 *
 * Example usage:
 *
 *     hash_table_t *ht = hash_table_create(256);
 *     hash_table_insert(ht, "key", &value);
 *     void *found = hash_table_lookup(ht, "key");
 *     hash_table_destroy(ht);
 *
 * This header provides a simple string-keyed hash table.
 * See hash_table.c for implementation details.
 */

#ifndef MYLIB_HASH_TABLE_H
#define MYLIB_HASH_TABLE_H

#include <stddef.h>
#include <stdbool.h>

typedef struct hash_table hash_table_t;

hash_table_t *hash_table_create(size_t initial_size);
void hash_table_destroy(hash_table_t *ht);
int hash_table_insert(hash_table_t *ht, const char *key, void *value);
void *hash_table_lookup(hash_table_t *ht, const char *key);
bool hash_table_remove(hash_table_t *ht, const char *key);

#endif /* MYLIB_HASH_TABLE_H */
```

## Doxygen Setup

### Minimal Doxyfile

```
PROJECT_NAME = "My Library"
INPUT = src/ include/
FILE_PATTERNS = *.c *.h
RECURSIVE = NO
EXTRACT_ALL = NO
GENERATE_HTML = YES
GENERATE_LATEX = NO
```

### Generate Documentation

```bash
doxygen Doxyfile
# Output in html/ directory
```

### Doxygen-Style Comments

```c
/**
 * @brief Short description.
 *
 * Longer description with @param, @return, @note, @warning.
 *
 * @param[in]  input  Input data (must be valid UTF-8)
 * @param[out] output Output buffer (caller-allocated)
 * @param[in]  len    Size of output buffer
 *
 * @return Number of bytes written, or -errno on error.
 *
 * @warning This function is not thread-safe.
 *
 * @code
 *   char buf[256];
 *   int n = process(input, buf, sizeof(buf));
 * @endcode
 */
```

## README Structure

1. **Title** — project name as `# heading`
2. **Badges** — CI, license, coverage
3. **Summary** — 1-2 sentences
4. **Quick Start** — minimal working example
5. **Installation** — build instructions
6. **API Reference** — link to Doxygen or inline
7. **Examples** — code snippets
8. **Contributing** — link to CONTRIBUTING.md
9. **License** — license name

## Comment Conventions

- Use `/* */` for multi-line comments (C89 compatible)
- Use `//` for single-line comments (C99+)
- Never nest `/* */` comments
- Comment out code with a `// TODO:` or remove it — don't leave dead code commented out

```c
/* Good — multi-line block comment explaining complex logic */

// Good — single-line comment
int x = 5;

/* Bad — nested comment */
/* outer /* inner */ outer */
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Restating the code in comments | Explain *why*, not *what* |
| Missing doc comments on public API | Every public function needs documentation |
| Stale comments | Update comments when code changes |
| `// TODO:` without tracking | Link to issue tracker |
| Commenting out code | Remove it; use version control |
| Undocumented error return values | Document every possible return value |

## Cross-References

- → See `lobor/cc-skills-clang@clang-naming` skill for naming conventions in doc comments
- → See `lobor/cc-skills-clang@clang-code-style` skill for comment formatting
