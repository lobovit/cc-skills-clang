---
name: clang-security
description: "C/Clang security best practices and vulnerability prevention. Covers buffer overflows, format string attacks, integer overflow exploitation, use-after-free, race conditions, injection (command, SQL), memory corruption, stack canaries, and secure coding standards (CERT C). Apply when writing, reviewing, or auditing C code for security, or when working on crypto, I/O, secrets management, user input handling, or authentication."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🔒"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(clang-tidy:*) Bash(git:*) Agent WebFetch WebSearch AskUserQuestion
---

**Persona:** You are a C security engineer. You apply security thinking both when auditing existing code and when writing new code — threats are easier to prevent than to fix.

**Thinking mode:** Use `ultrathink` for security audits and vulnerability analysis. Security bugs hide in subtle interactions — deep reasoning catches what surface-level review misses.

# C Security

## Security Thinking Model

Before writing or reviewing code, ask three questions:

1. **What are the trust boundaries?** — Where does untrusted data enter?
2. **What can an attacker control?** — Which inputs flow into sensitive operations?
3. **What is the blast radius?** — If this defense fails, what's the worst outcome?

## Severity Levels

| Level | Meaning | Action |
| --- | --- | --- |
| Critical | RCE, full data breach | Fix immediately |
| High | Auth bypass, significant data exposure | Fix in current sprint |
| Medium | Limited exposure, session issues | Fix in next sprint |
| Low | Minor info disclosure, best-practice deviations | Fix opportunistically |

## Top Vulnerability Categories

### Buffer Overflows

The #1 C vulnerability class. Every buffer access must be bounds-checked:

```c
// Critical — stack buffer overflow
void process(char *input) {
    char buf[64];
    strcpy(buf, input);  // overflow if input > 63 bytes
}

// Fixed — bounded copy
void process(char *input) {
    char buf[64];
    snprintf(buf, sizeof(buf), "%s", input);
}
```

**Defenses:**
- Use `snprintf`, `strlcpy`, `strlcat` instead of `strcpy`, `strcat`, `sprintf`
- Always pass `sizeof(buf)` as the bound
- Enable `-D_FORTIFY_SOURCE=2` at compile time
- Enable stack protectors: `-fstack-protector-strong`

### Format String Vulnerabilities

User-controlled format strings can read/write arbitrary memory:

```c
// Critical — format string attack
printf(user_input);  // attacker controls format string

// Fixed — always use as literal format
printf("%s", user_input);
```

### Integer Overflow Exploitation

Integer overflow can bypass size checks:

```c
// Vulnerable — overflow bypasses check
size_t total = count * elem_size;  // overflow wraps to small value
void *buf = malloc(total);         // allocates too little

// Fixed — check before multiplication
if (count > 0 && elem_size > SIZE_MAX / count) {
    return -EOVERFLOW;
}
void *buf = malloc(count * elem_size);
```

### Use-After-Free

Accessing memory after it has been freed:

```c
// Vulnerable
struct entry *e = hash_lookup(table, key);
hash_remove(table, key);
process(e->data);  // use-after-free

// Fixed — validate before use, set to NULL after free
hash_remove(table, key);
e = NULL;
```

### Command Injection

Never construct shell commands from user input:

```c
// Critical — command injection
char cmd[256];
snprintf(cmd, sizeof(cmd), "echo %s", user_input);
system(cmd);  // attacker can inject: "; rm -rf /"

// Fixed — use execve with explicit args
pid_t pid = fork();
if (pid == 0) {
    execlp("echo", "echo", user_input, NULL);
    _exit(1);
}
```

### Race Conditions (TOCTOU)

Time-of-check-to-time-of-use bugs:

```c
// Vulnerable — TOCTOU race
if (access(path, W_OK) == 0) {  // check
    fd = open(path, O_WRONLY);   // use — file may have changed
}

// Fixed — open first, then check
fd = open(path, O_WRONLY | O_NOFOLLOW);
if (fd < 0) { return -errno; }
```

## Secure Compilation Flags

```bash
# Recommended security flags for Clang
clang -Wall -Wextra -Werror \
      -fstack-protector-strong \
      -D_FORTIFY_SOURCE=2 \
      -fPIE -pie \
      -Wl,-z,relro,-z,now \
      -Wl,-z,noexecstack \
      -o output src.c
```

| Flag | Protection |
| --- | --- |
| `-fstack-protector-strong` | Stack buffer overflow detection (canaries) |
| `-D_FORTIFY_SOURCE=2` | Buffer overflow detection in standard library calls |
| `-fPIE -pie` | Position-independent executable (ASLR) |
| `-Wl,-z,relro,-z,now` | Full RELRO — read-only GOT |
| `-Wl,-z,noexecstack` | Non-executable stack |
| `-fsanitize=address` | Runtime memory error detection (testing) |
| `-fsanitize=undefined` | Runtime undefined behavior detection (testing) |

## CERT C Top Rules

From the [CERT C Secure Coding Standard](https://sei.cert.org/confluence/pages/viewpage.action?pageId=703):

| Rule | Vulnerability | Fix |
| --- | --- | --- |
| ARR30-C | Buffer overflow | Bounds check all array accesses |
| INT32-C | Integer overflow | Validate before arithmetic |
| FLP34-C | Float to int conversion | Check range before cast |
| FIO35-C | `fclose` on error | Check `fclose` return value |
| MEM35-C | Allocate sufficient memory | Check `calloc` for overflow |
| STR31-C | Insufficient string bound | Use bounded string functions |
| MSC24-C | `argv` validation | Check `argc` before accessing `argv[i]` |
| CON33-C | Race condition | Use proper synchronization |

## Code Review Checklist

- [ ] All buffer accesses are bounds-checked
- [ ] No `strcpy`, `strcat`, `sprintf` without bounds
- [ ] No user input in format strings
- [ ] All `malloc`/`calloc` returns checked
- [ ] Integer arithmetic checked for overflow
- [ ] No TOCTOU races on file access
- [ ] No shell command construction from user input
- [ ] Secrets not hardcoded in source
- [ ] Error messages don't leak internal details
- [ ] Compilation uses security flags

## Cross-References

- → See `lobor/cc-skills-clang@clang-safety` skill for general defensive coding
- → See `lobor/cc-skills-clang@clang-sanitizers` skill for runtime vulnerability detection
- → See `lobor/cc-skills-clang@clang-fuzzing` skill for automated vulnerability discovery
- → See `lobor/cc-skills-clang@clang-error-handling` skill for error information leakage
