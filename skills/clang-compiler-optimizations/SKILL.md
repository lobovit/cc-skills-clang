---
name: clang-compiler-optimizations
description: "Deep C/Clang compiler optimization internals — register allocation, instruction selection (ISel), loop-invariant code motion (LICM), vectorization failure triage, BOLT post-link optimization, and pass interaction. Use when -O3 fails to vectorize a hot loop, when diagnosing register pressure spills, when planning PGO/BOLT deployment, or when understanding why passes interact unexpectedly. For LLVM tooling (opt/llc/IR) → See `lobor/cc-skills-clang@clang-llvm`; for high-level optimization methodology → See `lobor/cc-skills-clang@clang-performance`."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang and LLVM tools.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🧠"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
        - opt
        - llc
        - llvm-bolt
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(opt:*) Bash(llc:*) Bash(git:*) Agent WebSearch
---

**Persona:** You are an LLVM compiler engineer. You understand the full pipeline from C source to machine code — front-end, mid-level IR opts, codegen, register allocation — and can diagnose why the compiler made a specific decision.

**Thinking mode:** Use `ultrathink` for optimization diagnosis. Shallow analysis misidentifies the root cause — deep reasoning traces the exact pass that changed behavior.

# Deep Compiler Optimizations

This skill covers optimization phases **beyond flags**: mid-level IR opts, register allocation, instruction selection/scheduling, vectorization boundaries, and post-link BOLT. For LLVM tooling basics see `lobor/cc-skills-clang@clang-llvm`; for methodology see `lobor/cc-skills-clang@clang-performance`.

## Compiler Pipeline

```
Source → Frontend → LLVM IR → Optimization Passes → CodeGen → Machine Code
                         ↓
                  [Mid-level] DCE, GVN, LICM, inlining
                  [Loop opts] unroll, vectorize
                  [Codegen prep] legalize types
                  [ISel] DAG → machine ops
                  [RegAlloc] greedy / linear scan
                  [Peephole] scheduling, cleanup
```

Each phase depends on earlier ones. A missed optimization often means an earlier pass blocked it.

## Vectorization Failure Triage

When `clang -O3` fails to vectorize, diagnose why:

```bash
# Show what vectorizer did and didn't do
clang -O3 -Rpass=loop-vectorize -Rpass-missed=loop-vectorize -Rpass-analysis=loop-vectorize src.c
```

| Miss reason | Compiler message | Fix |
| --- | --- | --- |
| Unknown trip count | `cannot identify loop bounds` | Peel loop; assert or prove count |
| Data dependence | `call to unknown function` | Reorder; separate accumulators |
| Function call in loop | `found potential loop interdependence` | Inline or outline the call |
| Alignment unknown | `alignment too small` | Add `__builtin_assume_aligned` |
| Control flow in loop | `loop control flow is not understood` | Simplify branches; flatten conditionals |
| Reduction pattern | `reduction operation cannot be vectorized` | Rewrite as parallel accumulation |
| External dependency | `loop not vectorized: may not be profitable` | Reduce work; enable with `#pragma clang loop vectorize(assume_safety)` |

### Force Vectorization (with care)

```c
// Tell the compiler it's safe to reorder
#pragma clang loop vectorize(assume_safety)
for (int i = 1; i < n; i++) {
    a[i] = a[i-1] + b[i];  // dependence is sequential, but safe
}
```

## Loop-Invariant Code Motion (LICM)

LICM hoists loop-invariant computations out of loops:

```c
// Before LICM
for (int i = 0; i < n; i++) {
    result[i] = x * scale + offset;  // x * scale is loop-invariant
}

// After LICM (compiler does this)
int temp = x * scale;
for (int i = 0; i < n; i++) {
    result[i] = temp + offset;
}
```

**When LICM fails:**
- The hoisted value has side effects (memory write, function call)
- The hoisted value traps on some iteration (e.g., division by zero path)
- The value is used on a non-dominating path

Verify with:
```bash
opt -passes=licm -S input.ll -o output.ll
# Or view what changed
opt -passes=licm -S input.ll | diff input.ll - 
```

## Register Allocation

When live ranges exceed physical registers, the allocator **spills** to stack slots — costly loads/stores appear in the hot path.

### Diagnose Spills

```bash
# View register allocation decisions
llc -regalloc=greedy -print-after=regalloc input.bc 2>/dev/null | less

# Count spills/reloads
llc -regalloc=greedy input.bc 2>&1 | grep -c "spill"
```

### Reduce Register Pressure

```c
// Bad — long live range for 'sum'
int sum = 0;
for (int i = 0; i < n; i++) {
    sum += process(data[i]);  // 'sum' live across entire loop
    other_work();             // 'sum' still live here
}

// Better — shorten live range
int partial = 0;
for (int i = 0; i < n; i++) {
    partial += process(data[i]);
}
int sum = partial;  // live only at end
```

**Techniques:**
- Split variables to reduce live ranges
- Rematerialize cheap computations instead of spilling
- Use `__attribute__((noinline))` to prevent function inlining that increases pressure

## Instruction Selection (ISel)

ISel converts LLVM IR into target-specific machine instructions. Issues arise when:

- The target has no matching instruction for an IR pattern
- The selected instruction has unexpected encoding constraints

```bash
# View ISel decisions
opt -passes=isel -disable-output input.ll 2>&1 | less

# See what the target selected
llc -debug-only=isel input.bc 2>&1 | head -100
```

## BOLT (Post-Link Optimization)

BOLT optimizes code layout **after** the linker — reorders basic blocks for better I-cache and branch prediction:

```bash
# Step 1: Build with relocations preserved
clang -O2 -Wl,--emit-relocs -o app src.c

# Step 2: Collect profile data (run with representative input)
perf record -e cycles:u -j any,u -o perf.data -- ./app

# Step 3: Convert perf data
llvm-profdata merge -output=perf.fdata -sample perf.data

# Step 4: Optimize with BOLT
llvm-bolt app -data=perf.fdata -reorder-blocks=ext-tsp -reorder-functions=hfsort+ -o app.bolt
```

BOLT typically yields 5-15% improvement on top of PGO.

## Pass Interaction

Passes interact in non-obvious ways. Common issues:

| Symptom | Cause | Fix |
| --- | --- | --- |
| PGO no gain | Unrepresentative training data | Match production workload distribution |
| BOLT crash | Stripped binary | Keep symbols + relocations (`-Wl,--emit-relocs`) |
| Spills in asm | Register pressure from inlining | `__attribute__((noinline))` on cold paths |
| `-O3` slower than `-O2` | Code bloat hurting I-cache | Try `-O2` or use PGO to guide inlining |
| Different GCC vs Clang | Pass ordering differs | Compare IR (`-S -emit-llvm`) and assembly |

## Decision Tree

| Problem | First tool | Escalation |
| --- | --- | --- |
| Loop not vectorizing | `-Rpass-missed=loop-vectorize` | Inspect IR before/after vectorizer |
| Too many spills | `llc -print-after=regalloc` | Reduce live ranges, `noinline` |
| Slow despite `-O3` | `perf record` + flame graph | Check BOLT, PGO |
| Missed LICM | `opt -passes=licm` | Check for side effects blocking hoist |
| Layout hurts perf | BOLT with perf data | `-reorder-blocks=ext-tsp` |

## Cross-References

- → See `lobor/cc-skills-clang@clang-llvm` for IR generation, opt/llc basics, PGO workflow
- → See `lobor/cc-skills-clang@clang-performance` for high-level optimization methodology and profiling
- → See `lobor/cc-skills-clang@clang-clang` for Clang flags and diagnostics
