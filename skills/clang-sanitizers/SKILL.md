---
name: clang-sanitizers
description: "C/Clang sanitizer skill for runtime bug detection. Use when enabling and interpreting AddressSanitizer (ASan), UndefinedBehaviorSanitizer (UBSan), ThreadSanitizer (TSan), MemorySanitizer (MSan), or LeakSanitizer (LSan) with Clang. Activates on queries about sanitizer flags, sanitizer reports, ASAN_OPTIONS, memory errors, data races, undefined behaviour, uninitialised reads, or choosing which sanitizer to use."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🔬"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

**Persona:** You are a C runtime safety engineer. You know which sanitizer detects which bug class and how to interpret their reports to pinpoint the root cause.

# C Sanitizers

## Decision Tree: Which Sanitizer?

```bash
Bug class?
├── Buffer overflow, use-after-free, double-free → AddressSanitizer (ASan)
├── Stack overflow, global OOB → ASan (all three covered)
├── Uninitialised reads → MemorySanitizer (MSan, Clang only, requires all-clang build)
├── Undefined behavior (int overflow, null deref, bad cast) → UBSan
├── Data races (multi-thread) → ThreadSanitizer (TSan)
├── Memory leaks only → LeakSanitizer (LSan, standalone or via ASan)
└── Multiple classes → ASan + UBSan (common combo); cannot combine with TSan or MSan
```

## AddressSanitizer (ASan)

```bash
clang -fsanitize=address -fno-omit-frame-pointer -g -O1 -o prog main.c
```

Runtime options via `ASAN_OPTIONS`:
```bash
ASAN_OPTIONS=detect_leaks=1:abort_on_error=1:log_path=/tmp/asan.log ./prog
```

| Option | Effect |
| --- | --- |
| `detect_leaks=0/1` | Enable LeakSanitizer (default 1 on Linux) |
| `abort_on_error=1` | Call `abort()` instead of `_exit()` (for core dumps) |
| `log_path=path` | Write report to file |
| `symbolize=1` | Symbolize addresses (needs `llvm-symbolizer` in PATH) |
| `fast_unwind_on_malloc=0` | More accurate stacks (slower) |

**Reading ASan output:**
```text
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000050
READ of size 4 at 0x602000000050 thread T0
    #0 0x401234 in foo /home/user/src/main.c:15
    #1 0x401567 in main /home/user/src/main.c:42

0x602000000050 is located 0 bytes after a 40-byte region
[0x602000000028, 0x602000000050) allocated at:
    #0 0x7f12345 in malloc ...
    #1 0x401234 in main /home/user/src/main.c:10
```

Reading: the top frame in `WRITE/READ` is the access site; the `allocated at` stack shows the allocation.

## UndefinedBehaviorSanitizer (UBSan)

```bash
clang -fsanitize=undefined \
    -fsanitize=signed-integer-overflow,unsigned-integer-overflow \
    -fno-sanitize-recover=all \
    -g -O1 -o prog main.c
```

Key checks:
- `signed-integer-overflow` — overflow on `int` is UB
- `unsigned-integer-overflow` — not in `undefined` by default
- `null` — null pointer dereference
- `bounds` — array index OOB
- `alignment` — misaligned pointer access
- `float-cast-overflow` — float-to-int conversion overflow

`-fno-sanitize-recover=all` — abort on first error (important for CI).

**Reading UBSan output:**
```text
src/main.c:15:12: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
```

## ThreadSanitizer (TSan)

```bash
clang -fsanitize=thread -g -O1 -o prog main.c
```

**TSan is incompatible with ASan and MSan.**

**Reading TSan output:**
```text
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x7f... by thread T2:
    #0 increment /home/user/src/counter.c:8
  Previous read of size 4 at 0x7f... by thread T1:
    #0 read_counter /home/user/src/counter.c:3
```

## MemorySanitizer (MSan)

Detects reads of uninitialised memory. **Clang only. Requires all-instrumented build.**

```bash
clang -fsanitize=memory -fsanitize-memory-track-origins=2 -g -O1 -o prog main.c
```

## ASan + UBSan Combined

```bash
clang -fsanitize=address,undefined \
    -fno-sanitize-recover=all \
    -fno-omit-frame-pointer -g -O1 -o prog main.c
```

Do not combine with TSan or MSan.

## Suppressions

```bash
# ASan suppression file
cat > asan.supp << 'EOF'
leak:CRYPTO_malloc
EOF
LSAN_OPTIONS=suppressions=asan.supp ./prog

# UBSan suppression
cat > ubsan.supp << 'EOF'
signed-integer-overflow:third_party/fast_math.c
EOF
UBSAN_OPTIONS=suppressions=ubsan.supp:print_stacktrace=1 ./prog
```

## CMake Integration

```cmake
option(SANITIZE "Enable sanitizers" OFF)
if(SANITIZE)
    set(san_flags -fsanitize=address,undefined -fno-sanitize-recover=all
                  -fno-omit-frame-pointer -g -O1)
    add_compile_options(${san_flags})
    add_link_options(${san_flags})
endif()
```

## CI Integration

```yaml
# GitHub Actions example
- name: Build with ASan+UBSan
  run: |
    cmake -S . -B build -DSANITIZE=ON
    cmake --build build -j$(nproc)

- name: Run tests under sanitizers
  run: |
    ASAN_OPTIONS=abort_on_error=1:detect_leaks=1 \
    UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1 \
    ctest --test-dir build -j$(nproc) --output-on-failure
```

## Quick Reference

| Sanitizer | Overhead | Detects | Combos |
| --- | --- | --- | --- |
| ASan | ~2x | OOB, UAF, double-free, leaks | + UBSan |
| UBSan | ~1x | UB (overflow, null, alignment) | + ASan |
| TSan | ~5-15x | Data races | Alone only |
| MSan | ~3x | Uninitialised reads | Alone only (Clang only) |

## Cross-References

- → See `lobor/cc-skills-clang@clang-security` skill for vulnerability patterns sanitizers detect
- → See `lobor/cc-skills-clang@clang-fuzzing` skill for combining sanitizers with fuzzing
- → See `lobor/cc-skills-clang@clang-safety` skill for code patterns that trigger sanitizer reports
- → See `lobor/cc-skills-clang@clang-testing` skill for CI integration
