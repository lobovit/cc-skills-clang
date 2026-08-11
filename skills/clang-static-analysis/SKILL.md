---
name: clang-static-analysis
description: "Detailed C/Clang static analysis with clang-tidy — configuration, custom checks, PR diff analysis, auto-fix workflows, CI integration, and check selection. Use when setting up clang-tidy for a project, running analysis on git diffs, auto-fixing issues, selecting specific check families, or integrating static analysis into CI pipelines. For basic clang-tidy flags → See `lobor/cc-skills-clang@clang-clang`; for formatting → See `lobor/cc-skills-clang@clang-format`."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang-tidy and clang.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🔍"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
        - clang-tidy
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(clang-tidy:*) Bash(git:*) Agent
---

**Persona:** You are a static analysis engineer. You configure clang-tidy to catch real bugs with zero false positives — not to generate noise that developers ignore.

# C Static Analysis with clang-tidy

For basic Clang flags and compiler diagnostics see `lobor/cc-skills-clang@clang-clang`. This skill covers detailed clang-tidy configuration, workflows, and CI integration.

## Quick Start

```bash
# Generate compile_commands.json (required)
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# Run clang-tidy on a file
clang-tidy src/main.c -- -std=c11 -Iinclude

# Run on entire project
run-clang-tidy -p build -j $(nproc)

# Auto-fix safe issues
clang-tidy -fix src/main.c -- -std=c11 -Iinclude
```

## .clang-tidy Configuration

Place in project root:

```yaml
Checks: >
  -*,
  bugprone-*,
  cert-*,
  clang-analyzer-*,
  misc-*,
  readability-identifier-naming,
  -bugprone-easily-swappable-parameters,
  -readability-magic-numbers

WarningsAsErrors: >
  bugprone-use-after-move,
  cert-err33-c,
  clang-analyzer-core.NullDereference

HeaderFilterRegex: 'include/.*\.h$'

CheckOptions:
  - key: readability-identifier-naming.FunctionCase
    value: CamelBack
  - key: readability-identifier-naming.VariableCase
    value: CamelBack
  - key: readability-identifier-naming.ConstantCase
    value: UPPER_CASE
```

## Check Families for C

| Family | Purpose | Key checks |
| --- | --- | --- |
| `bugprone-*` | Real bugs | `use-after-move`, `dangling-handle`, `sizeof-expression` |
| `cert-*` | CERT C rules | `err33-c` (check return value), `str50-cpp` (string bounds) |
| `clang-analyzer-*` | CSA deep checks | `core.NullDereference`, `deadcode.DeadStores` |
| `misc-*` | Miscellaneous | `misc-redundant-expression`, `misc-unused-using-decls` |
| `performance-*` | Perf anti-patterns | `unnecessary-value-param`, `move-const-arg` |

### Recommended C Check Set

```yaml
Checks: >
  -*,
  bugprone-*,
  -bugprone-easily-swappable-parameters,
  cert-*,
  -cert-dcl03-c,
  clang-analyzer-*,
  misc-*,
  -misc-no-recursion
```

## PR Diff Analysis

Run clang-tidy only on changed files:

```bash
# Analyze staged changes
git diff --cached --name-only --diff-filter=ACM | \
  grep -E '\.(c|h)$' | \
  xargs -I{} clang-tidy {} -- -std=c11 -Iinclude

# Analyze changes vs main branch
git diff main --name-only --diff-filter=ACM | \
  grep -E '\.(c|h)$' | \
  xargs -I{} clang-tidy {} -- -std=c11 -Iinclude
```

### Script for PR Analysis

```bash
#!/bin/bash
# scripts/clang-tidy-diff.sh
set -euo pipefail

BASE_BRANCH="${1:-main}"
BUILD_DIR="build"
FLAGS="-std=c11 -Iinclude -Wall"

# Generate compile commands if missing
if [ ! -f "$BUILD_DIR/compile_commands.json" ]; then
    cmake -B "$BUILD_DIR" -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
fi

# Get changed files
FILES=$(git diff "$BASE_BRANCH" --name-only --diff-filter=ACM | grep -E '\.(c|h)$' || true)

if [ -z "$FILES" ]; then
    echo "No C/H files changed."
    exit 0
fi

echo "Analyzing: $FILES"
echo "$FILES" | xargs clang-tidy -p "$BUILD_DIR" $FLAGS
```

## Auto-Fix Workflow

```bash
# Step 1: See all issues (no fix)
clang-tidy -p build src/*.c

# Step 2: Auto-fix safe issues
clang-tidy -fix -p build src/*.c

# Step 3: Verify remaining (manual review needed)
clang-tidy -p build src/*.c
```

**What clang-tidy auto-fixes:**
- Missing `#include` directives
- Redundant string conversions
- Some naming convention violations
- `sizeof(expression)` instead of `sizeof(type)`

**What requires manual fix:**
- Null pointer dereferences
- Use-after-move
- Resource leaks
- Logic errors

## CI Integration

### GitHub Actions

```yaml
- name: Generate compile_commands.json
  run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

- name: Run clang-tidy
  run: |
    FILES=$(git diff --name-only HEAD~1 | grep -E '\.(c|h)$' || true)
    if [ -n "$FILES" ]; then
      echo "$FILES" | xargs clang-tidy -p build --warnings-as-errors='*'
    fi
```

### Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit
FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(c|h)$')
if [ -n "$FILES" ]; then
    echo "$FILES" | xargs clang-tidy -p build --warnings-as-errors='*'
    if [ $? -ne 0 ]; then
        echo "clang-tidy failed. Fix errors before committing."
        exit 1
    fi
fi
```

## Performance-Specific Checks

```bash
# Find unnecessary copies
clang-tidy --checks='performance-unnecessary-value-param' src/*.c

# Find move anti-patterns  
clang-tidy --checks='performance-move-const-arg' src/*.c
```

## Security Checks

```bash
# CERT secure coding rules
clang-tidy --checks='cert-*' src/*.c

# Clang static analyzer deep checks
clang-tidy --checks='clang-analyzer-*' src/*.c
```

## Common Patterns

### Check Only Specific Files

```bash
# Only check files matching a pattern
clang-tidy src/network/*.c -- -std=c11 -Iinclude
```

### Suppress Specific Warnings

```cpp
// NOLINTNEXTLINE(bugprone-easily-swappable-parameters)
void process(int x, int y) { /* ... */ }

// Or for a specific line
int result = compute(a, b);  // NOLINT(clang-analyzer-core.DivideZero)
```

### Fix in Batch

```bash
# Fix all issues in a directory
find src/ -name '*.c' | xargs clang-tidy -fix -p build
```

## Cross-References

- → See `lobor/cc-skills-clang@clang-clang` for basic clang-tidy flags and compiler diagnostics
- → See `lobor/cc-skills-clang@clang-format` for automated code formatting
- → See `lobor/cc-skills-clang@clang-security` for security-focused analysis patterns
- → See `lobor/cc-skills-clang@clang-safety` for defensive coding patterns
