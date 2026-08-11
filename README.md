# cc-skills-clang

AI Agent Skills for production-ready C/Clang projects.

## Installation

```bash
# Install all skills
npx skills add https://github.com/lobor/cc-skills-clang --all

# Install a specific skill
npx skills add https://github.com/lobor/cc-skills-clang --skill clang-security
```

### Claude Code

```
/plugin marketplace add lobor/cc-skills-clang
```

### Cursor

```bash
git clone https://github.com/lobor/cc-skills-clang.git ~/.cursor/skills/cc-skills-clang
```

### OpenCode

```bash
git clone https://github.com/lobor/cc-skills-clang.git ~/.agents/skills/cc-skills-clang
```

## Skills (23)

```
                          ┌──────────────────────────────────────────────┐
                          │              C/Clang Skills                  │
                          └────────────────────┬─────────────────────────┘
                                               │
    ┌─────────────────┬────────────────────────┼────────────────────────┐
    ▼                 ▼                        ▼                        ▼
┌──────────────┐ ┌──────────────┐ ┌───────────────────┐ ┌──────────────────────┐
│ Code Quality │ │ Architecture │ │   QA & Perf       │ │    Toolchain         │
├──────────────┤ ├──────────────┤ ├───────────────────┤ ├──────────────────────┤
│ code-style   │ │ design-patt  │ │ testing           │ │ clang                │
│ naming       │ │ mem-safety   │ │ performance       │ │ llvm                 │
│ error-handl  │ │ concurrency  │ │ sanitizers        │ │ clang-format         │
│ safety       │ │              │ │ fuzzing           │ │ static-analysis      │
│ documentation│ │              │ │ compiler-opts     │ │ kernel-style         │
│ security     │ │              │ │                   │ │ doctrine             │
│              │ │              │ │                   │ │ bounds-safety        │
│              │ │              │ │                   │ │ assembly             │
│              │ │              │ │                   │ │ kernel-modules       │
└──────────────┘ └──────────────┘ └───────────────────┘ └──────────────────────┘
```

| Skill | Category | Description |
| --- | --- | --- |
| `clang-code-style` | Quality | Line length, braces, control flow, const correctness |
| `clang-naming` | Quality | Functions, variables, macros, enums, typedefs, file naming |
| `clang-error-handling` | Quality | Return codes, errno, goto cleanup, error context |
| `clang-safety` | Quality | Null safety, buffer overflow, integer overflow, uninitialized memory |
| `clang-security` | Quality | Buffer overflows, format strings, command injection, CERT C |
| `clang-documentation` | Quality | Doxygen comments, README, API documentation, inline comments |
| `clang-design-patterns` | Arch | Opaque structs, tagged unions, function tables, state machines |
| `clang-memory-safety` | Arch | Ownership, RAII-like patterns, leak prevention, Valgrind/ASan |
| `clang-concurrency` | Arch | pthreads, C11 atomics, mutexes, condition variables, thread pools |
| `clang-testing` | QA | Check/CMocka/munit, coverage, mocking, CI integration |
| `clang-performance` | QA | CPU optimization, profiling, cache-friendly data structures, SIMD |
| `clang-sanitizers` | QA | ASan/UBSan/TSan/MSan decision trees and report interpretation |
| `clang-fuzzing` | QA | libFuzzer/AFL++ fuzzing, corpus management, obstacle bypassing |
| `clang-compiler-optimizations` | QA | Register allocation, ISel, LICM, vectorization triage, BOLT |
| `clang-clang` | Toolchain | Compiler flags, diagnostics, static analysis, clang-tidy basics |
| `clang-llvm` | Toolchain | LLVM IR, optimization passes, opt/llc, PGO workflow, LTO |
| `clang-format` | Toolchain | .clang-format files, configuration, editor integration |
| `clang-static-analysis` | Toolchain | clang-tidy config, PR diff analysis, auto-fix, CI integration |
| `clang-kernel-style` | Toolchain | Tab indentation, goto cleanup, subsystem boundaries, refcounting |
| `clang-doctrine` | Toolchain | Data supremacy, simplicity, hardware truth, bogus detector |
| `clang-bounds-safety` | Toolchain | Apple -fbounds-safety extension, pointer annotations, bounds traps |
| `clang-assembly` | Toolchain | x86-64 assembly, inline asm, ABI, SSE/AVX intrinsics |
| `clang-kernel-modules` | Toolchain | LKMs, Kbuild, /proc, sysfs, character devices, KGDB |

## How Skills Work

Each skill is a standalone `SKILL.md` file with YAML frontmatter that defines when to activate. Skills are loaded on demand — only when the topic matches the current task.

Skills cross-reference each other via `lobor/cc-skills-clang@<skill-name>` identifiers.

## Project Structure

```
skills/
  <skill-name>/
    SKILL.md          # Required: metadata + instructions
    references/       # Optional: detailed docs loaded on demand
    assets/           # Optional: templates, configs (.clang-format, etc.)
.claude-plugin/       # Plugin metadata
.cursor-plugin/       # Cursor plugin metadata
gemini-extension.json # Gemini CLI extension
```

## Contributing

1. Fork the repository
2. Create skills in `skills/<skill-name>/SKILL.md`
3. Follow the Agent Skills specification
4. Keep SKILL.md under 500 lines — use `references/` for depth
5. Submit a pull request

## License

MIT — see [LICENSE](LICENSE).
