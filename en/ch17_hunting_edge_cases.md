# Chapter 17: Hunting the Toughest Edge Cases: Memory Leaks and Hangs
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch17_hunting_edge_cases.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"Debugging is twice as hard as writing the code in the first place. Therefore, if you write the code as cleverly as possible, you are, by definition, not smart enough to debug it."**  
> — **Brian W. Kernighan**  
> 
> **"Every bug eradicated is a monument to stability, carved into the granite of our codebase."**  
> — **Wirot Chookeaw**

---

## 🧭 1. Battlefield Chronicles

Operating system-level software is never born immaculate in a single stroke.  
Achieving a 1,000 / 1,000 parity score in `zbash` was not a product of fortune, but the outcome of unrelenting bug hunting across weeks of disciplined analysis.

In shell engineering, a class of subtle bugs remains dormant under casual command-line interactions, materializing only under deep script recursion, compound pipelines, or multi-thousand iteration stress runs.  
In this chapter, we open our engineering field logs to examine **Five Notorious Edge Cases** unmasked during differential parity testing and explain their permanent remedies in pure C.

---

## 🪲 Bug 1: The Forgotten Pipe Hang

### 📌 Symptoms:
When executing compound pipelines:
```bash
cat data.csv | grep "active" | wc -l
```
The pipeline froze indefinitely. The shell failed to return a prompt, and CPU utilization hovered at 0.0%.

### 🔍 Root Cause Analysis:
Inspecting file descriptors via `lsof`, the consumer process `wc -l` was blocked waiting on standard input. Although `grep` had completed its task and exited, why did `wc` never receive the **End-of-File (EOF)** notification?

**The Kernel Discovery:**  
During pipeline construction, the parent process (`zbash`) invoked `pipe()` to create inter-process channels, but **neglected to close the write-end (`pipe_fds[1]`) within its own process context!**  
Because the operating system observed that an active process (the parent shell) still held an open write handle, it maintained the channel. Downstream consumers waited perpetually for more data.

### 🛠️ Remediation in C:
In [`src/core/engine.c`](src/core/engine.c#L1076-L1079):

```c
/* File: src/core/engine.c */
    /* Strict Rule: The parent must close all pipeline pipe ends immediately after forking! */
    for (int j = 0; j < n - 1; j++) {
        close(pipe_fds[j][0]);
        close(pipe_fds[j][1]);
    }
```
Closing inherited write ends in the parent allowed EOF signals to propagate properly, instantly unfreezing all pipeline operations.

---

## 🪲 Bug 2: Keyboard EOF After Trailing Loops (`done < file`)

### 📌 Symptoms:
In Test #77 of `test_07_pipelines_redirections.sh`:
```bash
while read line; do
    echo "Item: $line"
done < file.txt

read user_input # Did not wait for keyboard input; skipped immediately!
```

### 🔍 Root Cause Analysis:
When redirecting standard input to `< file.txt`, the loop handler invoked `dup2(fd, STDIN_FILENO)` on the primary process. Because the loop executed in-process, replacing `STDIN_FILENO` permanently severed the shell's original terminal input stream. Upon consuming the file, the shell remained trapped in EOF.

### 🛠️ Remediation in C:
In [`src/core/engine.c`](src/core/engine.c#L1456-L1510):

```c
    /* Preserve standard terminal descriptor prior to loop entry */
    saved_in = dup(STDIN_FILENO);
    apply_redirections(&redir_cmd);

    /* Execute loop to completion... */

    /* Restore original terminal connection */
    dup2(saved_in, STDIN_FILENO);
    close(saved_in);
```
Preserving and restoring descriptor 0 resolved input stream corruption cleanly.

---

## 🪲 Bug 3: Variable Stomping in Recursive Functions

### 📌 Symptoms:
In recursive shell functions such as factorial calculations:
```bash
factorial() {
    local n=$1
    if [ "$n" -le 1 ]; then
        echo 1
        return
    fi
    local prev=$(factorial $(( n - 1 )))
    echo $(( n * prev ))
}
```
The computed results corrupted because inner activation frames overwrote outer instances of `$n`.

### 🔍 Root Cause Analysis:
Early prototypes managed `local` variables via unversioned `setenv()` calls without recording antecedent values. When an inner recursive frame set `local n=1`, it permanently trampled `n=5` in outer frames.

### 🛠️ Remediation in C:
We implemented an explicit **Call Scope Stack** in [`include/zbash.h`](include/zbash.h#L65-L70):

```c
    struct {
        char *name;
        char *old_val;
        bool was_set;
    } local_vars[512];
    int local_var_count;
```
Every `local` declaration records prior variable state. Upon function exit, the runtime unwinds the scope stack, restoring prior environment values to support clean recursion up to 512 activation frames.

---

## 🪲 Bug 4: Inode Exhaustion from Ephemeral Here-Strings (`<<<`)

### 📌 Symptoms:
During 10,000-iteration stress benchmarks, the operating system raised disk-full alerts despite hundreds of gigabytes of available storage.

### 🔍 Root Cause Analysis:
Querying filesystem health with `df -i` revealed that **available inode allocations were 100% exhausted**.  
Here-Strings (`cmd <<< "text"`) created temporary files under `/tmp/zbash_hs_XXXXXX` to feed input readers, but failed to unlink them upon statement completion.

### 🛠️ Remediation in C:
In [`src/core/engine.c`](src/core/engine.c#L768-L771):

```c
    if (cmd->in_file) {
        if (strncmp(cmd->in_file, "/tmp/zbash_hs_", 14) == 0) {
            unlink(cmd->in_file); /* Unlink disk inode immediately */
        }
        free(cmd->in_file);
    }
```
Unlinking temporary inodes in sub-millisecond timeframes eliminated disk footprint accumulation entirely.

---

## 🪲 Bug 5: Fatal SIGFPE Crash on Division by Zero

### 📌 Symptoms:
When processing an arithmetic expression with zero division:
```bash
echo $(( 10 / 0 ))
```
Rather than emitting a diagnostic message and continuing, the shell crashed violently with `Floating point exception: 8 (SIGFPE)`.

### 🔍 Root Cause Analysis:
On ARM64 and x86 architectures, executing integer division or modulo by zero at the C language level (`a / b`) triggers a fatal processor exception, aborting the process abruptly.

### 🛠️ Remediation in C:
In [`src/core/engine.c`](src/core/engine.c#L326-L335):

```c
    if (op == '/' || op == '%') {
        if (b == 0) {
            fprintf(stderr, "zbash: division by 0\n");
            state->last_status = 1;
            return 0; /* Software intercept before hardware fault! */
        }
        val = (op == '/') ? (val / b) : (val % b);
    }
```
Intercepting divisor boundaries in software shields the runtime from hardware faults, returning exit code 1 with standard error formatting identical to GNU Bash.

---

## 🏁 Summary

1. **Respect Kernel Resource Semantics:** Pipe descriptors and disk inodes must be reclaimed systematically at point of use.
2. **Trap Hardware Faults in Software:** Robust systems guard against processor-level interrupts such as division-by-zero.
3. **Comprehensive Test Suites Guarantee Longevity:** Without the 1,000-test differential suite, these five subtle failure modes would have surfaced in live production environments.

In **Part VI: Horizons & Engineering Mindset ([Chapter 18](ch18_wasm_webos_porting.md))**, we explore compiling `zbash` to the browser: porting native POSIX system calls to **WebAssembly & WebOS**!
