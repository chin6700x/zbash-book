# 📚 Table of Contents: Deconstructing the zbash Architecture
> **"Engineering a World-Class Unix Shell in C and ARM64 Assembly: From Scratch to 1,000 Parity Tests"**  
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Location:** `zbash/docs/book/en/`

---

## 🧭 Structure: 7 Parts, 25 Chapters

### Part I: The Legacy & Design Philosophy
* **[Chapter 1: The Unix Shell Journey: From a 40-Year Legacy to zbash](ch01_the_genesis.md)**
  * Honoring the pioneers: Stephen Bourne, Brian Fox, Chet Ramey, David Korn, Paul Falstad
  * Hardware evolution: From the PDP-11 era to Apple Silicon ARM64 NEON
  * Architectural positioning: 86 KB footprint, zero external dependencies, hardware vectorization
* **[Chapter 2: Design Philosophy: 86 KB, Zero Dependencies, Bare-Metal Silicon](ch02_design_philosophy.md)**
  * The three core pillars: True Drop-in Replacement, Bit-Exact Parity, Bare-Metal Velocity
  * Relying solely on `libSystem.B` / `libc` and eliminating runtime bloat
* **[Chapter 3: Architectural Overview: From Keystroke to Process Execution](ch03_architecture_overview.md)**
  * The shell lifecycle: Readline ➔ Lexer ➔ Parser ➔ AST ➔ Expansions ➔ Execution

---

### Part II: Lexer & Parser Architecture
* **[Chapter 4: The Lexer Secret: 128-bit ARM64 NEON SIMD Acceleration](ch04_simd_vector_lexer.md)**
  * Deep dive into `src/asm/simd_lexer.s`: Scanning 16 bytes per clock cycle
  * Hardware-accelerated delimiter and quote detection
* **[Chapter 5: Taming the Grammar: Deconstructing Quoting Hell and Tokenization](ch05_grammar_quoting_hell.md)**
  * Single quotes vs. double quotes vs. backslash escapes
  * Dynamic token buffer management without heap fragmentation
* **[Chapter 6: Constructing a Compact Abstract Syntax Tree (AST)](ch06_ast_construction.md)**
  * Cache-friendly 64-byte C AST node memory representation
  * Representing simple commands, pipelines, and compound structures

---

### Part III: Expansions & Evaluation Engine
* **[Chapter 7: The Art of Parameter Expansion: Slicing, Defaults, and Lengths](ch07_parameter_expansion.md)**
  * POSIX parameter expansion precedence: `${VAR:-def}`, `${#arr[@]}`, `${VAR:offset:len}`
  * High-performance string manipulation via zero-copy slicing
* **[Chapter 8: Embedded Math: Building the Arithmetic Evaluation Engine](ch08_arithmetic_engine.md)**
  * Recursive descent arithmetic evaluator for `$(( a + b * c ))`
  * Operator precedence, bitwise logic, and 64-bit integer overflows
* **[Chapter 9: Zero-Overhead Indexed Arrays in C](ch09_arrays_in_c.md)**
  * Memory-efficient sparse array implementation in pure C
  * Compound assignment and multi-index traversal

---

### Part IV: Execution Engine & System Plumbing
* **[Chapter 10: OS Deep Dive: Mastering fork(), execve(), and Process Lifecycles](ch10_fork_exec_internals.md)**
  * Child process creation, environment propagation, and `waitpid()` status decoding
  * High-speed `$PATH` binary search algorithm
* **[Chapter 11: Pipes and Redirections: pipe(), dup2(), and Trailing Loops](ch11_pipes_and_redirections.md)**
  * File descriptor table manipulation and atomic redirection via `dup2()`
  * Diagnosing and solving the classic trailing loop redirection deadlock (`done < file`)
* **[Chapter 12: Control Flow and Loops: while, for, until, if-then-else, case](ch12_control_flow_loops.md)**
  * Recursive execution flow of structured control flow in C
  * Pattern matching in `case...esac` blocks
