# Chapter 10: OS Deep Dive: Mastering fork(), execve(), and Process Lifecycles
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch10_fork_exec_internals.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"The Unix programming environment is based on the idea of small, modular programs that work together cleanly through files and processes."**  
> — **W. Richard Stevens** (*Advanced Programming in the UNIX Environment*)  
> 
> **"Talk is cheap. Show me the code."**  
> — **Linus Torvalds**

---

## 🧭 1. The Unix Process Paradigm: From String to Kernel Task

When you enter a command on your keyboard:
```bash
ls -la /tmp
```
and press Enter, what transpires within fractions of a millisecond is far more than simple string reading. The shell requests the operating system kernel (Darwin / Linux) to materialize a new execution image, supervise its run, and capture its termination metrics.

This interaction represents the core design of Unix established across five decades:
* **`fork()`:** Clones the active parent process into an isolated child process sharing memory mappings and file descriptor tables.
* **`execve()` / `execvp()`:** Discards the cloned process image, loading the target executable binary directly into the child process address space to invoke its `main()` entry point.
* **`waitpid()`:** Halts the parent until the child process completes execution, retrieving its termination status ($?$) and reaping zombie descriptors from the operating system process table.

In this chapter, we trace the implementation in `zbash` to observe how process lifecycles are orchestrated with zero abstraction overhead and 100% POSIX compliance.

---

## 🏗️ 2. Process Lifecycle Architecture

```
        Parent Process (zbash)
                 │
                 ├─── 1. fork() ─────────────────────────────┐
                 │                                           ▼
                 │                                  Child Process (Clone)
                 │                                           │
                 │                                           ├─── 2. apply_redirections()
                 │                                           │    (I/O plumbing via dup2)
                 │                                           │
                 │                                           ├─── 3. execvp("ls", argv)
                 │                                           │    (Image replacement)
                 │                                           ▼
                 │                                  External Binary Running...
                 │                                           │
                 │                                           │ (Exit / Signal)
                 ├─── 4. waitpid(pid, &status) ◄─────────────┘
                 │    (Parent waits for child)
                 ▼
         Extract Exit Status ($?)
       (127 = Not Found, 126 = No Perm,
        128 + N = Signal Termination)
```

---

## 🔍 3. Executing External Commands in `src/core/engine.c`

