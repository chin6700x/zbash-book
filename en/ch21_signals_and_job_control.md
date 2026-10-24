# Chapter 21: Kernel Signal Management: Job Control and Signals
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch21_signals_and_job_control.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"In systems programming, handling success is trivial; mastering asynchronous interrupts is where greatness lies."**  
> — **W. Richard Stevens** (*Advanced Programming in the UNIX Environment*)  
> 
> **"A kernel signal is like lightning; a resilient shell must act as a lightning rod, absorbing energy without system collapse."**  
> — **Wirot Chookeaw**

---

## 🧭 1. Asynchronous Kernel Interrupts

Consider a ubiquitous developer experience:  
You launch a prolonged build script or large network download and realize an argument was mistyped. You immediately press:
```
[ Ctrl + C ]
```
The active foreground task terminates instantly, and the shell prompt returns cleanly, ready for input.

While routine in daily operation, this sequence orchestrates intricate operating system mechanics:
* Why does striking `Ctrl+C` terminate only the active child task while **leaving the parent shell alive and unharmed**, even though both share the identical terminal interface?
* When a background child process terminates, where does the operating system store its exit metrics, and why does neglecting this cleanup produce system-freezing **Zombie Processes (`<defunct>`)**?
* How does the shell command `trap 'rm -f temp.lock; exit' INT TERM` intercept asynchronous kernel interrupts safely in pure C?

In this chapter, we examine **Kernel Signal Handling, Terminal Job Control, and Non-Blocking Process Reaping** within `zbash`.

---

## 🏗️ 2. Core Signals in Shell Architecture

Under Unix, signals represent asynchronous interrupt notifications dispatched from the kernel to user-space processes:

| Signal | Numeric ID | Origin / Keybinding | Default OS Action | Expected Shell Behavior |
| :--- | :---: | :--- | :--- | :--- |
| **`SIGINT`** | 2 | `Ctrl+C` | Terminate process | Forward to foreground process group; parent shell persists |
| **`SIGTSTP`** | 20 | `Ctrl+Z` | Suspend execution | Suspend foreground task and return interactive prompt |
| **`SIGQUIT`** | 3 | `Ctrl+\` | Terminate & dump core | Forward to foreground process; report core dump diagnostics |
| **`SIGCHLD`** | 17 (Darwin) / 20 (Linux) | Child state change / exit | Ignore | Reap zombie process descriptors via non-blocking `waitpid()` |
| **`SIGWINCH`** | 28 | Terminal window resize | Ignore | Recalculate row and column boundaries in line editor |
| **`SIGTERM`** | 15 | `kill <pid>` invocation | Graceful termination | Execute registered `trap` handlers prior to clean exit |

---

## 🛡️ 3. Modern POSIX `sigaction` vs. Legacy `signal()`

Legacy C code often configures signal handlers using `signal(SIGINT, handler)`. In production systems software, this approach is dangerous:
1. In historical Unix, the kernel reset the handler to `SIG_DFL` immediately after execution, leaving processes vulnerable to sudden termination upon back-to-back signals.
2. Blocking system calls (such as `read()` awaiting keyboard input) aborted immediately with `EINTR`, breaking interactive input loops.

The standards-compliant POSIX approach specifies `struct sigaction` with the **`SA_RESTART`** flag:

```c
#include <signal.h>

void zbash_sigint_handler(int sig) {
    (void)sig;
    /* Emit newline when Ctrl+C is struck at an empty prompt */
    write(STDOUT_FILENO, "\n", 1);
}

