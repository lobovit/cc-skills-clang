---
name: clang-assembly
description: "x86-64 assembly for C developers — reading compiler output, inline asm, ABI calling conventions, SSE/AVX intrinsics. Use when inspecting assembly from clang -S, writing inline asm, or debugging register state."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "⚙️"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

# Clang Assembly

Reading, writing, and debugging x86-64 assembly from C code.

## Generate Assembly

```bash
# AT&T syntax (default)
clang -S -O2 -fverbose-asm foo.c -o foo.s

# Intel syntax
clang -S -masm=intel -O2 foo.c -o foo.s

# With source interleaved
clang -S -g -masm=intel foo.c -o foo.s

# From objdump
objdump -d -M intel -S a.out  # needs -g
```

## System V AMD64 ABI

**Integer args:** `%rdi, %rsi, %rdx, %rcx, %r8, %r9`
**Float args:** `%xmm0`–`%xmm7`
**Return:** `%rax` (int), `%xmm0` (float)
**Caller-saved:** `%rax, %rcx, %rdx, %rsi, %rdi, %r8–%r11`
**Callee-saved:** `%rbx, %rbp, %r12–%r15`

## Common Patterns

| Pattern | Meaning |
| --- | --- |
| `mov %rdi, %rax` | Copy rdi to rax |
| `mov (%rdi), %rax` | Load 8 bytes from address |
| `lea 8(%rdi), %rax` | Compute address (no load) |
| `xor %eax, %eax` | Zero rax (smaller than `mov $0, %rax`) |
| `test %rax, %rax` | Check if rax is zero |
| `push %rbx` | Save callee-saved register |

## Inline Assembly

```c
// Atomic increment
static inline int atomic_inc(volatile int *p) {
    int ret;
    __asm__ volatile (
        "lock; xaddl %0, %1"
        : "=r"(ret), "+m"(*p)
        : "0"(1)
        : "memory"
    );
    return ret + 1;
}
```

**Constraints:** `"r"` any reg, `"m"` memory, `"a"/"b"/"c"/"d"` specific regs, `"="` write-only, `"+"` read-write, `"memory"` clobber.

## SSE/AVX Intrinsics (preferred over inline asm)

```c
#include <immintrin.h>

__m256 a = _mm256_loadu_ps(arr);
__m256 b = _mm256_loadu_ps(arr + 8);
__m256 c = _mm256_add_ps(a, b);
_mm256_storeu_ps(result, c);
```

Check at runtime: `__builtin_cpu_supports("avx2")`
