# Chapter 12: Control Flow and Loops: while, for, until, if-then-else, case
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch12_control_flow_loops.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"Simplicity is prerequisite for reliability."**  
> — **Edsger W. Dijkstra**  
> 
> **"Our creative thinking must be backed by rigorous testing and disciplined execution."**  
> — **Bjarne Stroustrup**

---

## 🧭 1. The Core of Programmable Shells: Control Flow

Without structured conditionals and loops, a shell is reduced to a primitive sequential batch runner executing commands from top to bottom.  
What elevates Bash into a versatile systems automation language across four decades is its expressive control flow grammar:
* **`if ... then ... elif ... else ... fi`:** Branch selection driven by exit status ($?$).
* **`while ... do ... done`:** Iteration sustained while a condition succeeds ($? == 0$).
* **`until ... do ... done`:** Iteration sustained while a condition fails ($? \neq 0$).
* **`for var in list; do ... done`:** Iteration over word lists.
* **`for (( i=0; i<n; i++ )); do ... done`:** C-style three-clause arithmetic loop.
* **`break [N]` and `continue [N]`:** Multi-tier loop interruption.

In this chapter, we explore how `zbash` implements these control flow primitives in clean, bare-metal C without heavy external parser-generator dependencies like Bison or Yacc.

---

## 🏗️ 2. Conditional Branch Architecture: `exec_if`

In shell scripts, `if` statements support arbitrary nesting:

```bash
if [ -f config.env ]; then
    if [ "$ENV" = "production" ]; then
        echo "Prod mode"
    fi
else
    echo "Default config"
fi
```

