---
name: clang-bounds-safety
description: "Apple -fbounds-safety extension for C — compile-time bounds annotations, pointer kinds, runtime traps. Use when adopting or reviewing code with -fbounds-safety, or when evaluating bounds-safe alternatives to raw pointers."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler with -fbounds-safety support (Apple Clang 14+).
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🛡️"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

# Clang Bounds Safety

`-fbounds-safety` is a C language extension that prevents out-of-bounds memory access by enforcing bounds safety at the language level.

## Overview

- Automatic runtime bounds checks
- Compile-time rejection of unsafe pointer operations
- Bounds annotations on pointers (`__counted_by`, `__sized_by`, `__single`, etc.)
- Out-of-bounds accesses become deterministic traps, not exploitable vulnerabilities

## Pointer Kinds

| Kind | Meaning | Example |
| --- | --- | --- |
| `__single` | Points to exactly one element | `int *__single p` |
| `__counted_by(n)` | Points to `n` elements | `int *__counted_by(len) arr` |
| `__sized_by(n)` | Points to `n` bytes | `void *__sized_by(size) buf` |
| `__ended_by(end)` | Valid until `end` pointer | `char *__ended_by(end) start` |
| `__indexable` | Indexed access | `int *__indexable p` |
| `__bidi_indexable` | Forward + backward traversal | `int *__bidi_indexable p` |
| `__null_terminated` | Null-terminated string | `char *__null_terminated s` |
| `__unsafe_indexable` | No bounds info (opt-out) | `void *__unsafe_indexable p` |

## Basic Pattern

```c
// WITHOUT -fbounds-safety: silent buffer overflow
void process(int *buf, size_t len) {
    for (size_t i = 0; i <= len; i++)  // off-by-one: writes past end
        buf[i] = 0;
}

// WITH -fbounds-safety: compiler catches or traps
void process(int *__counted_by(len) buf, size_t len) {
    for (size_t i = 0; i <= len; i++)  // compile-time or runtime trap
        buf[i] = 0;
}
```

## Adoption Strategy

1. **Header-only mode** — add annotations to headers first, keep implementation untouched
2. **Full mode** — annotate all pointers, fix type mismatches
3. **Iterative** — start with `__single` for simple pointers, add `__counted_by` for arrays

## Review Gate

- Does every pointer have a bounds annotation?
- Are `__unsafe_indexable` uses justified?
- Does the build use `-fbounds-safety` (not just `-fbounds-safety=strict`)?
