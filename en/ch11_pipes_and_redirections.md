# Chapter 11: Pipes and Redirections: pipe(), dup2(), and Trailing Loops
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch11_pipes_and_redirections.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"We should have some ways of connecting programs like garden hose--screw in another segment when it becomes necessary to massage data in another way."**  
> — **Doug McIlroy** (Originator of Unix Pipes, 1973)  
> 
> **"The UNIX system provides a convenient and uniform way for programs to communicate with each other: pipes."**  
> — **Brian W. Kernighan**

---

## 🧭 1. The Garden Hose of Unix

If operating systems are likened to automated manufacturing plants, **Unix Pipes** represent the most consequential assembly lines in computer history.  
Prior to 1973, programs operated in strict isolation: reading disk blocks, writing intermediate results to tape or temporary files, and terminating. Doug McIlroy formulated a revolutionary alternative: Why not establish unidirectional in-memory data channels between distinct processes so that the stdout stream of one flows directly into the stdin stream of another without hitting physical storage?

In shell scripts, invocations such as:
```bash
cat access.log | grep "404" | cut -d' ' -f1 | sort | uniq -c > summary.txt
```
appear deceptively straightforward. Yet behind this single line, the kernel must configure file descriptor tables, redirect standard streams using `dup2()`, construct inter-process byte streams via `pipe()`, and supervise descriptor closure across concurrent process hierarchies.

In this chapter, we dissect the **Plumbing and Redirection Engine** in `zbash` and reveal the resolution of the notorious **"Trailing Loop Redirection Trap (`done < file`)"**.

---

## 🏗️ 2. File Descriptor Tables

Under Unix standards, every process inherits three foundational standard file descriptors:
* **`0` (`STDIN_FILENO`):** Standard input (keyboard or redirected input stream).
* **`1` (`STDOUT_FILENO`):** Standard output (terminal console or redirected output target).
* **`2` (`STDERR_FILENO`):** Standard error (diagnostic output stream).

```
        Standard Streams                 Redirection Operators
    ┌───────────────────────┐            ┌───────────────────────┐
    │  0: STDIN_FILENO      │ ◄───────── │ < file,  <<< "string" │
    ├───────────────────────┤            ├───────────────────────┤
    │  1: STDOUT_FILENO     │ ─────────► │ > file,  >> file      │
    ├───────────────────────┤            ├───────────────────────┤
    │  2: STDERR_FILENO     │ ─────────► │ 2> file, 2>> file     │
    └───────────────────────┘            ├───────────────────────┤
                                         │ 2>&1  (ERR ➔ OUT)     │
                                         │ >&2   (OUT ➔ ERR)     │
                                         └───────────────────────┘
```

---

## 🔍 3. Dissecting `apply_redirections` in `src/core/engine.c`

