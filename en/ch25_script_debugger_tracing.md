# Chapter 25: Script Debugging & Execution Tracing: Building -x, -v, -n, and $LINENO
> **"The Shell Debugger Engine: Execution Tracing, Syntax Dry-Runs, and Error Trapping in Pure C"**  
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **Source Repository:** `zbash/docs/book/en/ch25_script_debugger_tracing.md`  
> **License:** [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

---

> [!NOTE]
> ### 🏛️ Chapter Epigraph
> **"A great tool does not just execute commands blindly; it illuminates the path of execution so the engineer can see."**  
> — **Wirot Chookeaw**  
> 
> **"The most effective debugging tool is still careful thought, coupled with judiciously placed print statements."**  
> — **Brian W. Kernighan**  

---

## 🧭 1. When a 1,000-Line Script Fails

Consider a realistic dilemma faced by systems engineers and DevOps practitioners:  
You maintain a 1,000-line shell script responsible for orchestrating continuous integration, automated builds, or cloud infrastructure provisioning.  
During execution, the script suddenly fails halfway through, printing a solitary, ambiguous message:
```
error: missing file
```
The immediate systems question is: **Which line in those 1,000 lines caused the failure? Which variable expanded to an empty string? Which loop iteration passed an invalid parameter?**

In the Unix shell ecosystem, the indispensable tool engineers turn to is **Execution Tracing**:
```bash
$ zbash -x deploy.sh
```
Or by surgically inserting `set -x` and `set +x` around specific code sections to inspect variable-expanded commands line by line before execution!

The architectural question is:  
> *"Behind switches like `-x` (xtrace), `-v` (verbose), `-n` (noexec), and `-e` (errexit), how must a shell pipeline order its internal C operations so that tracing is faithful, robust, and completely non-intrusive to program output?"*

In this chapter, we dissect the internal implementation of the **In-Process Debugger & Tracing Engine** within `zbash`.

---

## 🏗️ 2. The Four Pillars of Shell Script Debugging

In standard POSIX specification, a compliant shell provides four fundamental operational switches for debugging and error enforcement:

| Flag | Mode Name | Systems Semantics & Behavior |
| :---: | :--- | :--- |
| **`-x`** | **`xtrace` (Execution Trace)** | Fully expands parameters, variables, and wildcards, then formats and prints the concrete command prefixed with `+ ` to `STDERR` immediately prior to execution. |
| **`-v`** | **`verbose` (Verbose Input)** | Prints raw input lines to `STDERR` as they are read from input, prior to any tokenization, expansion, or substitution. |
| **`-n`** | **`noexec` (Syntax Dry-Run)** | Reads and parses full grammar into abstract syntax trees to validate syntax, but **bypasses execution entirely**. |
| **`-e`** | **`errexit` (Exit on Error)** | Terminates the script immediately if any simple command returns a non-zero exit status ($? \neq 0$). |

---

## 🔍 3. Dissecting `xtrace` and `noexec` Implementation in `engine.c`

Consider the production implementation in [`src/core/engine.c`](src/core/engine.c), lines 2282 through 2310:

```c
/* File: src/core/engine.c (Lines 2282 - 2310) */
    /* 1. xtrace mode (-x): Emit expanded command tokens to STDERR */
    if (state->xtrace) {
        for (int k = 0; k < pipelines[i].cmd_count; k++) {
            fprintf(stderr, "+ ");
            for (int a = 0; a < pipelines[i].cmds[k].argc; a++) {
                fprintf(stderr, "%s ", pipelines[i].cmds[k].argv[a]);
            }
            fprintf(stderr, "\n");
        }
    }

    /* 2. noexec mode (-n): When enabled, completely bypass pipeline execution! */
    if (!state->noexec) {
        state->last_status = execute_pipeline(state, &pipelines[i]);
        if (invert) {
            state->last_status = (state->last_status == 0) ? 1 : 0;
        }
    }

    /* 3. Reclaim command memory */
    prev_conn = pipelines[i].next_conn;
    for (int k = 0; k < pipelines[i].cmd_count; k++) {
        free_cmd_strings(&pipelines[i].cmds[k]);
    }

    /* 4. errexit mode (-e): Terminate execution immediately on non-zero exit status */
    if (state->errexit && state->last_status != 0) {
        state->should_exit = true;
        state->exit_code = state->last_status;
        break;
    }
```

### Why This Ordering Is Architecturally Sound:
1. **Post-Expansion Tracing:** Notice that `xtrace` operates on `pipelines[i].cmds[k].argv`. At this juncture, parameter expansion, command substitution, and quote removal have completed. The user sees the exact concrete strings that will be dispatched to `execve()`.
2. **Dedicated Channel on `STDERR`:** Routing trace statements strictly to file descriptor 2 (`STDERR`) prevents pollution of standard output (`STDOUT`), preserving pipeline integrity during downstream pipe redirection (`cmd1 | cmd2`).
3. **Safe Dry-Runs with `noexec`:** Allows developers and CI/CD pipelines to validate script syntax without risking the execution of destructive commands (such as `rm -rf /tmp/data`).

---

## ⚡ 4. Runtime Dynamic Toggling via the Builtin `set`

In addition to command-line invocation flags (e.g., `zbash -x script.sh`), users can dynamically activate or deactivate debugging modes during script execution via the `set` builtin:

```c
/* Code from src/builtins/builtins.c (Lines 470 - 490) */
    /* set -x enables tracing / set +x disables tracing */
    if (strcmp(argv[i], "-x") == 0) state->xtrace = true;
    else if (strcmp(argv[i], "+x") == 0) state->xtrace = false;

    /* set -e enables errexit / set +e disables errexit */
    else if (strcmp(argv[i], "-e") == 0) state->errexit = true;
    else if (strcmp(argv[i], "+e") == 0) state->errexit = false;
```

This flexibility allows developers to selectively isolate problematic code segments:
```bash
echo "Normal execution..."
set -x  # Begin granular execution tracing
temp_var=$(( a * b + 10 ))
process_data "$temp_var"
set +x  # Disable tracing to avoid terminal clutter
echo "Execution complete!"
```

---

## 🧭 5. Tracking Source Line Context ($LINENO and BASH_SOURCE Stack)

When shell scripts become modular, sourcing external helper scripts via `source helper.sh` or `. config.sh`, the engine must track source provenance and line offsets.  
In [`include/zbash.h`](include/zbash.h), lines 51 through 54, `zbash` defines a call frame stack:

```c
/* File: include/zbash.h (Lines 51 - 54) */
    /* Sourcing & BASH_SOURCE tracking */
    char *source_stack[ZBASH_MAX_SOURCE_DEPTH];
    int source_depth;
```

When `source file.sh` is invoked:
1. The shell pushes the filename onto `source_stack[source_depth++]`.
2. As lines are parsed, the per-file line counter increments, allowing `$LINENO` to reflect the active line of the sourced file.
3. Upon reaching EOF of the sourced file, the shell pops the entry from `source_stack`, seamlessly restoring execution context to the caller.

---

## 🎯 6. Key Takeaways for Systems Developers

1. **Great tools are transparent and introspectable:** Built-in tracing and debugging transform a shell from a black-box executor into a transparent systems workbench.
2. **`STDERR` is the dedicated systems channel:** Routing debugging telemetry to file descriptor 2 ensures that application data flows through `STDOUT` without contamination.
3. **Disciplined state machine design:** Structuring `xtrace`, `noexec`, and `errexit` checks cleanly around the core execution loop provides professional-grade observability with minimal source footprint.

---

## 🏆 7. Epilogue: The 25-Chapter Definitive Masterclass

With the completion of Chapter 25, **"Dissecting zbash Architecture"** stands as a comprehensive, **25-chapter definitive masterclass** in the engineering and implementation of a modern, bare-metal Unix shell.

From 86 KB of pure C and ARM64 assembly to a complete exploration of operating systems concepts—virtual memory, SIMD vector processing, AST design, processes and pipes, signals, terminal raw modes, and differential testing—we hope this work illuminates the craft of systems programming.

May this volume inspire engineers everywhere to explore the depths of bare-metal computing and craft sovereign, enduring software systems!
