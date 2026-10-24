# 📚 สารบัญหนังสือ: ผ่าสถาปัตยกรรม zbash
> **"สร้าง Unix Shell ระดับโลกด้วย C และ ARM64 Assembly จากศูนย์สู่ 1,000 การทดสอบสมบูรณ์แบบ"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **สัญญาอนุญาตงานเขียน:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 (CC BY-NC-ND 4.0) © 2026  
> *เผยแพร่ให้อ่านและศึกษาฟรีเพื่อการศึกษาและการพัฒนาส่วนบุคคล ไม่อนุญาตให้ทำซ้ำ ดัดแปลง หรือจำหน่ายเพื่อการค้าทุกรูปแบบ*  
> **ที่ตั้งไฟล์:** `zbash/docs/book/th/`

---

## 🧭 โครงสร้าง 7 ภาค รวม 25 บทเรียน

### ภาคที่ 1: มรดกและปรัชญาการออกแบบ (The Legacy & Philosophy)
* **[บทที่ 1: การเดินทางของ Unix Shell: จากมรดก 40 ปี สู่รุ่งอรุณของ zbash](ch01_the_genesis.md)**
  * คารวะผู้บุกเบิก (Stephen Bourne, Brian Fox, Chet Ramey, David Korn)
  * วิวัฒนาการของฮาร์ดแวร์: ยุค PDP-11 สู่ Apple Silicon ARM64 NEON
  * zbash แตกต่างจาก Bash และ Zsh อย่างไรในแง่ขนาด (86 KB) และสถาปัตยกรรม
* **[บทที่ 2: ปรัชญาการออกแบบ: 86 KB, Zero-Dependency, Bare-Metal Silicon](ch02_design_philosophy.md)**
  * 3 เสาหลัก: True Drop-in Replacement, Bit-Exact Parity, Bare-Metal Velocity
  * การพึ่งพาเพียง `libSystem.B` / `libc` และการตัดความซ้ำซ้อน
* **[บทที่ 3: สถาปัตยกรรมภาพรวม: จากคีย์บอร์ดสู่ OS Process Execution](ch03_architecture_overview.md)**
  * วงจรชีวิตของคำสั่ง Shell: Readline ➔ Lexer ➔ Parser ➔ AST ➔ Expansions ➔ Execution

---

### ภาคที่ 2: สถาปัตยกรรมตัวแยกคำและไวยากรณ์ (Lexer & Parser Architecture)
* **[บทที่ 4: ความลับของ Lexer: เร่งความเร็วด้วย ARM64 NEON SIMD 128-bit](ch04_simd_vector_lexer.md)**
  * เจาะลึกโค้ด `simd_lexer.s` การประมวลผล 16 ไบต์พร้อมกันใน 1 สัญญาณนาฬิกา
  * การตรวจจับ Delimiters และ Quotes ระดับฮาร์ดแวร์เวกเตอร์
