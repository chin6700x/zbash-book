# Chapter 3: Architectural Overview: From Keystroke to Process Execution
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch03_architecture_overview.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"A shell is not just a language for writing scripts; it is a direct extension of your thoughts into the operating system."**  
> — **Stephen Bourne** (1979)  
> 
> **"Data dominates. If you've chosen the right data structures and organized things well, the algorithms will almost always be self-evident."**  
> — **Rob Pike** (1989)

---

## 🧭 1. The Lifecycle of a Command

When a user submits a command line in their terminal, such as:
```bash
echo "Hello World" | grep Hello > result.txt
```
and presses **[Enter]**, what transpires beneath the surface within the core of `zbash`?

Rather than treating this conceptually, we trace each transformation through the codebase:

```
[ 1. Input Source ] ──> [ 2. SIMD Lexer ] ──> [ 3. AST Parser ] ──> [ 4. Variable Expander ] ──> [ 5. Execution Engine ]
 (stdin / -c / file)    (16 bytes / cycle)     (Pipeline AST graph)   (Expands $VAR, $((..)))     (Fork, Pipe, Execvp)
```

---

## 🚀 2. Ingestion: In-Memory Streaming with `fmemopen()` in `src/main.c`

`zbash` handles command input across three primary modes:
1. Command string execution via `-c`: e.g., `zbash -c "echo 123"`
2. Script file execution: e.g., `zbash script.sh`
3. Interactive terminal sessions: when launched without file or inline arguments

