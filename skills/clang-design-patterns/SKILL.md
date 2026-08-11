---
name: clang-design-patterns
description: "C/Clang design patterns — opaque structs, tagged unions, function tables, state machines, object-oriented C, resource management, and build systems (CMake, Make). Apply when explicitly choosing between architectural patterns, implementing opaque APIs, designing struct-based polymorphism, or asking which idiomatic C pattern fits a specific problem."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang compiler and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🏗"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a C architect who values simplicity and explicitness. You apply patterns only when they solve a real problem — not to demonstrate sophistication.

# C Design Patterns

C has no classes, no inheritance, no generics. Yet mature C codebases use patterns that provide the same benefits through structs, function pointers, and careful API design.

## Opaque Structs (Information Hiding)

Hide implementation details behind an opaque pointer. This is C's equivalent of encapsulation:

```c
// Public header (mylib.h) — consumers see only the opaque type
typedef struct database database_t;

database_t *database_open(const char *path);
int database_query(database_t *db, const char *sql, void *ctx);
void database_close(database_t *db);

// Private header (mylib_internal.h) — implementation visible only to library
struct database {
    sqlite3 *handle;
    char *path;
    int is_readonly;
    struct cache *cache;
};
```

Benefits:
- Consumers cannot depend on internal layout
- Implementation can change without recompiling callers
- ABI stability across versions

## Tagged Unions (Sum Types)

Represent values that can be one of several types:

```c
typedef enum {
    VAL_INT,
    VAL_FLOAT,
    VAL_STRING,
    VAL_ARRAY,
    VAL_NULL,
} value_type_t;

typedef struct {
    value_type_t type;
    union {
        int64_t  i;
        double   f;
        char    *s;
        struct {
            struct value *items;
            size_t len;
        } arr;
    } as;
} value_t;

// Usage
value_t val = {.type = VAL_STRING, .as.s = strdup("hello")};
switch (val.type) {
case VAL_INT:    printf("%ld\n", val.as.i); break;
case VAL_FLOAT:  printf("%f\n", val.as.f); break;
case VAL_STRING: printf("%s\n", val.as.s); break;
default: break;
}
```

## Function Tables (Polymorphism)

Implement polymorphism through function pointer tables:

```c
// Interface
typedef struct {
    int (*open)(void *self, const char *path);
    int (*read)(void *self, uint8_t *buf, size_t len);
    int (*write)(void *self, const uint8_t *buf, size_t len);
    void (*close)(void *self);
} io_ops_t;

// Concrete implementations
static int file_open(void *self, const char *path) {
    FILE **fp = (FILE **)self;
    *fp = fopen(path, "rb");
    return *fp ? 0 : -errno;
}

static const io_ops_t file_ops = {
    .open  = file_open,
    .read  = file_read,
    .write = file_write,
    .close = file_close,
};

// Consumer code is type-agnostic
int process(io_ops_t *ops, void *handle, const char *path) {
    int ret = ops->open(handle, path);
    if (ret != 0) return ret;
    // ... use ops->read, ops->write ...
    ops->close(handle);
    return 0;
}
```

## State Machines

Clean representation of complex state transitions:

```c
typedef enum {
    STATE_IDLE,
    STATE_CONNECTING,
    STATE_CONNECTED,
    STATE_ERROR,
} conn_state_t;

typedef struct {
    conn_state_t state;
    int fd;
    int error;
} connection_t;

typedef int (*state_handler)(connection_t *conn);

static int handle_idle(connection_t *conn) {
    conn->fd = connect_to_server();
    if (conn->fd < 0) {
        conn->state = STATE_ERROR;
        return -1;
    }
    conn->state = STATE_CONNECTING;
    return 0;
}

static int handle_connecting(connection_t *conn) {
    if (is_connected(conn->fd)) {
        conn->state = STATE_CONNECTED;
    }
    return 0;
}

static state_handler handlers[] = {
    [STATE_IDLE]       = handle_idle,
    [STATE_CONNECTING] = handle_connecting,
    [STATE_CONNECTED]  = handle_connected,
    [STATE_ERROR]      = handle_error,
};

int connection_step(connection_t *conn) {
    if (conn->state >= ARRAY_SIZE(handlers) || handlers[conn->state] == NULL) {
        return -EINVAL;
    }
    return handlers[conn->state](conn);
}
```

