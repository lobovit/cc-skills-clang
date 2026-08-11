---
name: clang-performance
description: "C/Clang performance optimization patterns and methodology — CPU profiling with perf/oprofile, memory optimization, cache-friendly data structures, compiler optimization levels, intrinsics, and hot-path optimization. Use when profiling has identified a bottleneck and you need the right optimization pattern to fix it. Also use when performing performance code review. Not for sanitizers (→ See `lobor/cc-skills-clang@clang-sanitizers` skill) or fuzzing (→ See `lobor/cc-skills-clang@clang-fuzzing` skill)."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🏎"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent WebSearch AskUserQuestion
---

**Persona:** You are a C performance engineer. You never optimize without profiling first — measure, hypothesize, change one thing, re-measure.

# C Performance Optimization

## Core Philosophy

1. **Profile before optimizing** — intuition about bottlenecks is wrong ~80% of the time
2. **Compiler optimizations do most of the work** — `-O2` or `-O3` handles low-hanging fruit
3. **Document optimizations** — add comments explaining why a pattern is faster

## Compiler Optimization Levels

| Level | Flags | When to Use |
| --- | --- | --- |
| Debug | `-O0 -g` | Development, debugging |
| Balanced | `-O2` | **Default for release builds** — best speed/size tradeoff |
| Maximum | `-O3` | Compute-heavy hot paths (may increase code size) |
| Size | `-Os -Oz` | Embedded, binary size matters |
| PGO | `-O2 -fprofile-instr-use` | After profiling with instrumentation |
| LTO | `-O2 -flto=thin` | Cross-module inlining and optimization |

**Start with `-O2`** for production. Only escalate to `-O3` after profiling confirms it helps.

## Profiling Tools

### perf (Linux)

```bash
# Record CPU profile
perf record -g -p <pid> -- sleep 30

# Or record a command
perf record -g -- ./benchmark

# Generate report
perf report

# Flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### gprof

```bash
# Compile with profiling
clang -pg -O2 -o app src.c

# Run with representative input
./app < workload.txt

# View profile
gprof app gmon.out
```

### Simple Timing

```c
#include <time.h>

struct timespec start, end;
clock_gettime(CLOCK_MONOTONIC, &start);

// ... hot code ...

clock_gettime(CLOCK_MONOTONIC, &end);
double elapsed = (end.tv_sec - start.tv_sec) +
                 (end.tv_nsec - start.tv_nsec) / 1e9;
printf("Elapsed: %.6f seconds\n", elapsed);
```

## CPU Optimization

### Inlining

The compiler inlines small functions automatically. Force inlining for hot paths:

```c
// Hint to compiler — use sparingly
static inline int fast_path(int x) __attribute__((always_inline));

// PGO is better — lets the compiler decide based on runtime data
```

### Branch Prediction

Help the compiler with likely/unlikely hints:

```c
#define LIKELY(x)   __builtin_expect(!!(x), 1)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)

if (LIKELY(valid_input)) {
    // hot path
} else if (UNLIKELY(error)) {
    // cold path
}
```

### Loop Optimization

```c
// Good — loop with predictable branch
for (int i = 0; i < n; i++) {
    data[i] = process(data[i]);
}

// Bad — branch inside loop prevents vectorization
for (int i = 0; i < n; i++) {
    if (data[i] > threshold) {
        data[i] = process(data[i]);
    }
}
```

### SIMD Intrinsics

For hot loops on known data types:

```c
#include <immintrin.h>

// Vectorized sum of float array
float sum_array(const float *arr, size_t n)
{
    __m256 sum = _mm256_setzero_ps();
    size_t i;

    for (i = 0; i + 8 <= n; i += 8) {
        __m256 v = _mm256_loadu_ps(arr + i);
        sum = _mm256_add_ps(sum, v);
    }

    // Horizontal sum
    float tmp[8];
    _mm256_storeu_ps(tmp, sum);
    float result = 0;
    for (int j = 0; j < 8; j++) result += tmp[j];

    // Handle remainder
    for (; i < n; i++) result += arr[i];

    return result;
}
```

## Memory Optimization

### Cache-Friendly Data Structures

```c
// Bad — AoS (Array of Structs): poor cache locality for hot field
struct particle {
    float x, y, z;     // position
    float vx, vy, vz;  // velocity
    float mass;         // hot field
    char name[32];      // cold field
};
struct particle particles[N];  // scanning 'mass' touches all fields

// Good — SoA (Struct of Arrays): hot fields contiguous
struct particles {
    float mass[N];      // contiguous in cache
    float x[N], y[N], z[N];
    float vx[N], vy[N], vz[N];
    char name[N][32];
};
```

### Memory Prefetching

```c
// Prefetch next cache line while processing current
for (int i = 0; i < n; i++) {
    __builtin_prefetch(&data[i + 8], 0, 3);  // read, high locality
    process(data[i]);
}
```

### Allocation Patterns

```c
// Bad — per-element allocation (lots of malloc overhead)
for (int i = 0; i < n; i++) {
    items[i] = malloc(sizeof(struct item));
}

// Good — single allocation
struct item **items = malloc(n * sizeof(*items));
struct item *pool = malloc(n * sizeof(*pool));
for (int i = 0; i < n; i++) {
    items[i] = &pool[i];
}
```

## Decision Tree

| Symptom | Tool | Action |
| --- |---|---|
| CPU-bound hot loop | `perf` / gprof | Optimize the hot function |
| Too many allocations | malloc tracking / ASan | Batch allocate, use pools |
| Cache misses | `perf stat -e cache-misses` | Restructure data layout (AoS → SoA) |
| Branch mispredictions | `perf stat -e branch-miss` | Restructure branches, use likely/unlikely |
| I/O bound | `strace` | Reduce syscalls, batch operations |
| String-heavy processing | Benchmarks | Use `memcpy`/`memchr` instead of `strcmp`/`strchr` |

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Optimizing without profiling | Profile first — `perf record` takes 30 seconds |
| `-O3` without measuring | `-O3` may increase code size and hurt cache — benchmark |
| Micro-optimizing cold code | Focus on hot paths identified by profiler |
| Ignoring compiler warnings | Warnings often indicate missed optimization opportunities |
| Manual optimization before compiler | Let `-O2`/`-O3` do its job first |
| Not using PGO | PGO can yield 10-30% improvement with zero code changes |

## Cross-References

- → See `lobor/cc-skills-clang@clang-llvm` skill for LLVM optimization remarks and IR inspection
- → See `lobor/cc-skills-clang@clang-testing` skill for benchmarking infrastructure
- → See `lobor/cc-skills-clang@clang-code-style` skill for code clarity vs performance tradeoffs