Consider the entry point in [`src/main.c`](src/main.c#L70-L93):

```c
/* File: src/main.c (lines 70 - 93) */
    if (cmd_string) {
        state.argc = (argc > opt_idx) ? (argc - opt_idx + 1) : 1;
        state.argv = (argc > opt_idx) ? (argv + opt_idx - 1) : argv;
        FILE *mem_fp = fmemopen((char *)cmd_string, strlen(cmd_string), "r");
        if (mem_fp) {
            zbash_run_stream(&state, mem_fp);
            fclose(mem_fp);
        } else {
            zbash_run_line(&state, cmd_string);
        }
        return state.should_exit ? state.exit_code : state.last_status;
    }

    if (opt_idx < argc) {
        const char *script_file = argv[opt_idx];
        state.argc = argc - opt_idx;
        state.argv = argv + opt_idx;
        return zbash_run_file(&state, script_file);
    }

    state.argc = 1;
    state.argv = argv;
    return zbash_run_interactive(&state);
```

### The Architectural Role of `fmemopen()`:
* In line 73, rather than writing a temporary file to disk or duplicating data through slow string buffers, `zbash` leverages the POSIX **`fmemopen()`** interface.
* This interface wraps the in-memory string `cmd_string` directly into a standard `FILE*` stream without physical storage I/O, allowing `zbash_run_stream` to parse input streams at system memory speeds.

---

## 🏗️ 3. Core Data Structures: `zbash_cmd_t` and `zbash_pipeline_t`

Clean architecture begins with disciplined data layout. In [`include/zbash.h`](include/zbash.h#L73-L97), commands and pipelines are modeled as explicit, flat structures:

### 1) Discrete Command Representation (`zbash_cmd_t`)
```c
/* File: include/zbash.h (lines 73 - 84) */
typedef struct {
    char *argv[ZBASH_MAX_ARGS];   /* Argument vector (maximum 64 entries) */
    int argc;                     /* Total argument count */
    char *in_file;                /* Input redirection (< file) */
    char *out_file;               /* Output redirection (> file) */
    bool append_out;              /* Append output mode (>> file) */
    char *err_file;               /* Standard error redirection (2> file) */
    bool append_err;              /* Append standard error mode (2>> file) */
    bool err_to_out;              /* Merge stderr into stdout (2>&1) */
    bool out_to_err;              /* Direct stdout into stderr (>&2 or 1>&2) */
} zbash_cmd_t;
```
* **Explicit Redirection Flags:** Notice that complex file descriptor merges like `2>&1` are tracked via direct boolean fields such as `err_to_out` (line 81). This allows the execution engine to configure descriptor tables atomically without reparsing string tokens during process spawn.

### 2) Pipeline Sequences and Logical Connectors (`zbash_pipeline_t`)
```c
/* File: include/zbash.h (lines 85 - 97) */
typedef enum {
    CONN_NONE,
    CONN_AND,     /* && (execute next if previous succeeded) */
    CONN_OR,      /* || (execute next if previous failed) */
    CONN_SEMI     /* ;  (sequential execution regardless of status) */
} zbash_conn_t;

typedef struct {
    zbash_cmd_t cmds[ZBASH_MAX_PIPELINE];  /* Commands joined via | (up to 16) */
    int cmd_count;                         /* Total commands in current pipeline */
    zbash_conn_t next_conn;                /* Connector to subsequent pipeline (&&, ||, ;) */
} zbash_pipeline_t;
```
* Through this representation, a compound command sequence such as:
  `cat data.txt | grep error | wc -l && echo "Done"`
  is organized into:
  - **Pipeline 1:** 3 commands (`cat`, `grep`, `wc`) connected via pipes (`|`), with `next_conn = CONN_AND`.
  - **Pipeline 2:** 1 command (`echo "Done"`), with `next_conn = CONN_NONE`.

---

## ⚙️ 4. Dispatcher Engine: Command Resolution in `src/core/engine.c`

Once the parser produces a populated `zbash_pipeline_t`, control transfers to the execution dispatcher in [`src/core/engine.c`](src/core/engine.c#L1037-L1070):

```c
/* File: src/core/engine.c (lines 1037 - 1070) */
// 1. Evaluate Command Grouping, e.g., { echo 1; echo 2; }
if (strcmp(cmd->argv[0], "{") == 0) {
    int rc = (cmd->argc > 1 && cmd->argv[1]) ? zbash_run_line(state, cmd->argv[1]) : 0;
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 2. Evaluate Subshell Isolation, e.g., ( cd /tmp && ls )
if (strcmp(cmd->argv[0], "(") == 0) {
    int rc = (cmd->argc > 1 && cmd->argv[1]) ? zbash_run_line(state, cmd->argv[1]) : 0;
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 3. Resolve User-Defined Shell Functions
zbash_func_t *fn = zbash_find_func(state, cmd->argv[0]);
if (fn) {
    int rc = zbash_exec_func(state, fn, cmd->argc, cmd->argv);
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 4. Resolve Shell Builtins (cd, export, echo, read, etc.)
if (zbash_is_builtin(cmd->argv[0])) {
    int rc = zbash_exec_builtin(state, cmd->argc, cmd->argv);
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 5. Fallback: Dispatch External Operating System Binary
execvp(cmd->argv[0], cmd->argv);
fprintf(stderr, "zbash: %s: %s\n", cmd->argv[0], strerror(errno));
_exit((errno == ENOENT) ? 127 : 126);
```

### Command Execution Precedence:
1. **Compound Enclosures (`{` or `(`):** Route encapsulated statements to the nested execution evaluator.
2. **User-Defined Functions:** Match against `state->funcs` in user scope before checking external paths.
3. **Shell Builtins:** Execute in-process C implementations for state-modifying operations like `cd`, `export`, and `read`.
4. **External System Executables (`execvp`):** Delegate path resolution to the operating system kernel across `$PATH`.
5. **Strict POSIX Exit Codes:** Return exit code **`127`** when an executable is missing (`ENOENT`), and **`126`** on permission failure (`EACCES`).

---

## 🚰 5. Pipeline Plumbing and Exit Status Extraction

In Unix systems, commands connected by pipes (`cmd1 | cmd2`) execute concurrently across dedicated child processes.

Examining process creation in [`src/core/engine.c`](src/core/engine.c#L1007-L1031):

```c
/* File: src/core/engine.c (lines 1007 - 1031) */
    int pipe_fds[64][2];
    for (int i = 0; i < n - 1; i++) {
        if (pipe(pipe_fds[i]) < 0) {
            perror("zbash: pipe");
            return 1;
        }
    }

    pid_t pids[64];
    for (int i = 0; i < n; i++) {
        zbash_cmd_t *cmd = &pipeline->cmds[i];
        pids[i] = fork();

        if (pids[i] == 0) {
            // Non-first processes read from previous pipe's read-end
            if (i > 0) {
                dup2(pipe_fds[i - 1][0], STDIN_FILENO);
            }
            // Non-last processes write to current pipe's write-end
            if (i < n - 1) {
                dup2(pipe_fds[i][1], STDOUT_FILENO);
            }

            // Close all inherited pipe descriptors in child to prevent hanging
            for (int j = 0; j < n - 1; j++) {
                close(pipe_fds[j][0]);
                close(pipe_fds[j][1]);
            }
```

### The Critical Role of Parent Descriptor Cleanup:
Consider lines 1076 to 1089:

```c
/* File: src/core/engine.c (lines 1076 - 1089) */
    for (int j = 0; j < n - 1; j++) {
        close(pipe_fds[j][0]);
        close(pipe_fds[j][1]);
    }

    int last_status = 0;
    for (int i = 0; i < n; i++) {
        int status = 0;
        waitpid(pids[i], &status, 0);
        if (i == n - 1) {
            if (WIFEXITED(status)) last_status = WEXITSTATUS(status);
            else if (WIFSIGNALED(status)) last_status = 128 + WTERMSIG(status);
        }
    }
    return last_status;
```

#### Why Parent Descriptor Closure Is Mandatory:
* If the parent shell process retains an open write-end (`pipe_fds[j][1]`), downstream filter utilities like `grep` or `wc` will never receive an **EOF (End-Of-File)** indicator and will wait indefinitely for input.
* In line 1085, `last_status` captures specifically the exit condition of the rightmost pipeline command (`i == n - 1`), satisfying the POSIX requirement for `$?`.

---

## 🏁 Summary

In this chapter, we reviewed the architectural backbone of `zbash`:
1. In-memory command ingestion using `fmemopen()` in `src/main.c`.
2. Compact representations in `include/zbash.h` (`zbash_cmd_t`, `zbash_pipeline_t`) with explicit redirection semantics.
3. Execution resolution order in `src/core/engine.c` ensuring proper evaluation precedence.
4. Concurrency management using `pipe()`, `dup2()`, descriptor isolation, and `waitpid()`.

In **Part II: Lexer & Parser Architecture ([Chapter 4](ch04_simd_vector_lexer.md))**, we explore the hardware layer: analyzing the **128-bit ARM64 NEON Assembly** engine that accelerates lexical scanning and delimiter detection.
