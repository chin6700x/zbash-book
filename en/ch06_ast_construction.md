# Chapter 6: Constructing a Compact Abstract Syntax Tree (AST)
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch06_ast_construction.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"Conceptual integrity is the most important consideration in system design."**  
> — **Fred Brooks** (The Mythical Man-Month)  
> 
> **"Keep it decoupled. The less your components know about each other, the more resilient your system becomes."**  
> — **David Parnas** (Pioneer of Modular Programming)

---

## 🧭 1. Why a Shell AST Avoids Deep Pointer Graphs

In general-purpose compilers (e.g., Clang, GCC) and interpreters (e.g., CPython), the **Abstract Syntax Tree (AST)** is typically structured as an extensive dynamic pointer graph comprising hundreds of specialized node types.  
Managing such trees requires frequent `malloc` and `free` operations, introducing two notable performance costs for interactive shells:
1. **Heap Fragmentation:** Scattering tiny metadata allocations across dynamic memory.
2. **CPU Pointer Chasing:** Traversing indirect memory addresses, which incurs CPU cache misses compared to sequential array reads.

A formal inspection of Unix shell grammar reveals that shell execution naturally resolves into **three well-defined structural tiers**:

```
[ Tier 1: Command Lists ] ──> Evaluated via logical operators (&&, ||, ;)
       │
       ▼
[ Tier 2: Pipeline ]       ──> Interconnected via data pipes (|)
       │
       ▼
[ Tier 3: Simple Command ] ──> Composed of argv[] + Redirections (<, >, 2>&1)
```

Because of this inherent hierarchy, `zbash` models its AST using **compact, flat contiguous memory blocks**, avoiding deep dynamic pointer graphs.

---

## 🏗️ 2. Flat Cache-Friendly Structures in `include/zbash.h`

Consider the definitions in [`include/zbash.h`](include/zbash.h#L73-L97):

```c
/* File: include/zbash.h (lines 73 - 97) */
typedef struct {
    char *argv[ZBASH_MAX_ARGS];   /* Argument vector array (ZBASH_MAX_ARGS = 64) */
    int argc;                     /* Total argument count */
    char *in_file;                /* < file */
    char *out_file;               /* > file */
    bool append_out;              /* >> file */
    char *err_file;               /* 2> file */
    bool append_err;              /* 2>> file */
    bool err_to_out;              /* 2>&1 */
    bool out_to_err;              /* >&2 or 1>&2 */
} zbash_cmd_t;

typedef enum {
    CONN_NONE,
    CONN_AND,     /* && */
    CONN_OR,      /* || */
    CONN_SEMI     /* ; */
} zbash_conn_t;

typedef struct {
    zbash_cmd_t cmds[ZBASH_MAX_PIPELINE];  /* Array of commands in pipeline (max 16) */
    int cmd_count;                         /* Number of pipeline commands */
    zbash_conn_t next_conn;                /* Connector to subsequent pipeline */
} zbash_pipeline_t;
```

### Architectural Benefits of Flat Arrays:
* **Zero Pointer Trees:** In `zbash_pipeline_t`, the array of 16 command structures (`cmds[16]`) is allocated as a single **contiguous memory block**.
* **High Cache Locality:** When modern 64-bit processors load a pipeline structure, its associated command metadata is brought into the **L1 Data Cache** in adjacent cache lines, eliminating indirect memory fetches during command execution.

---

## ⚙️ 3. Dissecting Redirections and Grouping in `src/parser/lexer.c`

Transforming a raw command string into a populated `zbash_cmd_t` node occurs within `zbash_parse_simple_cmd` in [`src/parser/lexer.c`](src/parser/lexer.c#L660-L750):

### 1) Detecting Compound Blocks `{ ... }` and `( ... )`
```c
/* File: src/parser/lexer.c (lines 669 - 716) */
    /* Detect Compound Brace Grouping { ... } */
    if (cmd_str[i] == '{' && (i + 1 == len || isspace((unsigned char)cmd_str[i + 1]))) {
        size_t b_start = i + 1;
        int depth = 1;
        bool in_s = false, in_d = false;
        size_t j = b_start;
        while (j < len && depth > 0) {
            char c = cmd_str[j];
            if (c == '\'' && !in_d) in_s = !in_s;
            else if (c == '"' && !in_s) in_d = !in_d;
            else if (c == '{' && !in_s && !in_d) depth++;
            else if (c == '}' && !in_s && !in_d) {
                depth--;
                if (depth == 0) break;
            }
            j++;
        }
        if (depth == 0) {
            cmd->argv[0] = strdup("{");
            // Isolate encapsulated inner command block as argument index 1
            size_t inner_len = j - b_start;
            char *inner = malloc(inner_len + 1);
            strncpy(inner, cmd_str + b_start, inner_len);
            inner[inner_len] = '\0';
            cmd->argv[1] = inner;
            cmd->argv[2] = NULL;
            cmd->argc = 2;
            i = j + 1;
        }
    }
```

* **Bracket Depth Tracking:** The parser tracks brace nesting depth (`depth`) while simultaneously evaluating quoting contexts (`in_s`, `in_d`). This prevents braces enclosed inside literal string quotes (e.g., `echo "{}"`) from erroneously triggering compound group blocks.

### 2) Zero-Latency Redirection Lookahead
```c
/* File: src/parser/lexer.c (lines 735 - 750) */
        /* Detect Standard Error to Standard Output merge (2>&1) */
        if (cmd_str[i] == '2' && i + 3 < len && cmd_str[i + 1] == '>' && cmd_str[i + 2] == '&' && cmd_str[i + 3] == '1') {
            cmd->err_to_out = true;
            i += 4;
            continue;
        }
        /* Detect Standard Output to Standard Error redirection (1>&2 or >&2) */
        if (cmd_str[i] == '1' && i + 3 < len && cmd_str[i + 1] == '>' && cmd_str[i + 2] == '&' && cmd_str[i + 3] == '2') {
            cmd->out_to_err = true;
            i += 4;
            continue;
        }
        if (cmd_str[i] == '>' && i + 2 < len && cmd_str[i + 1] == '&' && cmd_str[i + 2] == '2') {
            cmd->out_to_err = true;
            i += 3;
            continue;
        }
```

* **Constant-Time Lookahead:** By examining a 3- to 4-byte window, patterns like `2>&1` and `>&2` are identified immediately. The parser configures boolean flags on the `cmd` node and advances the scanning index, keeping parser throughput strictly linear ($O(N)$).

---

## 🏁 Summary

1. **Avoid Accidental Complexity:** Shells benefit from flat, contiguous data structures rather than multi-layered pointer trees.
2. **Cache-Aligned Performance:** Embedding `zbash_cmd_t` directly into `zbash_pipeline_t` ensures high cache hit rates and eliminates dynamic heap allocation overhead.
3. **Quote-Aware Enclosure Scanning:** Tracking nesting depth alongside quote states guarantees robust handling of nested subshells and group expressions.

In **[Chapter 7: The Art of Parameter Expansion: Slicing, Defaults, and Lengths](ch07_parameter_expansion.md)**, we examine how parameter expansions—such as `${VAR:-def}`, `${#arr[@]}`, and `${VAR:offset:len}`—are implemented in pure C with zero-copy slicing.