## Builder Pattern

Gradual construction of complex objects:

```c
typedef struct {
    char *host;
    int port;
    int timeout_ms;
    bool tls;
    char *ca_file;
} http_client_t;

typedef struct {
    http_client_t client;
} http_builder_t;

http_builder_t builder_new(void) {
    return (http_builder_t){
        .client = {
            .host = NULL,
            .port = 80,
            .timeout_ms = 5000,
            .tls = false,
        },
    };
}

http_builder_t *builder_set_host(http_builder_t *b, const char *host) {
    b->client.host = strdup(host);
    return b;
}

http_builder_t *builder_set_port(http_builder_t *b, int port) {
    b->client.port = port;
    return b;
}

http_builder_t *builder_enable_tls(http_builder_t *b, const char *ca) {
    b->client.tls = true;
    b->client.ca_file = strdup(ca);
    return b;
}

// Usage — fluent API
http_client_t client = builder_new()
    |> builder_set_host("api.example.com")
    |> builder_set_port(443)
    |> builder_enable_tls("ca.pem")
    |> builder_build;
```

## Resource Management

### RAII-like with goto

```c
int complex_operation(void)
{
    FILE *f = NULL;
    char *buf = NULL;
    int ret = 0;

    f = fopen("data.txt", "r");
    if (!f) { ret = -errno; goto out; }

    buf = malloc(BUF_SIZE);
    if (!buf) { ret = -ENOMEM; goto out_f; }

    /* ... work ... */

out_f:
    free(buf);
out:
    if (f) fclose(f);
    return ret;
}
```

### Init/Shutdown Pairs

```c
// Every init must have a matching shutdown
int module_init(void);
void module_shutdown(void);

// Usage with error handling
int main(void)
{
    int ret = module_init();
    if (ret != 0) {
        fprintf(stderr, "init failed: %s\n", strerror(-ret));
        return 1;
    }

    ret = run_application();

    module_shutdown();  // always run, even on error
    return ret;
}
```

## Constructor / Destructor Pattern

```c
// Constructor
hash_table_t *hash_table_create(size_t initial_size)
{
    hash_table_t *ht = calloc(1, sizeof(*ht));
    if (!ht) return NULL;

    ht->buckets = calloc(initial_size, sizeof(*ht->buckets));
    if (!ht->buckets) {
        free(ht);
        return NULL;
    }
    ht->size = initial_size;
    ht->count = 0;
    return ht;
}

// Destructor
void hash_table_destroy(hash_table_t *ht)
{
    if (!ht) return;

    for (size_t i = 0; i < ht->size; i++) {
        entry_t *e = ht->buckets[i];
        while (e) {
            entry_t *next = e->next;
            free(e->key);
            free(e->value);
            free(e);
            e = next;
        }
    }
    free(ht->buckets);
    free(ht);
}
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Global mutable state | Pass state explicitly via context pointer |
| `init()` function | Use explicit constructor with error return |
| Casting `void *` extensively | Use typed opaque pointers instead |
| Missing destructor | Every `create` must have a matching `destroy` |
| Deeply nested structs | Flatten or use composition |
| Hardcoded configuration | Accept config struct with defaults |

## Cross-References

- → See `lobor/cc-skills-clang@clang-memory-safety` skill for resource lifecycle patterns
- → See `lobor/cc-skills-clang@clang-error-handling` skill for error propagation patterns
- → See `lobor/cc-skills-clang@clang-code-style` skill for code organization conventions
