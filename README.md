# Dissecting zbash Architecture
## The 25-Chapter Definitive Masterclass in Unix Shell Engineering

> **Author:** Wirot Chookeaw ([`chin6700x`](https://github.com/chin6700x))  
> **Repository:** [`zbash-book`](https://github.com/chin6700x/zbash-book)  
> **Companion Codebase:** [`zbash`](https://github.com/chin6700x/zbash)  
> **License:** [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](LICENSE)  
> **Available Languages:** [English](README.md) | [ภาษาไทย](th.README.md)

---

## 🧭 About This Book

The Unix shell is one of the most enduring, ubiquitous, and intellectually fascinating software abstractions in computing history. For more than four decades—from Stephen Bourne's pioneering V7 Bourne Shell, through Brian Fox and Chet Ramey's GNU Bash, to David Korn's ksh and Paul Falstad's zsh—the shell has served as the bridge between human intention and the operating system kernel.

Yet, despite its daily ubiquity in modern development and cloud engineering, how a shell actually works internally remains an enigma to most programmers:
* How does a command line transform from a raw ASCII stream into kernel system calls?
* How can a lexer process nested quotes, backticks, and escape characters without succumbing to "Quoting Hell"?
* How do pipes (`|`), redirections (`>`, `2>&1`), subshells (`(...)`), and job control work at the raw POSIX `fork()`, `execve()`, `pipe()`, `dup2()`, and `termios` layer?
* And how can a fully-featured shell that passes **1,000 out of 1,000 differential parity tests** against macOS Bash be engineered into an executable weighing only **86 KB with zero external dependencies**?

**"Dissecting zbash Architecture"** is a comprehensive, 25-chapter textbook written from the ground up to answer these questions with unvarnished, bare-metal clarity.

---

## 🏛️ Project & Architectural Highlights

* **Pure C99 & ARM64 Assembly:** Zero third-party dependencies, zero external libraries. Interacts directly with POSIX kernel primitives.
* **128-bit ARM64 NEON SIMD Acceleration:** A vector scanner lexer that processes 16 bytes per instruction cycle, skipping delimiters at hardware speed.
* **Extreme Binary Minimization (86 KB):** 22x smaller than GNU Bash (1.9 MB), achieved via whole-program Link-Time Optimization (`-flto`), dead-code stripping (`-Wl,-dead_strip`), and zero-bloat standard library patterns.
* **Sub-Millisecond Startup (~0.3 ms):** 14x faster cold start than traditional shells, residing comfortably in 620 KB of RAM.
* **100.0% Behavioral Parity:** Validated against macOS `/bin/bash` across a rigorous, 10-module, 1,000-test differential verification harness.

---

## 🗺️ Curriculum Roadmap (The 7 Parts)

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DISSECTING ZBASH ARCHITECTURE                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
    ┌───────────────────────────────┼───────────────────────────────┐
    ▼                               ▼                               ▼
[ Part I: Foundations ]     [ Part II: Lexer/Parser ]   [ Part III: Expansions ]
• Genesis & 40-yr Legacy    • ARM64 NEON SIMD Lexer     • Parameter Expansion
• 86 KB Design Philosophy   • Quoting Hell State Mach.  • Arithmetic Engine
• End-to-End Architecture   • Compact AST Design        • Zero-Overhead Arrays
    │                               │                               │
    └───────────────────────────────┼───────────────────────────────┘
                                    │
    ┌───────────────────────────────┴───────────────────────────────┐
    ▼                                                               ▼
[ Part IV: Execution & Plumbing ]                       [ Part V: World-Class Validation ]
• fork(), execve(), Process Lifecycle                   • Differential Testing
• Pipes, dup2(), Redirections                           • Anatomy of the 1,000-Test Suite
• Loops & Control Flow (while/for/case)                 • Hunting Tough Edge Cases
• Subshells (...) vs Grouping {...}
• Builtin Commands Anatomy
    │                                                               │
    └───────────────────────────────┬───────────────────────────────┘
                                    │
    ┌───────────────────────────────┴───────────────────────────────┐
    ▼                                                               ▼
[ Part VI: Horizons & Mindset ]                 [ Part VII: Advanced Systems Masterclass ]
• WebAssembly & WebOS Porting                   • Zero-Dep VT100 Readline
• The Creator's Sovereign Mindset               • Signals & POSIX Job Control
                                                • Pathname Globbing & Inodes
                                                • Context Tab-Completion & Hash Table
                                                • Binary Minimization: 86 KB
                                                • In-Process Debugger Tracing (-x, -v)
```

---

## 📖 Complete Bilingual Table of Contents

| Part | Chapter | English Title | Thai Title |
| :---: | :---: | :--- | :--- |
| **I** | **Ch 01** | [The Unix Shell Journey: From a 40-Year Legacy to zbash](en/ch01_the_genesis.md) | [การเดินทางของ Unix Shell: จากมรดก 40 ปี สู่ zbash](th/ch01_the_genesis.md) |
| **I** | **Ch 02** | [Design Philosophy: 86 KB, Zero Dependencies, Bare-Metal Silicon](en/ch02_design_philosophy.md) | [ปรัชญาการออกแบบ: 86 KB, Zero Dependencies, และ Bare-Metal](th/ch02_design_philosophy.md) |
| **I** | **Ch 03** | [Architectural Overview: From Keystroke to Process Execution](en/ch03_architecture_overview.md) | [ภาพรวมสถาปัตยกรรม: จากการกดคีย์บอร์ด สู่ Process Execution](th/ch03_architecture_overview.md) |
| **II** | **Ch 04** | [The Lexer Secret: 128-bit ARM64 NEON SIMD Acceleration](en/ch04_simd_vector_lexer.md) | [ความลับของ Lexer: เร่งความเร็วด้วย 128-bit ARM64 NEON SIMD](th/ch04_simd_vector_lexer.md) |
| **II** | **Ch 05** | [Taming the Grammar: Deconstructing Quoting Hell & Tokenization](en/ch05_grammar_quoting_hell.md) | [ปราบพยศไวยากรณ์: ถอดรหัส Quoting Hell และการตัดคำ](th/ch05_grammar_quoting_hell.md) |
| **II** | **Ch 06** | [Constructing a Compact Abstract Syntax Tree (AST)](en/ch06_ast_construction.md) | [สร้าง Abstract Syntax Tree (AST) ขนาดกะทัดรัด](th/ch06_ast_construction.md) |
| **III** | **Ch 07** | [The Art of Parameter Expansion: Slicing, Defaults, and Lengths](en/ch07_parameter_expansion.md) | [ศิลปะแห่ง Parameter Expansion: ตัด สไลซ์ แทนที่ และความยาว](th/ch07_parameter_expansion.md) |
| **III** | **Ch 08** | [Embedded Math: Building the Arithmetic Evaluation Engine](en/ch08_arithmetic_engine.md) | [คณิตศาสตร์ในตัวเชลล์: สร้างตัวประเมินผลนิพจน์ $((...))](th/ch08_arithmetic_engine.md) |
| **III** | **Ch 09** | [Zero-Overhead Indexed Arrays in C](en/ch09_arrays_in_c.md) | [อาเรย์ไร้ Overhead: โครงสร้างข้อมูล Indexed Array ในภาษา C](th/ch09_arrays_in_c.md) |
| **IV** | **Ch 10** | [OS Deep Dive: Mastering fork(), execve(), and Process Lifecycles](en/ch10_fork_exec_internals.md) | [ดำดิ่งสู่ OS: ความลับของ fork(), execve() และ Process Lifecycle](th/ch10_fork_exec_internals.md) |
| **IV** | **Ch 11** | [Pipes and Redirections: pipe(), dup2(), and Trailing Loops](en/ch11_pipes_and_redirections.md) | [ท่อและทางเบี่ยง: pipe(), dup2() และปริศนา Trailing Loop](th/ch11_pipes_and_redirections.md) |
| **IV** | **Ch 12** | [Control Flow and Loops: while, for, until, if-then-else, case](en/ch12_control_flow_loops.md) | [การควบคุมทิศทางและลูป: while, for, until, if-then-else, case](th/ch12_control_flow_loops.md) |
| **IV** | **Ch 13** | [Scope Isolation: Subshells (...) vs Grouping {...}](en/ch13_subshells_and_scope.md) | [การแยกแยะ Scope: Subshell (...) vs Grouping {...}](th/ch13_subshells_and_scope.md) |
| **IV** | **Ch 14** | [Behind Builtin Commands: Why cd, export, and read Must Be Internal](en/ch14_builtin_commands.md) | [เบื้องหลังคำสั่ง Builtins: ทำไม cd, export, read ต้องเขียนในตัว](th/ch14_builtin_commands.md) |
| **V** | **Ch 15** | ["Fast with Wrong Output Is Worthless": Differential Testing](en/ch15_differential_testing.md) | ["เร็วแต่ผลลัพธ์ผิด ย่อมไร้ค่า": ศาสตร์แห่ง Differential Testing](th/ch15_differential_testing.md) |
| **V** | **Ch 16** | [Anatomy of the 1,000-Test Suite: 10 Modules vs macOS Bash](en/ch16_anatomy_of_1000_tests.md) | [ผ่ากายวิภาคชุดทดสอบ 1,000 ข้อ: 10 โมดูลประชัน macOS Bash](th/ch16_anatomy_of_1000_tests.md) |
| **V** | **Ch 17** | [Hunting the Toughest Edge Cases: Memory Leaks and Hangs](en/ch17_hunting_edge_cases.md) | [ล่าบั๊กขอบเหว: เมมโมรี่รั่ว ลูปค้าง และการจัดการ Trap สัญญาณ](th/ch17_hunting_edge_cases.md) |
| **VI** | **Ch 18** | [From Terminal to Browser: Porting zbash to WebAssembly & WebOS](en/ch18_wasm_webos_porting.md) | [จาก Terminal สู่เว็บเบราว์เซอร์: การพอร์ต zbash สู่ WebAssembly](th/ch18_wasm_webos_porting.md) |
| **VI** | **Ch 19** | [The Creator's Mindset: Crafting Sovereign Systems Software](en/ch19_the_creators_mindset.md) | [จิตวิญญาณแห่งผู้สร้าง: จากจุดเริ่มต้นสู่ผลงานระดับโลก](th/ch19_the_creators_mindset.md) |
| **VII** | **Ch 20** | [Zero-Dependency Readline: VT100 Line Editing from Scratch](en/ch20_zero_dep_readline.md) | [สร้าง Readline ไร้ Dependency: ระบบ Line Editor และ VT100](th/ch20_zero_dep_readline.md) |
| **VII** | **Ch 21** | [Kernel Signal Management: Job Control and Signals](en/ch21_signals_and_job_control.md) | [การควบคุมสัญญาณระดับเคอร์เนล: Job Control และ Signals](th/ch21_signals_and_job_control.md) |
| **VII** | **Ch 22** | [High-Velocity Vector Scanner: Pathname Expansion & Wildcards](en/ch22_pathname_globbing.md) | [เวกเตอร์สแกนเนอร์ความเร็วสูง: การขยาย Pathname & Wildcards](th/ch22_pathname_globbing.md) |
| **VII** | **Ch 23** | [Interactive Autocompletion & Command Lookup: Tab & $O(1)$ Hash](en/ch23_tab_completion_path_hashing.md) | [ระบบ Autocompletion & Command Lookup: Tab และ Fast $PATH Hash](th/ch23_tab_completion_path_hashing.md) |
| **VII** | **Ch 24** | [Extreme Binary Minimization: From C Source to 86 KB Executable](en/ch24_binary_minimization_86kb.md) | [ศิลปะการรีดไบนารีระดับเอ็กซ์ตรีม: จาก C Source สู่ 86 KB Executable](th/ch24_binary_minimization_86kb.md) |
| **VII** | **Ch 25** | [Script Debugging & Execution Tracing: Building -x, -v, -n, $LINENO](en/ch25_script_debugger_tracing.md) | [เครื่องยนต์ตรวจสอบและดีบั๊กสคริปต์: สร้าง Execution Tracing (-x, -v, -n)](th/ch25_script_debugger_tracing.md) |

---

## 🎯 Intended Audience & Prerequisites

This book is crafted for:
* **Systems Software Engineers:** Developers working in C, C++, Rust, Zig, or Go who desire a master-level understanding of POSIX execution internals.
* **Computer Science Students & Educators:** An empirical, production-tested reference for operating systems courses, compiler engineering, and runtime architectures.
* **DevOps, SRE, and Infrastructure Engineers:** Practitioners who write and maintain mission-critical shell pipelines and want to know precisely what happens under the hood when a script executes.

**Prerequisites:** Familiarity with basic C syntax (pointers, structs, memory management) and standard command-line usage.

---

## 🔗 Companion Codebase

The code discussed throughout this textbook is open source and actively developed in the companion repository:  
👉 **[https://github.com/chin6700x/zbash](https://github.com/chin6700x/zbash)**

---

## 📄 License & Attribution

Copyright © 2026 **Wirot Chookeaw ([`chin6700x`](https://github.com/chin6700x))**.  
All rights reserved.

The literary content, explanations, diagrams, and structural designs within this book are licensed under the **[Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International Public License (CC BY-NC-ND 4.0)](LICENSE)**.

* **You are free to:** Share, copy, and redistribute the material in any medium or format for NonCommercial purposes.
* **Under the following terms:**
  * **Attribution:** You must give appropriate credit to Wirot Chookeaw (`chin6700x`), provide a link to the license, and indicate if changes were made.
  * **NonCommercial:** You may not use the material for commercial purposes without prior written consent from the author.
  * **NoDerivatives:** If you remix, transform, or build upon the material, you may not distribute the modified material.
