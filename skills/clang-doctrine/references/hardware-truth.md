# Hardware Truth — Why Hardware Sets the Limits

## The Principle

C is a thin layer over the machine. Ignoring hardware reality does not make it go away — it makes performance problems mysterious and hard to fix.

## Key Hardware Facts

### Cache Lines (64 bytes on x86_64)

- Accessing data outside the current cache line costs ~100 cycles (L2) to ~200+ cycles (L3/RAM)
- A struct with 10 fields at 8 bytes each = 80 bytes = 2 cache lines minimum
- Hot fields should be packed together; cold fields can go at the end

```c
// BAD: hot and cold fields interleaved
struct connection {
    int fd;                    // hot
    time_t last_active;        // hot
    char *log_buffer;          // cold
    char *temp_storage;        // cold
    int flags;                 // hot
};

// GOOD: hot fields together
struct connection {
    int fd;
    int flags;
    time_t last_active;
    // --- cache line boundary ---
    char *log_buffer;
    char *temp_storage;
};
```

### Branch Prediction

- Modern CPUs speculatively execute the predicted branch
- Misprediction penalty: ~15-20 cycles
- `__builtin_expect()` is not cosmetic — it rearranges generated code

```c
// Use sparingly, only when profile shows mispredictions
if (__builtin_expect(ptr != NULL, 1)) {
    // hot path: likely case first
}
```

### Memory Locality

- Array-of-structs (AoS) is better for iterating over all fields
- Struct-of-arrays (SoA) is better for processing one field across many elements
- Pick based on access pattern, not convention

### Lock Costs

- Uncontended mutex: ~20-50 ns
- Contended mutex: can block for microseconds
- `pthread_spin_lock`: cheap when contention is low, burns CPU when high
- Atomics (`_Atomic`): cheapest, but limited operations

## Review Checklist

1. **What is the hot path?** Profile before optimizing cold code.
2. **How many cache lines does the inner loop touch?** Fewer = faster.
3. **Are locks actually necessary?** Can the data be made read-only or copy-on-write?
4. **Is the struct size padding-wasteful?** Check with `_Static_assert(sizeof(x) == expected)`.

## The Litmus Test

> "If the hardware pays for the mistake, the mistake is yours."

Do not add `__attribute__((packed))` to "fix" a layout problem. Redesign the struct instead.
