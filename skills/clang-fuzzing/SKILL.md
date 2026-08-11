---
name: clang-fuzzing
description: "C/Clang fuzzing with libFuzzer and AFL. Use when writing fuzz harnesses, setting up corpus management, integrating fuzzing into CI, or overcoming fuzzing obstacles (checksums, global state, complex validation). Covers libFuzzer (LLVM), AFL++, corpus minimization, and coverage-guided fuzzing for C codebases."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🔍"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent
---

**Persona:** You are a C fuzzing engineer. You systematically explore code paths that manual testing misses, using coverage-guided feedback to find bugs in parsers, protocol handlers, and file format processors.

# C Fuzzing

## When to Fuzz

Fuzzing is most effective on:
- **Parsers** — file formats, network protocols, configuration files
- **Input validation** — any function that processes untrusted data
- **Serialization/deserialization** — JSON, XML, protobuf, custom formats
- **Cryptographic code** — edge cases in encoding/decoding

## libFuzzer Setup

libFuzzer is an in-process, coverage-guided fuzzer. It's the recommended starting point for Clang projects.

### Minimal Harness

```c
// fuzz_target.c
#include <stdint.h>
#include <stdlib.h>

// Function under test
int parse_input(const uint8_t *data, size_t len);

// libFuzzer entry point
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t len)
{
    if (len > 1024) return 0;  // skip oversized inputs

    parse_input(data, len);

    return 0;  // always return 0
}
```

### Build & Run

```bash
# Build with coverage and sanitizer
clang -fsanitize=fuzzer,address,undefined \
      -fno-omit-frame-pointer -g -O1 \
      fuzz_target.c -o fuzz_target

# Run with seed corpus
mkdir corpus
echo "test input" > corpus/test.txt
./fuzz_target corpus/ -max_total_time=60

# Run with crash reduction
./fuzz_target crash-corpus/ -reduce_inputs=1
```

### Seed Corpus

A good seed corpus dramatically improves coverage:

```bash
# Create initial seeds
echo "GET / HTTP/1.1" > corpus/http_get.txt
echo "POST /api HTTP/1.1" > corpus/http_post.txt
echo "" > corpus/empty.txt

# libFuzzer will mutate these to explore new paths
```

## AFL++ Setup

AFL++ is better for multi-core fuzzing and complex mutation strategies:

```bash
# Build with AFL++ instrumentation
afl-clang-fast -fsanitize=address -g -O1 \
    fuzz_target.c -o fuzz_target

# Run fuzzing
afl-fuzz -i corpus/ -o findings/ -- ./fuzz_target @@

# Multi-core
afl-fuzz -i corpus/ -o findings/ -M main -- ./fuzz_target @@
afl-fuzz -i corpus/ -o findings/ -S worker1 -- ./fuzz_target @@
afl-fuzz -i corpus/ -o findings/ -S worker2 -- ./fuzz_target @@
```

## Fuzzer Comparison

| Fuzzer | Best For | Complexity |
| --- | --- | --- |
| libFuzzer | Quick setup, single-project fuzzing | Low |
| AFL++ | Multi-core fuzzing, diverse mutations | Medium |
| Honggfuzz | Hardware-based coverage | Medium |

**Choose libFuzzer when:**
- You need simple, quick setup for C code
- Project uses Clang for compilation
- Single-core fuzzing is sufficient initially

## Overcoming Fuzzing Obstacles

### Checksums

Programs that verify checksums before processing block fuzzers:

```c
// Vulnerable — fuzzer can't guess valid checksum
int process_file(const uint8_t *data, size_t len) {
    uint32_t expected = crc32(data + 4, len - 4);
    if (*(uint32_t *)data != expected) return -1;  // fuzzer stuck
    return parse(data + 4, len - 4);
}

// Fixed — bypass checksum during fuzzing
#ifdef FUZZING
#define CHECKSUM_OK 1
#else
#define CHECKSUM_OK (crc32(data + 4, len - 4) == *(uint32_t *)data)
#endif

int process_file(const uint8_t *data, size_t len) {
    if (!CHECKSUM_OK) return -1;
    return parse(data + 4, len - 4);
}
```

Build with: `clang -DFUZZING -fsanitize=fuzzer ...`

### Global State

Global state makes fuzzing non-deterministic. Reset it between runs:

```c
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t len)
{
    reset_global_state();  // ensure deterministic behavior
    return process(data, len);
}
```

## Corpus Management

```bash
# Minimize corpus (remove redundant seeds)
afl-cmin corpus/ -i corpus/ -o corpus_min/ -- ./fuzz_target

# Minimize individual inputs
afl-tmin -i crash-input.bin -o crash-min.bin -- ./fuzz_target @@

# Check coverage
afl-showmap -i corpus/ -o map.txt -- ./fuzz_target
```

## CI Integration

```yaml
# GitHub Actions with libFuzzer
- name: Build fuzzer
  run: |
    clang -fsanitize=fuzzer,address,undefined \
          -fno-omit-frame-pointer -g -O1 \
          fuzz_target.c -o fuzz_target

- name: Run fuzzing (2 minutes)
  run: |
    ./fuzz_target corpus/ -max_total_time=120 -detect_leaks=1

- name: Run crashes through test suite
  run: |
    for crash in findings/crash-*; do
      ./test_from_input "$crash"
    done
```

## Reproducing Crashes

```bash
# libFuzzer — crashes are saved as files
./fuzz_target crash-<hash>

# AFL++ — crashes in findings/crash-*/
./fuzz_target findings/crash-<id>

# Reduce crash to minimal reproducer
afl-tmin -i crash.bin -o crash-min.bin -- ./fuzz_target @@
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| No seed corpus | Create diverse seeds covering different input types |
| Fuzzer rejects most inputs | Add input validation early in the harness |
| Fuzzer stuck on magic bytes | Add magic bytes as seed corpus entries |
| No sanitizer during fuzzing | Always combine with ASan+UBSan |
| Too short timeout | Set `-timeout=10` to catch infinite loops |
| Not minimizing crashes | Use `afl-tmin` or `-reduce_inputs` for minimal reproducers |

## Cross-References

- → See `lobor/cc-skills-clang@clang-sanitizers` skill for sanitizer configuration
- → See `lobor/cc-skills-clang@clang-security` skill for vulnerability patterns to fuzz for
- → See `lobor/cc-skills-clang@clang-testing` skill for test infrastructure