void zbash_init_signals(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = zbash_sigint_handler;
    sigemptyset(&sa.sa_mask);
    
    /* SA_RESTART: Automatically resume interrupted system calls without returning EINTR */
    sa.sa_flags = SA_RESTART;
    sigaction(SIGINT, &sa, NULL);

    /* Ignore job-suspension signals within the parent interactive shell */
    signal(SIGTSTP, SIG_IGN);
    signal(SIGQUIT, SIG_IGN);
}
```

---

## ⚡ 4. Terminal Job Control and `tcsetpgrp()`

How does `Ctrl+C` kill only child processes without aborting the parent shell?

Under Unix, keyboard interrupt characters generate signals routed exclusively to the **Foreground Process Group** of the controlling terminal:
* The parent shell operates within its own dedicated Process Group ID (PGID).
* Before spawning external binaries (e.g., `grep` or `vim`), the shell segregates child tasks into an independent process group via `setpgid(0, 0)`.
* The parent shell then transfers terminal foreground mastership to the child using **`tcsetpgrp(STDIN_FILENO, child_pgid)`**!

```
                    Terminal Keyboard (STDIN)
                                │
            ┌───────────────────┴───────────────────┐
            ▼                                       ▼
   [ Interactive Idle Shell ]              [ Foreground Task Running ]
    Foreground PGID = zbash                 Foreground PGID = child_pid
    Ctrl+C ➔ Shell prints newline           Ctrl+C ➔ Child process terminated!
```

Upon child termination, the parent shell reclaims foreground control via `tcsetpgrp(STDIN_FILENO, getpgrp())` before rendering the next prompt.

---

## 🪤 5. Implementing the `trap` Builtin

The `trap` builtin registers shell scripts to execute upon signal arrival:

```bash
trap 'echo "Cleaning up..."; rm -f /tmp/lock' INT TERM EXIT
```

Modeling signal traps in C:

```c
typedef struct {
    char *handler_cmd;  /* Script statement to execute on signal arrival */
    bool is_ignored;    /* Flag for explicitly ignored signals (trap '' INT) */
} zbash_trap_t;

static zbash_trap_t g_traps[32];
static volatile sig_atomic_t g_pending_signal = -1;

static void global_signal_dispatcher(int sig) {
    if (sig >= 0 && sig < 32 && g_traps[sig].handler_cmd) {
        /* Defer execution to the primary shell event loop safely */
        g_pending_signal = sig;
    }
}
```

### Async-Signal-Safety Constraints:
Directly invoking `malloc()`, `printf()`, or the statement parser within a kernel signal handler risks deadlocks or memory corruption because these functions are not reentrant.  
The robust architectural pattern sets an atomic flag (`g_pending_signal`) inside the handler, allowing the primary shell loop to evaluate `zbash_run_line(state, g_traps[sig].handler_cmd)` safely at standard execution points.

---

## 🧟 6. Non-Blocking Zombie Reaping via `SIGCHLD` and `WNOHANG`

When a background job executes:
```bash
sleep 10 &
```
The command runs asynchronously. Upon completion 10 seconds later, it enters a **Zombie state (`<defunct>`)**.  
If the parent shell neglects to invoke `waitpid()` to collect its exit metrics, the kernel retains the process entry. Unchecked accumulation exhausts the kernel process table, blocking new program creation.

### Non-Blocking Zombie Reaper Implementation:
```c
void zbash_sigchld_handler(int sig) {
    (void)sig;
    int saved_errno = errno;
    pid_t pid;
    int status;

    /* Harvest all terminated background processes without blocking */
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        /* Record termination metrics in background job table */
        zbash_mark_job_completed(pid, status);
    }

    errno = saved_errno; /* Restore errno to avoid polluting standard routines */
}
```

Using `waitpid(-1, &status, WNOHANG)` purges terminated child tasks in sub-microsecond intervals without introducing latency to active typing.

---

## 🏁 Summary

1. **Rely on `sigaction` Over Legacy `signal()`:** Always specify `SA_RESTART` to maintain system call continuity across signal interruptions.
2. **Master Terminal Ownership:** `tcsetpgrp()` governs foreground process group arbitration, separating robust production shells from unmanaged prototypes.
3. **Reap Zombie Processes Continuously:** Handling `SIGCHLD` with non-blocking `waitpid(..., WNOHANG)` prevents kernel process table exhaustion.

In **[Chapter 22: High-Speed Pathname Globbing: Wildcards Without Stack Overflow](ch22_pathname_globbing.md)**, we examine high-speed filesystem pattern matching: implementing `*`, `?`, and `[...]` expansions with safeguards against circular symlink loops.