Consider external process dispatching in [`src/core/engine.c`](src/core/engine.c#L954-L970):

```c
/* File: src/core/engine.c (lines 954 - 970) */
    pid_t pid = fork();
    if (pid == 0) {
        /* Executed within Child Process context */
        if (apply_redirections(cmd) != 0) _exit(1);
        execvp(effective_argv[0], effective_argv);
        
        /* If execution reaches this line, execvp failed! */
        fprintf(stderr, "zbash: %s: %s\n", effective_argv[0], strerror(errno));
        _exit((errno == ENOENT) ? 127 : 126);
    } else if (pid < 0) {
        /* Fork failure (e.g., process table exhaustion) */
        perror("zbash: fork");
        return 1;
    }

    /* Executed within Parent Process context (primary zbash) */
    int status = 0;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status)) return WEXITSTATUS(status);
    if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);
    return 1;
```

### Critical Systems Engineering Insights:

#### 1. Why `_exit()` is Mandatory Over `exit()` in Child Branches
Notice that error branches in the child process invoke `_exit()` rather than standard `exit()`:
* **`exit()`:** Flushes C standard I/O buffers (`stdio`) and triggers cleanup handlers registered via `atexit()`, which can disrupt or flush parent buffers prematurely.
* **`_exit()`:** Invokes the kernel termination system call directly, cleanly terminating the child process without interfering with parent execution state.

#### 2. Strict POSIX Status Codes: 127 vs. 126
```c
_exit((errno == ENOENT) ? 127 : 126);
```
Adhering to strict POSIX shell specifications:
* **Exit Code `127`:** Emitted when `errno == ENOENT` (No such file or directory), indicating the command binary **does not exist anywhere in `$PATH`**.
* **Exit Code `126`:** Emitted when the file is discovered but **lacks executable permissions (`EACCES`)** or is not a recognizable binary format (`ENOEXEC`).

#### 3. Signal Decoding and Status Calculation
```c
if (WIFEXITED(status)) return WEXITSTATUS(status);
if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);
```
* **`WIFEXITED(status)`:** Indicates the process terminated cleanly via `exit(code)`, returning $0–255$ via `WEXITSTATUS`.
* **`WIFSIGNALED(status)`:** Indicates the process was terminated by an unhandled signal (such as `SIGINT` from `Ctrl+C` or `SIGKILL`). POSIX specifies that `$?` must reflect **$128 + \text{Signal Number}$**. For instance, `SIGINT` (signal 2) yields $128 + 2 = 130$.

---

## ⚡ 4. Fast-Path In-Process Execution for Builtin Commands

While modern kernels utilize Copy-on-Write (COW) during `fork()`, context switching and page table updates still incur measurable overhead.

For internal shell operations—such as `cd`, `export`, `set`, `read`, and `echo`—creating child processes is unnecessary:

```c
/* File: src/core/engine.c (lines 920 - 947) */
    /* Evaluate Internal Builtin Commands */
    if (zbash_is_builtin(effective_argv[0])) {
        /* Fast Path: No I/O redirection; execute directly in parent process */
        if (!cmd->in_file && !cmd->out_file && !cmd->err_file && !cmd->err_to_out && !cmd->out_to_err) {
            return zbash_exec_builtin(state, effective_argc, effective_argv);
        }

        /* Redirection Present: e.g., echo "hello" > file.txt */
        int saved_in = dup(STDIN_FILENO);
        int saved_out = dup(STDOUT_FILENO);
        int saved_err = dup(STDERR_FILENO);

        if (apply_redirections(cmd) != 0) {
            close(saved_in);
            close(saved_out);
            close(saved_err);
            return 1;
        }
        int ret = zbash_exec_builtin(state, effective_argc, effective_argv);
        fflush(stdout);
        fflush(stderr);

        /* Restore original terminal file descriptors */
        dup2(saved_in, STDIN_FILENO);
        dup2(saved_out, STDOUT_FILENO);
        dup2(saved_err, STDERR_FILENO);
        close(saved_in);
        close(saved_out);
        close(saved_err);
        return ret;
    }
```

### Architectural Advantages:
1. **Sub-Microsecond Latency:** Operations like loops mutating variables or running builtins execute entirely within CPU L1/L2 cache lines without issuing process-creation system calls.
2. **Safe Descriptor Restoration:** When redirections accompany builtins (`echo "data" > out.log`), `dup()` caches standard descriptors (0, 1, 2). Once the builtin executes and flushes its buffers via `fflush()`, `dup2()` restores the terminal connections cleanly.

---

## 🌊 5. Multi-Stage Pipeline Process Groups

When executing a multi-stage pipeline:
```bash
cat large.txt | grep "error" | wc -l
```
The shell spawns 3 concurrent child processes interconnected via kernel data pipes (`pipe()`):

```c
/* File: src/core/engine.c (lines 1007 - 1089) */
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
            /* Child process i */
            if (i > 0) {
                dup2(pipe_fds[i - 1][0], STDIN_FILENO);   /* Read from preceding stage */
            }
            if (i < n - 1) {
                dup2(pipe_fds[i][1], STDOUT_FILENO);     /* Write to succeeding stage */
            }

            /* Close all inherited descriptors inside child */
            for (int j = 0; j < n - 1; j++) {
                close(pipe_fds[j][0]);
                close(pipe_fds[j][1]);
            }

            if (apply_redirections(cmd) != 0) {
                _exit(1);
            }

            execvp(cmd->argv[0], cmd->argv);
            fprintf(stderr, "zbash: %s: %s\n", cmd->argv[0], strerror(errno));
            _exit((errno == ENOENT) ? 127 : 126);
        } else if (pids[i] < 0) {
            perror("zbash: fork");
            return 1;
        }
    }

    /* Parent process must close all pipeline descriptors immediately! */
    for (int j = 0; j < n - 1; j++) {
        close(pipe_fds[j][0]);
        close(pipe_fds[j][1]);
    }

    /* Reap all child processes to prevent zombie table buildup */
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

### Deadlock Prevention via Parent Descriptor Hygiene:
If the parent process neglects to close its write descriptor (`pipe_fds[j][1]`) after spawning child tasks, downstream reader processes (such as `wc -l`) will never observe an **End-of-File (EOF)** condition because the kernel still considers the parent an active writer. The pipeline will hang permanently. Immediate closure of pipe ends across both child and parent contexts is mandatory for shell stability.

---

## 🧪 6. Differential Test Verification

In `test/1000-test/test_07_pipelines_redirections.sh`, process lifecycle behaviors are tested directly against macOS Bash:

| Scenario | Invocation | Observed Status | Compatibility Result |
| :--- | :--- | :--- | :--- |
| **Command Not Found** | `non_existent_command_xyz` | `127` | **Exact Match (100%)** |
| **Permission Denied** | `/tmp/no_exec_file.sh` (mode 0644) | `126` | **Exact Match (100%)** |
| **Signal Interruption** | `kill -9 $$` or `kill -TERM` | `128 + SIG` | **Exact Match (100%)** |
| **Pipeline Termination** | `false \| true` vs `true \| false` | `0` vs `1` | **Exact Match (100%)** |
| **Process Reaping** | 5,000 sequential forks | Zero zombies | **Exact Match (100%)** |

---

## 🏁 Summary

1. **POSIX Process Contracts:** Emitting exit code 127 on missing binaries, 126 on permission denials, and $128 + \text{Signal}$ on terminations is essential for production compatibility.
2. **Descriptor Hygiene:** Every descriptor created via `pipe()` or `dup()` must be closed in the exact required sequence to avoid deadlocks.
3. **In-Process Optimization:** Bypassing `fork()` for internal builtins preserves CPU cycles and memory bandwidth.

In **[Chapter 11: Pipes and Redirections: pipe(), dup2(), and Trailing Loops](ch11_pipes_and_redirections.md)**, we examine advanced descriptor plumbing and resolve the classic trailing loop redirection hang (`done < file`).
