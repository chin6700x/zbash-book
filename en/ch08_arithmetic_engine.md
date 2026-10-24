# Chapter 8: Embedded Math: Building the Arithmetic Evaluation Engine
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch08_arithmetic_engine.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"First, solve the problem. Then, write the code."**  
> — **John Johnson**  
> 
> **"Science is what we understand well enough to explain to a computer. Art is everything else we do."**  
> — **Donald Knuth**

---

## 🧭 1. Why a Shell Requires an In-Process Arithmetic Evaluator

In early Unix environments, performing integer arithmetic required delegating to external utilities such as `expr` or `bc`:

```bash
# Historical 1970s paradigm: spawns a child process for every arithmetic operation!
result=$(expr $a + $b)
```

Invoking `expr` obligates the operating system kernel to `fork()` and `execve()` an external binary, introducing several milliseconds of scheduling and memory mapping overhead per invocation. In a loop executing 10,000 iterations, relying on external processes renders scripts impractically sluggish.

To address this bottleneck, POSIX and modern shells mandate **In-Process Arithmetic Expansion `$(( expression ))`**.  
In this chapter, we explore how `zbash` implements a **Recursive Descent Arithmetic Engine** in C supporting:
* Standard operator precedence (multiplication/division evaluated before addition/subtraction).
* Diverse numerical bases: Hexadecimal (`0xFF`), Octal (`077`), and Decimal.
* Bitwise operators (`<<`, `>>`, `&`, `^`, `|`).
* Prefix and postfix mutating increments and decrements (`++x`, `x++`, `--x`, `x--`).
* Exponentiation (`**`) and ternary conditionals (`condition ? true_val : false_val`).

This functionality is consolidated cleanly in [`src/core/engine.c`](src/core/engine.c#L197-L390).

---

## 🏗️ 2. Operator Precedence Cascade

The classic technique for implementing expression evaluation without external parser-generator dependencies is a precedence cascade—layering evaluation functions from lowest precedence to highest:

```
eval_assign      (=, +=, -=, *=, /=)        [Lowest Precedence]
    └── eval_ternary     (?:)
          └── eval_log_or      (||)
                └── eval_log_and     (&&)
                      └── eval_bit_or      (|)
                            └── eval_bit_xor     (^)
                                  └── eval_bit_and     (&)
                                        └── eval_equality    (==, !=)
                                              └── eval_relational (<, <=, >, >=)
                                                    └── eval_shift      (<<, >>)
                                                          └── eval_add_sub    (+, -)
                                                                └── eval_mul_div    (*, /, %)
                                                                      └── eval_power      (**)
                                                                            └── eval_primary    (Numbers, Identifiers, Parens) [Highest]
```

---

## 🔍 3. Dissecting `eval_primary`: Identifiers, Radixes, and Mutation

Consider the terminal evaluator in [`src/core/engine.c`](src/core/engine.c#L206-L255):

```c
/* File: src/core/engine.c (lines 206 - 255) */
static long long eval_primary(zbash_state_t *state, const char **p) {
    skip_ws(p);
    if (**p == '\0') return 0;

    /* 1. Parenthesized Expressions ( expr ) */
    if (**p == '(') {
        (*p)++;
        long long val = eval_expr(state, p);
        skip_ws(p);
        if (**p == ')') (*p)++;
        return val;
    }

    /* 2. Prefix Operators: ++var or --var */
    if (**p == '+' && *(*p + 1) == '+') {
        *p += 2;
        skip_ws(p);
        const char *vstart = *p;
        while (isalnum((unsigned char)**p) || **p == '_') (*p)++;
        char vname[128];
        size_t vlen = *p - vstart;
        if (vlen >= sizeof(vname)) vlen = sizeof(vname) - 1;
        strncpy(vname, vstart, vlen); vname[vlen] = '\0';

        const char *ev = getenv(vname);
        long long val = (ev && *ev) ? strtoll(ev, NULL, 0) : 0;
        val++; // Increment value prior to returning
        char nbuf[32]; snprintf(nbuf, sizeof(nbuf), "%lld", val);
        setenv(vname, nbuf, 1); // Persist updated value to environment
        return val;
    }

    /* 3. Unary Operators: +, -, ~, ! */
    if (**p == '+') { (*p)++; return eval_primary(state, p); }
    if (**p == '-') { (*p)++; return -eval_primary(state, p); }
    if (**p == '~') { (*p)++; return ~eval_primary(state, p); }
    if (**p == '!') { (*p)++; return !eval_primary(state, p); }

    /* 4. Integer Literals: Hexadecimal (0x), Octal (0), and Decimal */
    if (isdigit((unsigned char)**p)) {
        char *end;
        long long val = strtoll(*p, &end, 0); // Base 0 auto-detects 0x and leading 0 prefixes!
        *p = end;
        return val;
    }
```

### Key Engineering Details:
1. **Auto-Radix Parsing via `strtoll(*p, &end, 0)`:**  
   Supplying `0` as the radix parameter enables automatic prefix detection: `0x` is interpreted as hexadecimal, a leading `0` as octal, and all others as standard decimal, eliminating custom string conversion routines.
2. **Environment Mutation on `++var`:**  
   In shell scripts, expressions like `$(( ++x ))` carry mutating side effects. `zbash` calculates `val++`, formats the result, and commits it back into the process environment via `setenv(vname, nbuf, 1)`.

---

## ✖️ 4. Multiplicative Evaluation and Division-by-Zero Trapping

Consider lines 315 to 345 in [`src/core/engine.c`](src/core/engine.c):

```c
/* File: src/core/engine.c (lines 315 - 345) */
/* Multiplicative *, /, % */
static long long eval_mul_div(zbash_state_t *state, const char **p) {
    long long left = eval_power(state, p);
    while (1) {
        skip_ws(p);
        if (**p == '*' && *(*p + 1) != '*' && *(*p + 1) != '=') {
            (*p)++;
            long long right = eval_power(state, p);
            left *= right;
        } else if (**p == '/' && *(*p + 1) != '=') {
            (*p)++;
            long long right = eval_power(state, p);
            if (right == 0) {
                // Intercept division-by-zero to prevent fatal SIGFPE signals!
                fprintf(stderr, "zbash: division by 0\n");
                left = 0;
            } else {
                left /= right;
            }
        } else if (**p == '%' && *(*p + 1) != '=') {
            (*p)++;
            long long right = eval_power(state, p);
            if (right == 0) {
                fprintf(stderr, "zbash: division by 0\n");
                left = 0;
            } else {
                left %= right;
            }
        } else {
            break;
        }
    }
    return left;
}
```

* **Defensive Hardware Exception Guarding:**  
  Unchecked integer division by zero triggers a fatal `SIGFPE` at the processor level, abruptly aborting the shell process. `zbash` checks `if (right == 0)` before executing division or modulo instructions, emitting standard diagnostic messages and continuing cleanly.

---

## 🏁 Summary

1. **In-Process Arithmetic Velocity:** Native arithmetic parsing enables evaluating mathematical expressions in loops at microsecond latency without kernel process overhead.
2. **Recursive Descent Elegance:** Structuring operators in a clean precedence hierarchy yields maintainable, modular parsing code.
3. **Defensive Robustness:** Intercepting division-by-zero and supporting diverse integer representations ensures flawless compliance with differential shell test suites.

In **[Chapter 9: Zero-Overhead Indexed Arrays in C](ch09_arrays_in_c.md)**, we examine how sparse arrays, indexing, and slice assignments are managed without excessive memory allocation.