Consider descriptor switching in [`src/core/engine.c`](src/core/engine.c#L716-L758):

```c
/* File: src/core/engine.c (lines 716 - 758) */
static int apply_redirections(zbash_cmd_t *cmd) {
    /* 1. Input Redirection: < input.txt */
    if (cmd->in_file) {
        int fd = open(cmd->in_file, O_RDONLY);
        if (fd < 0) {
            fprintf(stderr, "zbash: %s: %s\n", cmd->in_file, strerror(errno));
            return -1;
        }
        dup2(fd, STDIN_FILENO);
        close(fd);
    }

    /* 2. Output Redirection: > out.txt or >> out.txt */
    if (cmd->out_file) {
        int flags = O_WRONLY | O_CREAT | (cmd->append_out ? O_APPEND : O_TRUNC);
        int fd = open(cmd->out_file, flags, 0666);
        if (fd < 0) {
            fprintf(stderr, "zbash: %s: %s\n", cmd->out_file, strerror(errno));
            return -1;
        }
        dup2(fd, STDOUT_FILENO);
        close(fd);
    }

    /* 3. Diagnostic Stream Redirection: 2> err.log or 2>> err.log */
    if (cmd->err_file) {
        int flags = O_WRONLY | O_CREAT | (cmd->append_err ? O_APPEND : O_TRUNC);
        int fd = open(cmd->err_file, flags, 0666);
        if (fd < 0) {
            fprintf(stderr, "zbash: %s: %s\n", cmd->err_file, strerror(errno));
            return -1;
        }
        dup2(fd, STDERR_FILENO);
        close(fd);
    }

    /* 4. Merge Error into Output: 2>&1 */
    if (cmd->err_to_out) {
        dup2(STDOUT_FILENO, STDERR_FILENO);
    }

    /* 5. Direct Output to Error: >&2 */
    if (cmd->out_to_err) {
        dup2(STDERR_FILENO, STDOUT_FILENO);
    }

    return 0;
}
```

### Why `close(fd)` Immediately Following `dup2()` is Mandatory
The system call `dup2(fd, STDOUT_FILENO)` clones descriptor `fd` onto target index 1 (`STDOUT`). If the original `fd` is not closed immediately, two open descriptors reference the identical file description table. In long-running shell scripts, neglecting this cleanup exhausts the process limit (`ulimit -n: Too many open files`).

---

## ⚡ 4. Here-Strings (`<<<`) and Here-Documents (`<<`)

Shell developers rely on direct string ingestion via Here-Strings:
```bash
grep "needle" <<< "haystack with needle inside"
```
and multi-line declarations via Here-Documents:
```bash
cat << 'EOF' > config.json
{
  "status": "active"
}
EOF
```

In `zbash`, Here-Strings materialize via anonymous temporary inodes in `/tmp/zbash_hs_XXXXXX`. Once command execution completes, `free_cmd_strings` in [`src/core/engine.c`](src/core/engine.c#L760-L776) instantly unlinks the file from the filesystem:

```c
/* File: src/core/engine.c (lines 767 - 773) */
    if (cmd->in_file) {
        if (strncmp(cmd->in_file, "/tmp/zbash_hs_", 14) == 0) {
            unlink(cmd->in_file); /* Unlink file from disk inode table immediately! */
        }
        free(cmd->in_file);
        cmd->in_file = NULL;
    }
```
Immediate `unlink()` ensures zero residual storage artifacts on disk even when scripts process hundreds of thousands of Here-Strings per session.

---

## 🪤 5. The Classic Trap: Trailing Loop Redirection (`done < file`)

During parity testing, an intricate edge case emerged in `test_07_pipelines_redirections.sh` (Test #77):

```bash
# Ingest lines from file sequentially
while read line; do
    echo "Processing: $line"
done < input_data.txt

# Subsequent command reading from primary standard input
read next_cmd
```

### The Initial Failure Mode:
When the `while` loop concluded, the subsequent `read next_cmd` failed and returned `EOF` immediately.  
**Root Cause:** Redirecting `< input_data.txt` overwrote `STDIN_FILENO (0)` in the primary shell process. Upon reaching file exhaustion, the shell lost its original connection to the standard input channel permanently!

### The Robust Architectural Remedy:
Consider lines 1446 to 1465 in [`src/core/engine.c`](src/core/engine.c):

```c
/* File: src/core/engine.c (lines 1446 - 1465) */
    zbash_cmd_t redir_cmd;
    memset(&redir_cmd, 0, sizeof(redir_cmd));
    const char *rp = p;
    while (*rp && isspace((unsigned char)*rp)) rp++;
    bool has_redir = false;
    int saved_in = -1, saved_out = -1, saved_err = -1;
    if (*rp) {
        zbash_parse_simple_cmd(state, rp, &redir_cmd);
        if (redir_cmd.in_file || redir_cmd.out_file || redir_cmd.err_file || 
            redir_cmd.err_to_out || redir_cmd.out_to_err) {
            has_redir = true;
            /* 1. Preserve active shell file descriptors prior to entering loop! */
            saved_in = dup(STDIN_FILENO);
            saved_out = dup(STDOUT_FILENO);
            saved_err = dup(STDERR_FILENO);

            /* 2. Switch streams exclusively for loop execution */
            if (apply_redirections(&redir_cmd) != 0) {
                close(saved_in); close(saved_out); close(saved_err);
                free_cmd_strings(&redir_cmd);
                return 1;
            }
        }
    }
```

Upon loop completion (line 1505):
```c
    /* 3. Restore original system descriptors 100% */
    if (has_redir) {
        dup2(saved_in, STDIN_FILENO);
        dup2(saved_out, STDOUT_FILENO);
        dup2(saved_err, STDERR_FILENO);
        close(saved_in);
        close(saved_out);
        close(saved_err);
        free_cmd_strings(&redir_cmd);
    }
```

The disciplined sequence **Save ➔ Redirect ➔ Execute ➔ Restore** preserves input stream continuity across compound loops without side-effect pollution.

---

## 🧪 6. Differential Test Verification

In `test/1000-test/test_07_pipelines_redirections.sh`, 100 tests validate descriptor manipulation:

| Test Group | Pattern | Validated Mechanism | Result vs macOS Bash |
| :--- | :--- | :--- | :--- |
| **Simple Redirections** | `echo foo > out.txt`, `cat < in.txt` | Inode creation and truncation | **PASS (100%)** |
| **Append Mode** | `echo bar >> out.txt` | `O_APPEND` non-destructive append | **PASS (100%)** |
| **Error Merging** | `cmd 2>&1`, `cmd >&2` | Atomic `dup2()` ordering | **PASS (100%)** |
| **Multi-Stage Pipes** | `seq 100 \| grep 5 \| wc -l` | Multi-process concurrent flow | **PASS (100%)** |
| **Trailing Loop Redirection** | `while read; do ... done < data` | Descriptor save and restoration | **PASS (100%)** |
| **Here-String & Expansion** | `grep needle <<< "$VAR"` | Temporary inode unlinking | **PASS (100%)** |

**Module Summary:** `100 / 100 PASS (100.0%)` exact match with macOS Bash.

---

## 🏁 Summary

1. **`dup2()` Is Fundamental to Unix I/O:** Understanding the difference between `dup()` (allocating the lowest free descriptor) and `dup2()` (atomically overwriting a specified target) is essential.
2. **State Preservation Integrity:** Whenever changing I/O paths on the parent process, save original descriptors with `dup()` and restore them with `dup2()`.
3. **Closing Unused Write Ends:** Leaving write descriptors open in sender processes blocks EOF signals down the pipeline, hanging downstream filters indefinitely.

In **[Chapter 12: Control Flow and Loops: while, for, until, if-then-else, case](ch12_control_flow_loops.md)**, we examine recursive evaluation of structured branching and loop controls in pure C.