* **[Chapter 13: Scope Isolation: Subshells (...) vs Grouping {...}](ch13_subshells_and_scope.md)**
  * Process boundary isolation vs. in-process grouped evaluation
  * Variable mutation leak prevention across subshell boundaries
* **[Chapter 14: Behind Builtin Commands: Why cd, export, and read Must Be Internal](ch14_builtin_commands.md)**
  * Modifying parent process state vs. child process constraints
  * Complete builtin lookup table and fast dispatching

---

### Part V: World-Class Validation: 1,000 Parity Tests
* **[Chapter 15: "Fast with Wrong Output Is Worthless": Differential Testing](ch15_differential_testing.md)**
  * Why standard unit tests fall short for system shells
  * Bit-for-bit differential test harness comparing zbash with macOS Bash
* **[Chapter 16: Anatomy of the 1,000-Test Suite: 10 Modules vs macOS Bash](ch16_anatomy_of_1000_tests.md)**
  * 100 tests per module: Core syntax, expansions, pipelines, subshells, and stress tests
  * Clean-room regression verification and 100.0% exact match parity
* **[Chapter 17: Hunting the Toughest Edge Cases: Memory Leaks and Hangs](ch17_hunting_edge_cases.md)**
  * Five real-world post-mortems: Deep redirection chains, signal races, descriptor leaks

---

### Part VI: Horizons & Engineering Mindset
* **[Chapter 18: From Terminal to Browser: Porting zbash to WebAssembly & WebOS](ch18_wasm_webos_porting.md)**
  * Compiling C and POSIX system call abstractions to WASM / Emscripten
  * Running an interactive shell terminal natively inside web environments
* **[Chapter 19: The Creator's Mindset: Crafting Sovereign Systems Software](ch19_the_creators_mindset.md)**
  * Engineering philosophy: Respecting heritage while pioneering modern bare-metal systems
  * Principles of sustainable, zero-external-debt software development

---

### Part VII: Advanced Systems Engineering Masterclass
* **[Chapter 20: Zero-Dependency Readline: VT100 Line Editing from Scratch](ch20_zero_dep_readline.md)**
  * Switching termios raw mode (disabling ICANON / ECHO) and decoding VT100 escape sequences
  * Ring buffer command history management with zero memory leaks
* **[Chapter 21: Kernel Signal Management: Job Control and Signals](ch21_signals_and_job_control.md)**
  * Mastering `sigaction`, `SA_RESTART`, and terminal foreground arbitration via `tcsetpgrp()`
  * Internal `trap` engine and reap routines for zombie processes via `waitpid(-1, &st, WNOHANG)`
* **[Chapter 22: High-Speed Pathname Globbing: Wildcards Without Stack Overflow](ch22_pathname_globbing.md)**
  * Building a robust POSIX-compliant wildcard pattern matcher (`*`, `?`, `[...]`)
  * Safe directory traversal (`opendir`/`readdir`) with circular symlink safeguards
* **[Chapter 23: Tab Autocompletion and Fast $PATH Hashing](ch23_tab_completion_path_hashing.md)**
  * Contextual Tab key (`\t`) interception: Commands, filesystem paths, and environment variables
  * $O(1)$ executable lookup table and the internal `hash` builtin
* **[Chapter 24: Binary Minimization: From C Source to 86 KB Executable](ch24_binary_minimization_86kb.md)**
  * Dissecting Mach-O / ELF binary layouts (`__TEXT`, `__DATA`, `__LINKEDIT`)
  * Compiler and linker optimization flags: `-Os`, `-flto`, `-fvisibility=hidden`, `-Wl,-dead_strip`
* **[Chapter 25: Script Debugger & Tracing Engine: -x, -v, -n, and $LINENO](ch25_script_debugger_tracing.md)**
  * Implementing execution tracing (`set -x`), syntax validation (`set -n`), and error exit (`set -e`)
  * Dynamic line tracking with `$LINENO` and `$BASH_SOURCE` call stacks
