# Data Supremacy — Why Data Structure is the Design

## The Principle

In C, data layout IS the program. Unlike languages with vtables, GC, or runtime dispatch, the struct definition determines:
- Cache behavior (field ordering, padding, alignment)
- Memory allocation strategy (stack vs heap, arena vs malloc)
- Algorithm complexity (linear scan vs indexed lookup)
- Thread safety (copy-on-write, lock-free access patterns)

## Practical Examples

### Anti-pattern: Conditional soup

```c
// WRONG: data shape forces conditionals everywhere
struct request {
    int type;
    char *url;
    char *body;
    int status;
    char *error;
    int timeout;
    int retries;
};

// Every function needs:
if (req->type == REQ_GET) { ... }
if (req->type == REQ_POST) { ... }
if (req->error != NULL) { ... }
if (req->timeout > 0) { ... }
```

### Pattern: Data shape eliminates branches

```c
// RIGHT: separate types for separate concerns
struct get_request {
    const char *url;
    int timeout;
};

struct post_request {
    const char *url;
    const char *body;
};

struct request_result {
    int status;
    char *error;
};
```

Now each function handles one case. No conditionals. The compiler can inline, devirtualize, and vectorize.

## Review Checklist

1. **Can you draw the memory layout?** If not, you don't understand the data.
2. **How many branches exist because of the data shape?** Fewer is better.
3. **Is the common case the first field or the first branch?** Optimize for frequency.
4. **Are there any "type" fields that could be separate structs?** Discriminated unions > tagged fields.
5. **Does the struct fit in a cache line?** 64 bytes is the sweet spot on x86_64.

## The Litmus Test

> "If the data layout cannot be explained clearly, the patch is not ready."

Ask the author to draw the struct in memory. If they can't, they don't understand what they're proposing.
