---
name: clang-clang
description: "Clang/LLVM compiler skill for C projects. Use when working with clang for diagnostics, sanitizer instrumentation, optimization remarks, static analysis with clang-tidy, LTO via lld, or when migrating from GCC to Clang. Activates on queries about clang flags, clang-tidy, better error messages, Apple/FreeBSD toolchains, or LLVM-specific optimizations. Covers flag selection, diagnostic tuning, and integration with LLVM tooling."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🔧"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(clang-tidy:*) Bash(clang-format:*) Bash(git:*) Agent
---

**Persona:** You are a Clang compiler engineer. You leverage Clang's superior diagnostics and LLVM tooling to find and fix issues that other compilers miss.

# Clang Compiler

## Build Flags

Clang accepts most GCC flags. Key differences:

| Feature | GCC | Clang |
| --- | --- | --- |
| Min size | `-Os` | `-Os` or `-Oz` (more aggressive) |
| Thin LTO | `-flto` | `-flto=thin` (faster) |
| Static analyzer | `-fanalyzer` | `clang --analyze` or `clang-tidy` |
| Fix-it hints | — | `--show-fixits` |

## Diagnostic Flags

```bash
# Show fix-it hints inline
clang -Wall -Wextra --show-fixits src.c

# Limit error count
clang -ferror-limit=5 src.c

# Verbose errors
clang -fdiagnostics-show-template-tree src.c
```

## Recommended Warning Flags

```bash
clang -Wall -Wextra -Werror \
      -Wshadow -Wconversion -Wsign-conversion \
      -Wformat=2 -Wnull-dereference \
      -Wdouble-promotion -Wimplicit-fallthrough \
      -Wpedantic -std=c11 \
      -o output src.c
```

## Static Analysis

```bash
# Built-in Clang Static Analyzer (CSA)
clang --analyze -Xanalyzer -analyzer-output=text src.c

# clang-tidy (richer checks)
clang-tidy src.c -- -std=c11 -I/usr/include

# Enable specific check families
clang-tidy -checks='bugprone-*,clang-analyzer-*,cert-*' src.c --

# Apply fixits automatically
clang-tidy -fix src.c --
```

### Key clang-tidy Checks for C

| Check | Purpose |
| --- | --- |
| `bugprone-*` | Real bugs (use-after-move, dangling, etc.) |
| `clang-analyzer-*` | CSA checks (memory, null deref) |
| `cert-*` | CERT C secure coding rules |
| `misc-*` | Miscellaneous best practices |
| `readability-*` | Naming, complexity |

## LTO with lld

```bash
# Full LTO
clang -O2 -flto -fuse-ld=lld src.c -o prog

# Thin LTO (faster link, nearly same quality)
clang -O2 -flto=thin -fuse-ld=lld src.c -o prog
```

For large projects, ThinLTO is preferred: link times 5-10x faster than full LTO.

## PGO (LLVM Instrumentation)

```bash
# Step 1: Instrument
clang -O2 -fprofile-instr-generate prog.c -o prog_inst

# Step 2: Run with representative input
./prog_inst < workload.input

# Step 3: Merge profiles
llvm-profdata merge -output=prog.profdata default.profraw

# Step 4: Use profile
clang -O2 -fprofile-instr-use=prog.profdata prog.c -o prog
```

## macOS Specifics

- `ld64` is the default linker; `lld` requires `-fuse-ld=lld` and Homebrew LLVM
- Use `-mmacosx-version-min=X.Y` for deployment target
- Sanitizers use `DYLD_INSERT_LIBRARIES`

## Cross-References

- → See `lobor/cc-skills-clang@clang-static-analysis` skill for detailed clang-tidy configuration, PR analysis, and CI integration
- → See `lobor/cc-skills-clang@clang-llvm` skill for IR-level work (`opt`, `llc`, `llvm-dis`)
- → See `lobor/cc-skills-clang@clang-sanitizers` skill for `-fsanitize=*` workflows
- → See `lobor/cc-skills-clang@clang-format` skill for formatting configuration
- → See `lobor/cc-skills-clang@clang-performance` skill for optimization methodology
