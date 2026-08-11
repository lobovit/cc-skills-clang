---
name: clang-testing
description: "Production-ready C/Clang tests — unit testing with Check/CMocka/CUnit, integration testing, fuzzing with libFuzzer/AFL, test organization, mocking, coverage with gcov/llvm-cov, and CI integration. Use when writing or reviewing C tests, choosing a testing approach, setting up C test CI, or debugging flaky/slow tests."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🧪"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a C engineer who treats tests as executable specifications. You write tests to constrain behavior, not to hit coverage targets.

# C Testing Best Practices

## Test Frameworks

| Framework | Style | Best For |
| --- | --- | --- |
| [Check](https://libcheck.github.io/check/) | TAP output, fork-based | POSIX systems, TDD |
| [CMocka](https://cmocka.org/) | xUnit, lightweight | Embedded, cross-platform |
| [CUnit](https://sourceforge.net/projects/cunit/) | xUnit, GUI option | Desktop testing |
| [munit](https://nemequ.github.io/munit/) | Single-header, modern | Small projects |
| Unity + CMock | TCMock, lightweight | Embedded, bare-metal |
| GDB scripts | Manual assertions | Kernel, low-level |

**Recommended**: Use **munit** for new projects (single header, no deps, modern API) or **Check** for established POSIX projects.

## Test Structure

### File Organization

```
tests/
  test_hash_table.c     # unit tests for src/hash_table.c
  test_config.c         # unit tests for src/config.c
  test_integration.c    # integration tests
  test_main.c           # test runner (if not using a framework)
  fixtures/
    test_data.bin       # binary test fixtures
    config_valid.toml   # config test cases
```

Name test files after the source file they test:
```c
// test_hash_table.c — tests for hash_table.c
#include <munit.h>

static MunitResult test_insert(const MunitParameter params[], void *data)
{
    hash_table_t *ht = hash_table_create(16);
    munit_assert_not_null(ht);

    int ret = hash_table_insert(ht, "key", "value");
    munit_assert_int(ret, ==, 0);

    hash_table_destroy(ht);
    return MUNIT_OK;
}
```

### Table-Driven Tests

Test multiple scenarios with data tables:

```c
typedef struct {
    const char *name;
    int input;
    int expected;
} test_case_t;

static test_case_t cases[] = {
    {"positive",   42,  42},
    {"zero",        0,   0},
    {"negative",  -10, -10},
    {"overflow", INT_MAX, -1},  // expected failure
};

static MunitResult test_compute(const MunitParameter params[], void *data)
{
    for (size_t i = 0; i < sizeof(cases)/sizeof(cases[0]); i++) {
        munit_logf(MUNIT_LOG_INFO, "case: %s", cases[i].name);
        int result = compute(cases[i].input);
        munit_assert_int(result, ==, cases[i].expected);
    }
    return MUNIT_OK;
}
```

## Test Categories

### Unit Tests

Fast, isolated, deterministic. Run on every commit:

```c
// Each test is independent — no shared state between tests
static MunitResult test_parse_valid(const MunitParameter params[], void *data)
{
    config_t cfg = {0};
    int ret = config_parse("key=value", &cfg);
    munit_assert_int(ret, ==, 0);
    munit_assert_string_equal(cfg.key, "key");
    munit_assert_string_equal(cfg.value, "value");
    config_free(&cfg);
    return MUNIT_OK;
}
```

### Integration Tests

Test component interactions. Use build tags to separate:

```bash
# Compile integration tests separately
clang -DINTEGRATION_TEST -o test_int test_integration.c -lcheck
./test_int
```

### Edge Case Tests

Test boundary conditions explicitly:

```c
static MunitResult test_empty_input(const MunitParameter params[], void *data)
{
    munit_assert_null(parse(NULL, 0));
    return MUNIT_OK;
}

static MunitResult test_max_size(const MunitParameter params[], void *data)
{
    char buf[MAX_SIZE + 1];
    memset(buf, 'A', sizeof(buf));
    buf[sizeof(buf) - 1] = '\0';

    // Should handle gracefully, not crash
    int ret = process(buf, sizeof(buf));
    munit_assert_int(ret, ==, -E2BIG);
    return MUNIT_OK;
}
```

## Mocking in C

### Function Pointers (Dependency Injection)

```c
// Interface via function pointers
typedef struct {
    int (*read)(void *ctx, uint8_t *buf, size_t len);
    int (*write)(void *ctx, const uint8_t *buf, size_t len);
    void (*close)(void *ctx);
} io_ops_t;

// Production implementation
static int real_read(void *ctx, uint8_t *buf, size_t len)
{
    return fread(buf, 1, len, ctx);
}

// Mock for testing
static int mock_read(void *ctx, uint8_t *buf, size_t len)
{
    // Return canned data
    memcpy(buf, "mock data", len);
    return (int)len;
}
```

### Link-time Mocking

Compile test files with mock implementations:
```bash
# Link against mock instead of real implementation
clang -Dmock_db_connect=test_db_connect \
      -o test_app test_app.c mock_db.c
```

## Code Coverage

### gcov (GCC/Clang)

```bash
# Compile with coverage
clang --coverage -o test_app test_app.c

# Run tests
./test_app

# Generate coverage report
gcov -r src/*.c

# HTML report with llvm-cov
llvm-cov show ./test_app -format=html > coverage.html
llvm-cov report ./test_app
```

### Coverage Targets

| Metric | Target | Meaning |
| --- | --- | --- |
| Line coverage | ≥ 80% | Most code paths exercised |
| Branch coverage | ≥ 70% | Decision points tested |
| Function coverage | 100% | Every function called |

## Test Runner

Simple test runner without a framework:

```c
#include <stdio.h>

typedef int (*test_fn)(void);

typedef struct {
    const char *name;
    test_fn fn;
} test_t;

static int run_tests(test_t *tests, size_t count)
{
    int passed = 0, failed = 0;

    for (size_t i = 0; i < count; i++) {
        printf("  %-40s", tests[i].name);
        int result = tests[i].fn();
        if (result == 0) {
            printf("PASS\n");
            passed++;
        } else {
            printf("FAIL\n");
            failed++;
        }
    }

    printf("\n%d passed, %d failed\n", passed, failed);
    return failed > 0 ? 1 : 0;
}

// Usage
int main(void)
{
    test_t tests[] = {
        {"hash_table: insert",  test_ht_insert},
        {"hash_table: lookup",  test_ht_lookup},
        {"hash_table: remove",  test_ht_remove},
        {"config: parse valid", test_config_valid},
        {"config: parse NULL",  test_config_null},
    };
    return run_tests(tests, sizeof(tests)/sizeof(tests[0]));
}
```

## CMake Integration

```cmake
enable_testing()
find_package(Check REQUIRED)

add_executable(test_hash_table tests/test_hash_table.c src/hash_table.c)
target_link_libraries(test_hash_table Check::Check)

add_executable(test_config tests/test_config.c src/config.c)
target_link_libraries(test_config Check::Check)

add_test(NAME hash_table COMMAND test_hash_table)
add_test(NAME config COMMAND test_config)
```

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
ctest --test-dir build --output-on-failure
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Tests that depend on execution order | Each test must be independently runnable |
| Testing implementation details | Test observable behavior and public API |
| Missing edge case tests | Test NULL, empty, max-size, boundary values |
| No cleanup after tests | Every `malloc` has a matching `free` in test teardown |
| Ignoring test failures | CI must fail on any test failure |
| Hardcoded paths in tests | Use relative paths or temp directories |

## Cross-References

- → See `lobor/cc-skills-clang@clang-fuzzing` skill for automated test case generation
- → See `lobor/cc-skills-clang@clang-sanitizers` skill for finding bugs with runtime checks
- → See `lobor/cc-skills-clang@clang-safety` skill for defensive coding patterns to test
- → See `lobor/cc-skills-clang@clang-error-handling` skill for testing error paths