Naive pattern matching fails immediately on nested `fi` terminators. In [`src/core/engine.c`](src/core/engine.c#L1224-L1378), `zbash` tracks nesting boundaries via a dedicated **depth counter (`if_d`)**:

```c
/* File: src/core/engine.c (lines 1265 - 1285) */
    if (at_word_start) {
        if (is_word_match(p, "if", 2)) {
            if_d++; /* Inner conditional detected; increment depth */
        } else if (is_word_match(p, "fi", 2)) {
            if (if_d > 0) {
                if_d--; /* Inner conditional closed; decrement depth */
            } else {
                /* Outermost delimiter encountered; terminate scanning */
                p += 2;
                break;
            }
        }
```

### Branch Collection and Recursive Evaluation:
Conditionals are collected into an array of branch descriptors (`cond`, `body`, `is_else`). The engine evaluates clauses sequentially:

```c
/* File: src/core/engine.c (lines 1354 - 1365) */
    int cond_rc = zbash_run_line(state, branches[b].cond);
    if (cond_rc == 0) {
        executed = true;
        if (branches[b].body && branches[b].body[0]) {
            zbash_run_line(state, branches[b].body);
        }
        break; /* Condition succeeded; bypass remaining elif/else branches */
    }
```

By delegating to `zbash_run_line(state, cond)` recursively, conditions can be arbitrary shell constructs: `test -f file`, `[ $a -eq $b ]`, `grep -q pattern text`, or user-defined shell functions.

---

## 🔄 3. Iteration Engines: `exec_while` and `exec_until`

The evaluation models for `while` and `until` share structural symmetry, differing solely in the condition termination polarity:
* `while`: Terminates when `cond_rc != 0`.
* `until`: Terminates when `cond_rc == 0`.

Consider [`src/core/engine.c`](src/core/engine.c#L1380-L1475):

```c
/* File: src/core/engine.c (lines 1380 - 1475) */
static int exec_while(zbash_state_t *state, const char *cmd) {
    bool is_until = (strncmp(cmd, "until", 5) == 0);
    /* Extract cond_buf and body_buf tracking inner_loop_d */
    ...
    while (1) {
        int cond_rc = zbash_run_line(state, cond_buf);
        if (is_until ? (cond_rc == 0) : (cond_rc != 0)) {
            state->last_status = 0;
            break; /* Terminating condition satisfied */
        }

        if (body_buf[0]) {
            zbash_run_line(state, body_buf);
        }
```

---

## ⚡ 4. Multi-Level Interruption: `break [N]` and `continue [N]`

Managing nested loop breaks is a frequent source of bugs in custom shells. When three loops are nested:

```bash
for i in 1 2 3; do
    for j in a b c; do
        if [ "$i" -eq 2 ] && [ "$j" = "b" ]; then
            break 2   # Break out of both inner and outer loops simultaneously!
        fi
        echo "$i-$j"
    done
done
```
The command `break 2` must terminate the current loop frame and propagate the break signal to parent loop frames.

### State-Machine Implementation in `zbash`:
In the primary shell state [`include/zbash.h`](include/zbash.h#L44-L47):

```c
/* File: include/zbash.h (lines 44 - 47) */
    bool loop_break;
    int loop_break_count;
    bool loop_continue;
    int loop_continue_count;
```

When `break 2` executes in [`src/builtins/builtins.c`](src/builtins/builtins.c#L628-L632):

```c
/* File: src/builtins/builtins.c */
    if (strcmp(cmd, "break") == 0) {
        state->loop_break = true;
        state->loop_break_count = (argc > 1) ? atoi(argv[1]) : 1;
        return 0;
    }
```

Within the loop dispatchers in [`src/core/engine.c`](src/core/engine.c#L1488-L1500):

```c
/* File: src/core/engine.c */
    if (state->loop_break) {
        state->loop_break_count--; /* Consume one level of break */
        if (state->loop_break_count <= 0) {
            state->loop_break = false;
            state->loop_break_count = 0;
        }
        break; /* Exit active loop frame immediately */
    }
```

If `loop_break_count` remains greater than 0, `state->loop_break` stays active, prompting enclosing parent loop frames to terminate sequentially until all requested levels are consumed.

---

## 🧮 5. C-Style Arithmetic Loops: `exec_for_c`

Complex numerical calculations in Bash frequently utilize C-style loop syntax:

```bash
for (( i = 0; i < 1000; i++ )); do
    sum=$(( sum + i ))
done
```

Examining [`src/core/engine.c`](src/core/engine.c#L1516-L1628):

```c
/* File: src/core/engine.c (lines 1516 - 1628) */
static int exec_for_c(zbash_state_t *state, const char *cmd) {
    /* 1. Extract clauses: init_expr ; cond_expr ; step_expr */
    ...
    /* 2. Execute initialization */
    if (init_expr[0]) zbash_eval_arithmetic(state, init_expr);

    while (1) {
        /* 3. Evaluate loop guard condition */
        if (cond_expr[0]) {
            long long cond_val = zbash_eval_arithmetic(state, cond_expr);
            if (cond_val == 0) break; /* 0 indicates false; exit loop */
        }

        /* 4. Execute loop body */
        if (body_buf[0]) zbash_run_line(state, body_buf);

        /* Handle break / continue flags */
        if (state->loop_break) { ... break; }

        /* 5. Evaluate post-step expression (e.g., i++) */
        if (step_expr[0]) zbash_eval_arithmetic(state, step_expr);
    }
    return state->last_status;
}
```

Reusing the **Arithmetic Engine** (Chapter 8) grants the loop engine native support for post-increments (`i++`), assignment operators (`i += 2`), and complex arithmetic bounds with zero added syntax complexity.

---

## 🧪 6. Differential Test Verification

In `test/1000-test/test_04_control_flow_loops.sh`, 100 comprehensive validation tests verify control flow semantics:

| Group | Structure | Target Behavior | Result vs macOS Bash |
| :--- | :--- | :--- | :--- |
| **If-Elif-Else** | 4-level nesting | Branch selection order | **PASS (100%)** |
| **While Loop** | 1 to 1,000 count | Loop termination precision | **PASS (100%)** |
| **Until Loop** | Inverted conditions | Guard condition evaluation | **PASS (100%)** |
| **For In Words** | Word splitting with spaces | List tokenization | **PASS (100%)** |
| **C-Style For** | `for (( i=0; i<10; i+=2 ))` | Arithmetic expressions | **PASS (100%)** |
| **Break & Continue** | Multi-level `break 2` | Unwinding nested loops | **PASS (100%)** |

**Suite Result:** `100 / 100 PASS (100.0%)` exact match with macOS Bash.

---

## 🏁 Summary

1. **Embrace Recursive Execution:** Re-invoking `zbash_run_line()` within control structures provides powerful nesting while keeping the engine compact.
2. **Track Nesting Depths:** Maintaining structural counters (`if_d`, `inner_loop_d`) prevents premature boundary termination.
3. **Multi-Tier Interruption State:** Propagating `loop_break_count` across the runtime state enables exact POSIX multi-level break and continue compliance.

In **[Chapter 13: Scope Isolation: Subshells (...) vs Grouping {...}](ch13_subshells_and_scope.md)**, we examine process isolation boundaries, variable mutation leaks, and the exact distinctions between subshells and grouping.
