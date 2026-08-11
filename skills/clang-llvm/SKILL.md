---
name: clang-llvm
description: "LLVM IR and tooling for C/Clang projects. Use when generating LLVM IR, running optimization passes with opt, inspecting assembly with llc, diagnosing missed optimizations, or understanding how Clang lowers C code. Covers IR inspection, optimization remarks, PGO workflow, and LLVM toolchain usage."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang and LLVM tools (opt, llc, llvm-dis).
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "⚙"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
        - opt
        - llc
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(opt:*) Bash(llc:*) Bash(git:*) Agent
---

**Persona:** You are an LLVM engineer. You think in SSA form and understand how C constructs lower to IR, enabling precise optimization diagnosis.

# LLVM IR and Tooling

## Generate LLVM IR

```bash
# Emit LLVM IR (human-readable)
clang -S -emit-llvm -O2 -o output.ll src.c

# Emit LLVM IR (bitcode, for tools)
clang -c -emit-llvm -O2 -o output.bc src.c

# From C with specific standard
clang -S -emit-llvm -std=c11 -O2 -o output.ll src.c
```

## Inspect IR

```bash
# Disassemble bitcode to human-readable IR
llvm-dis output.bc -o output.ll

# View IR in browser (requires LLVM built)
llvm-as output.ll -o output.bc
opt -view-bc output.bc
```

## Run Optimization Passes

```bash
# List available passes
opt --help

# Run specific pass
opt -passes=mem2reg -S input.ll -o output.ll

# Run standard optimization pipeline
opt -O2 -S input.ll -o output.ll

# Run specific passes
opt -passes='function(mem2reg),function(sccp),function(dce)' \
    -S input.ll -o output.ll

# View pass analysis results
opt -passes=loop-info -disable-output input.ll
```

## Lower to Assembly

```bash
# Generate x86-64 assembly
llc -march=x86-64 -O2 input.bc -o output.s

# Generate with specific CPU features
llc -march=x86-64 -mcpu=native -O2 input.bc -o output.s

# View what the compiler optimized away
llc -O2 -print-after-all input.bc 2>/dev/null | less
```

## Optimization Remarks

See what Clang/LLVM did or refused to do:

```bash
# Inliner decisions
clang -O2 -Rpass=inline src.c

# Missed vectorization
clang -O2 -Rpass-missed=loop-vectorize src.c

# Analysis of why loop wasn't vectorized
clang -O2 -Rpass-analysis=loop-vectorize src.c

# Save all remarks to YAML
clang -O2 -fsave-optimization-record src.c
# Produces src.opt.yaml
```

### Interpreting Remarks

| Remark | Meaning | Action |
| --- | --- | --- |
| `foo inlined into bar` | Inlining happened | Good for hot paths |
| `loop not vectorized: control flow not understood` | Restructure loop | Simplify branches |
| `not vectorized: cannot prove safe to reorder` | Add `__restrict__` | Help alias analysis |
| `not vectorized: calls with side effects` | Move side effects out | Enable vectorization |

## Profile-Guided Optimization (PGO)

```bash
# Step 1: Instrument
clang -O2 -fprofile-instr-generate prog.c -o prog_inst

# Step 2: Run with representative input
./prog_inst < workload.input
# Generates default.profraw

# Step 3: Merge profiles
llvm-profdata merge -output=prog.profdata default.profraw

# Step 4: Use profile
clang -O2 -fprofile-instr-use=prog.profdata prog.c -o prog
```

PGO typically yields 10-30% improvement with zero code changes.

## Link-Time Optimization (LTO)

```bash
# Full LTO
clang -O2 -flto -fuse-ld=lld src.c -o prog

# Thin LTO (faster link, nearly same quality)
clang -O2 -flto=thin -fuse-ld=lld src.c -o prog
```

For large projects, ThinLTO is preferred: link times 5-10x faster than full LTO.

## Useful llvm-* Tools

```bash
# Display object file sections and symbols
llvm-size output.o
llvm-nm output.o
llvm-objdump -d output.o

# Symbolize addresses from sanitizer output
llvm-symbolizer --obj=prog 0x401234

# Merge profiling data
llvm-profdata merge -output=merged.profdata *.profraw

# Check binary for specific features
llvm-readelf -h output  # ELF header
llvm-readelf -S output  # Section headers
```

## Common Patterns

### Why Isn't My Loop Vectorizing?

```bash
# Check why
clang -O2 -Rpass-analysis=loop-vectorize src.c

# Common fixes:
# 1. Add __restrict__ to pointer parameters
# 2. Remove conditional inside loop
# 3. Use #pragma clang loop vectorize(assume_safety)
# 4. Ensure trip count is known or computable
```

### Escape Analysis

```bash
# Check which variables escape to heap
clang -O2 -Xclang -fdump-record-layouts src.c

# Or use opt to see allocas
opt -passes=mem2reg -S input.ll | grep alloca
```

## Cross-References

- → See `lobor/cc-skills-clang@clang-compiler-optimizations` skill for deep optimization internals (register allocation, BOLT, LICM, vectorization failure triage)
- → See `lobor/cc-skills-clang@clang-performance` skill for optimization methodology and profiling
- → See `lobor/cc-skills-clang@clang-sanitizers` skill for sanitizer instrumentation
- → See `lobor/cc-skills-clang@clang-clang` skill for Clang-specific flags and diagnostics