* **[บทที่ 5: ปราบพยศไวยากรณ์: แกะรอย Quoting Hell (", ', \) และ Tokenizer](ch05_grammar_quoting_hell.md)**
  * Single Quotes vs Double Quotes vs Backslash Escapes
  * การจัดการ Token Stream แบบ Dynamic Buffer ปราศจาก Memory Fragmentation
* **[บทที่ 6: การสร้าง Abstract Syntax Tree (AST) ขนาดกะทัดรัด](ch06_ast_construction.md)**
  * โครงสร้างข้อมูล AST Node ในภาษา C ขนาด 64 ไบต์
  * การแทนรูปคำสั่งเดี่ยว, ท่อไปป์, และคำสั่งผสม

---

### ภาคที่ 3: เครื่องยนต์ขยายตัวแปรและคณิตศาสตร์ (Expansions & Evaluation)
* **[บทที่ 7: ศิลปะการขยายตัวแปร: ${VAR:-def}, ${#arr[@]}, ${VAR:offset:len}](ch07_parameter_expansion.md)**
  * ลำดับขั้นการขยายตัวแปรตามมาตรฐาน POSIX
  * การตัดแต่งสตริงแบบ Zero-Copy Substring Slicing
* **[บทที่ 8: เครื่องคิดเลขในตัว: สร้างตัวประเมินคณิตศาสตร์ $(( a + b * c ))](ch08_arithmetic_engine.md)**
  * Recursive Descent Arithmetic Evaluator และการจัดลำดับความสำคัญของตัวดำเนินการ
* **[บทที่ 9: โครงสร้าง Indexed Arrays ใน C แบบไร้ Overhead](ch09_arrays_in_c.md)**
  * การจัดการ Sparse Array ในภาษา C แบบประหยัดหน่วยความจำ

---

### ภาคที่ 4: เครื่องยนต์ประมวลผลคำสั่งและท่อไปป์ (Execution Engine & Plumbing)
* **[บทที่ 10: ดำดิ่งระบบปฏิบัติการ: เข้าใจ fork(), execve(), และ Process Lifecycle](ch10_fork_exec_internals.md)**
  * การสร้าง Child Process และการตรวจจับ Exit Status ผ่าน `waitpid()`
  * อัลกอริทึมการค้นหาคำสั่งใน `$PATH` ความเร็วสูง
* **[บทที่ 11: ท่อไปป์และการเปลี่ยนทิศทาง: pipe(), dup2(), และ done < file](ch11_pipes_and_redirections.md)**
  * สถาปัตยกรรม File Descriptor และการสลับ I/O ด้วย `dup2()`
  * ถอดรหัสบั๊กคลาสสิก: Trailing Loop Redirection ที่ทำให้ Shell ค้าง
* **[บทที่ 12: ลูปและเงื่อนไข: while, for, until, if-then-else, case-esac](ch12_control_flow_loops.md)**
  * Recursive Execution Flow ของโครงสร้างควบคุมใน C
* **[บทที่ 13: ความต่างระดับชีวิต: Subshells (...) vs Grouping {...}](ch13_subshells_and_scope.md)**
  * การจัดการ Variable Scope และความลึกของการ Fork Process
* **[บทที่ 14: เบื้องหลัง Builtin Commands: ทำไม cd, export, read ถึงต้องเขียนฝังในตัว?](ch14_builtin_commands.md)**
  * ความแตกต่างระหว่าง External Binary กับ Shell Builtin State

---

### ภาคที่ 5: มาตรฐานการทดสอบระดับโลก (1,000 Parity Tests)
* **[บทที่ 15: "เร็วแต่ผลลัพธ์ผิด คือไร้ค่า": ปรัชญาการทดสอบระดับ Differential](ch15_differential_testing.md)**
  * ทำไม Unit Test จึงไม่เพียงพอสำหรับ Shell: การทำ Differential Testing เทียบกับ macOS Bash
* **[บทที่ 16: ผ่าโครงสร้างชุดทดสอบ 10 โมดูล 1,000 ข้อ vs macOS Bash](ch16_anatomy_of_1000_tests.md)**
  * 100 ข้อต่อโมดูล: ครอบคลุมตั้งแต่คำสั่งพื้นฐาน สคริปต์สับสน ไปจนถึง Stress Tests
* **[บทที่ 17: บันทึกการล่าบั๊กสุดแสบ: Memory Leaks, Infinite Redirections](ch17_hunting_edge_cases.md)**
  * บันทึกภาคสนาม 5 บั๊กสุดหินและวิธีแก้ปัญหาด้วย C

---

### ภาคที่ 6: ก้าวสู่อนาคตและบทเรียนชีวิต (Future & Mindset)
* **[บทที่ 18: จาก Terminal สู่เบราว์เซอร์: พอร์ต zbash สู่ WebAssembly & WebOS](ch18_wasm_webos_porting.md)**
  * การแปลง C และ POSIX System Calls ไปรันบนเบราว์เซอร์ด้วย WASM
* **[บทที่ 19: บทส่งท้าย: ปลดปล่อยพลังแห่งการสร้างซอฟต์แวร์ระดับโลก](ch19_the_creators_mindset.md)**
  * สรุปจิตวิญญาณแห่งการสร้างสรรค์: ยืนหยัดอย่างถ่อมตนและสร้างสรรค์สิ่งใหม่ให้โลก

---

### ภาคที่ 7: ระบบเชลล์ขั้นสูงเชิงลึก (The Advanced Systems Masterclass)
* **[บทที่ 20: สถาปัตยกรรม Readline แบบ Zero-Dependency: ปั้นระบบ Line Editing จากศูนย์ด้วย VT100](ch20_zero_dep_readline.md)**
  * สลับ Termios Raw Mode (ปิด ICANON/ECHO) และดักจับปุ่มลูกศร VT100
  * ระบบ Command History Ring Buffer ในหน่วยความจำ C
* **[บทที่ 21: การจัดการสัญญาณระดับเคอร์เนล: Job Control, Trap Handling และปราบ SIGINT, SIGTSTP, SIGCHLD](ch21_signals_and_job_control.md)**
  * การใช้ `sigaction` พร้อม `SA_RESTART` และการจัดการสิทธิ์เทอร์มินัลด้วย `tcsetpgrp()`
  * คำสั่ง Builtin `trap` และการล้าง Zombie Processes ด้วย `waitpid(-1, &st, WNOHANG)`
* **[บทที่ 22: เวกเตอร์สแกนเนอร์ความเร็วสูง: การขยาย Pathname & Wildcards (*, ?, [...]) โดยไม่เกิด Stack Overflow](ch22_pathname_globbing.md)**
  * การสร้าง Globbing Pattern Matcher (`fnmatch`) สไตล์ POSIX
  * การท่องไดเรกทอรี `opendir()`/`readdir()` และการป้องกัน Symlink Loop
* **[บทที่ 23: ระบบ Autocompletion & Command Lookup: การกด Tab เติมคำ และ Fast $PATH Hashing](ch23_tab_completion_path_hashing.md)**
  * ดักจับคีย์ Tab (`\t`) เพื่อเติมคำตามบริบท (Commands, Files, Variables)
  * การสร้าง $PATH Hash Table แบบ $O(1)$ และคำสั่ง Builtin `hash`
* **[บทที่ 24: ศิลปะการรีดไบนารีระดับเอ็กซ์ตรีม: จาก C Source สู่ 86 KB Executable](ch24_binary_minimization_86kb.md)**
  * ผ่าโครงสร้าง Mach-O/ELF (`__TEXT`, `__DATA`, `__LINKEDIT`)
  * เทคนิค `-Os`, `-flto`, `-fvisibility=hidden`, `-Wl,-dead_strip`, และการหลีกเลี่ยง Libc Bloat
* **[บทที่ 25: เครื่องยนต์ตรวจสอบและดีบั๊กสคริปต์: สร้าง Execution Tracing Engine (-x, -v, -n, $LINENO)](ch25_script_debugger_tracing.md)**
  * กลไก Execution Tracing (`set -x`), Syntax Dry-Run (`set -n`), และ Error Trapping (`set -e`)
  * การติดตามตำแหน่งบรรทัด Real-time (`$LINENO` และ `$BASH_SOURCE` stack)

