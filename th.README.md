# ผ่าสถาปัตยกรรม zbash
## ตำราวิศวกรรมการสร้าง Unix Shell ฉบับสมบูรณ์ 25 บท (The Definitive 25-Chapter Masterclass)

> **ผู้เขียน:** วิโรจน์ ชูแก้ว ([`chin6700x`](https://github.com/chin6700x))  
> **คลังเอกสาร (Repository):** [`zbash-book`](https://github.com/chin6700x/zbash-book)  
> **คลังโค้ดต้นฉบับ (Companion Codebase):** [`zbash`](https://github.com/chin6700x/zbash)  
> **สัญญาอนุญาต:** [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](LICENSE)  
> **ภาษาที่รองรับ:** [ภาษาไทย](th.README.md) | [English](README.md)

---

## 🧭 เกี่ยวกับหนังสือเล่มนี้

Unix Shell คือหนึ่งในสถาปัตยกรรมซอฟต์แวร์ที่เก่าแก่ ยืนยง และทรงอิทธิพลที่สุดในประวัติศาสตร์คอมพิวเตอร์ ตลอดระยะเวลากว่า 4 ทศวรรษ นับตั้งแต่ Bourne Shell ของ Stephen Bourne, GNU Bash ของ Brian Fox และ Chet Ramey, จนถึง ksh ของ David Korn และ zsh ของ Paul Falstad เชลล์ทำหน้าที่เป็นสะพานเชื่อมระหว่างความคิดของมนุษย์กับแกนกลางระบบปฏิบัติการ (Operating System Kernel) เสมอมา

ทว่า แม้เราจะพิมพ์คำสั่งใน Terminal อยู่ทุกวัน น้อยคนนักที่จะเข้าใจกลไกภายในอย่างถ่องแท้:
* ตัวอักษรบนแป้นพิมพ์ถูกแปลงเป็น System Call ของเคอร์เนลได้อย่างไร?
* Lexer จัดการกับเครื่องหมายคำพูดซ้อนกัน (Quoting Hell) อย่างไรโดยไม่สับสน?
* ท่อไปป์ (`|`), การเบี่ยงเบน I/O (`>`, `2>&1`), Subshell (`(...)`), และ Job Control ทำงานอย่างไรในระดับลึกของ `fork()`, `execve()`, `pipe()`, `dup2()` และ `termios`?
* และเราจะสร้างเชลล์ที่มีฟังก์ชันครบถ้วน ผ่านชุดทดสอบ Parity **1,000 จาก 1,000 ข้อ (100.0%)** เทียบเท่า macOS Bash ให้มีขนาดเพียง **86 KB และไร้ Dependency ภายนอก** ได้อย่างไร?

**"ผ่าสถาปัตยกรรม zbash"** คือตำราวิศวกรรมระบบ 25 บท ที่เขียนขึ้นจากประสบการณ์จริง เพื่อผ่าเปิดกลไกภายในของ Unix Shell ในระดับ Bare-Metal อย่างโปร่งใสและลึกซึ้งที่สุด

---

## 🏛️ ไฮไลท์เชิงวิศวกรรมของ zbash

* **Pure C99 & ARM64 Assembly:** ไร้การพึ่งพาไลบรารีภายนอกแม้แต่ตัวเดียว (Zero-Dependency) สื่อสารกับ POSIX Kernel Primitives โดยตรง
* **128-bit ARM64 NEON SIMD Acceleration:** ตัวตัดคำแบบเวกเตอร์ สแกนข้อความทีละ 16 ไบต์ต่อไซเคิลซีพียู ข้ามช่องว่างและตัวคั่นด้วยความเร็วระดับฮาร์ดแวร์
* **ขนาดไบนารีจิ๋วเพียง 86 KB:** เล็กกว่า GNU Bash ดั้งเดิม (1.9 MB) ถึง **22 เท่า** ด้วยการรีดไบนารีผ่าน `-flto`, `-Wl,-dead_strip` และการขจัด Dead Code ระดับไบต์
* **เปิดเครื่องเร็วระดับฟ้าแลบ (~0.3 ms):** โหลดเข้าแรมเร็วกว่าเชลล์ทั่วไป 14 เท่า และกินแรมเริ่มต้นเพียง 620 KB
* **ความถูกต้องสมบูรณ์แบบ (100.0% Parity):** ผ่านการทดสอบ Differential Testing 1,000 ข้อ 10 โมดูล ประชันผลลัพธ์กับ macOS Bash ทุกตัวอักษร

---

## 🗺️ แผนที่หลักสูตร 7 ภาค (Curriculum Roadmap)

```
┌────────────────────────────────────────────────────────────────────────┐
│                        ผ่าสถาปัตยกรรม ZBASH (25 บท)                    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
    ┌───────────────────────────────┼───────────────────────────────┐
    ▼                               ▼                               ▼
[ ภาคที่ 1: มรดกและปรัชญา ]   [ ภาคที่ 2: สถาปัตยกรรม Lexer/Parser ] [ ภาคที่ 3: ระบบขยายตัวแปร ]
• มรดก 40 ปี สู่ zbash        • ARM64 NEON SIMD Lexer         • Parameter Expansion
• ปรัชญา 86 KB Bare-Metal     • ปราบพยศ Quoting Hell          • Arithmetic Engine $((...))
• ภาพรวมสถาปัตยกรรม           • ออกแบบ AST ขนาดกะทัดรัด       • โครงสร้าง Indexed Arrays
    │                               │                               │
    └───────────────────────────────┼───────────────────────────────┘
                                    │
    ┌───────────────────────────────┴───────────────────────────────┐
    ▼                                                               ▼
[ ภาคที่ 4: การประมวลผลคำสั่งและท่อไปป์ ]                  [ ภาคที่ 5: การทดสอบระดับโลก ]
• fork(), execve(), Process Lifecycle                   • ศาสตร์แห่ง Differential Testing
• ท่อและทางเบี่ยง: pipe(), dup2()                        • กายวิภาคชุดทดสอบ 1,000 ข้อ
• ลูปและเงื่อนไข (while/for/case)                       • ล่าบั๊กขอบเหวและ Memory Leaks
• Scope Isolation: Subshells vs Grouping
• ผ่าเบื้องหลังคำสั่ง Builtins (cd, export, read)
    │                                                               │
    └───────────────────────────────┬───────────────────────────────┘
                                    │
    ┌───────────────────────────────┴───────────────────────────────┐
    ▼                                                               ▼
[ ภาคที่ 6: ขอบฟ้าใหม่และจิตวิญญาณ ]             [ ภาคที่ 7: ภาคพิเศษ: Advanced Systems ]
• พอร์ตสู่ WebAssembly & WebOS                  • สร้าง Readline ไร้ Dependency (VT100)
• จิตวิญญาณแห่งผู้สร้างซอฟต์แวร์อธิปไตย           • การจัดการสัญญาณเคอร์เนลและ Job Control
                                                • เวกเตอร์สแกนเนอร์ Pathname Globbing
                                                • Tab Autocompletion และ $PATH Hashing
                                                • ศิลปะการรีดไบนารีสู่ 86 KB
                                                • ตัวดีบั๊กและตรวจสอบสคริปต์ (-x, -v, -n)
```

---

## 📖 สารบัญครบทั้ง 25 บท (ฉบับสองภาษา)

| ภาค | บทที่ | ชื่อบทภาษาไทย | English Title |
| :---: | :---: | :--- | :--- |
| **I** | **บทที่ 1** | [การเดินทางของ Unix Shell: จากมรดก 40 ปี สู่ zbash](th/ch01_the_genesis.md) | [The Unix Shell Journey: From a 40-Year Legacy to zbash](en/ch01_the_genesis.md) |
| **I** | **บทที่ 2** | [ปรัชญาการออกแบบ: 86 KB, Zero Dependencies, และ Bare-Metal](th/ch02_design_philosophy.md) | [Design Philosophy: 86 KB, Zero Dependencies, Bare-Metal Silicon](en/ch02_design_philosophy.md) |
| **I** | **บทที่ 3** | [ภาพรวมสถาปัตยกรรม: จากการกดคีย์บอร์ด สู่ Process Execution](th/ch03_architecture_overview.md) | [Architectural Overview: From Keystroke to Process Execution](en/ch03_architecture_overview.md) |
| **II** | **บทที่ 4** | [ความลับของ Lexer: เร่งความเร็วด้วย 128-bit ARM64 NEON SIMD](th/ch04_simd_vector_lexer.md) | [The Lexer Secret: 128-bit ARM64 NEON SIMD Acceleration](en/ch04_simd_vector_lexer.md) |
| **II** | **บทที่ 5** | [ปราบพยศไวยากรณ์: ถอดรหัส Quoting Hell และการตัดคำ](th/ch05_grammar_quoting_hell.md) | [Taming the Grammar: Deconstructing Quoting Hell & Tokenization](en/ch05_grammar_quoting_hell.md) |
| **II** | **บทที่ 6** | [สร้าง Abstract Syntax Tree (AST) ขนาดกะทัดรัด](th/ch06_ast_construction.md) | [Constructing a Compact Abstract Syntax Tree (AST)](en/ch06_ast_construction.md) |
| **III** | **บทที่ 7** | [ศิลปะแห่ง Parameter Expansion: ตัด สไลซ์ แทนที่ และความยาว](th/ch07_parameter_expansion.md) | [The Art of Parameter Expansion: Slicing, Defaults, and Lengths](en/ch07_parameter_expansion.md) |
| **III** | **บทที่ 8** | [คณิตศาสตร์ในตัวเชลล์: สร้างตัวประเมินผลนิพจน์ $((...))](th/ch08_arithmetic_engine.md) | [Embedded Math: Building the Arithmetic Evaluation Engine](en/ch08_arithmetic_engine.md) |
| **III** | **บทที่ 9** | [อาเรย์ไร้ Overhead: โครงสร้างข้อมูล Indexed Array ในภาษา C](th/ch09_arrays_in_c.md) | [Zero-Overhead Indexed Arrays in C](en/ch09_arrays_in_c.md) |
| **IV** | **บทที่ 10** | [ดำดิ่งสู่ OS: ความลับของ fork(), execve() และ Process Lifecycle](th/ch10_fork_exec_internals.md) | [OS Deep Dive: Mastering fork(), execve(), and Process Lifecycles](en/ch10_fork_exec_internals.md) |
| **IV** | **บทที่ 11** | [ท่อและทางเบี่ยง: pipe(), dup2() และปริศนา Trailing Loop](th/ch11_pipes_and_redirections.md) | [Pipes and Redirections: pipe(), dup2(), and Trailing Loops](en/ch11_pipes_and_redirections.md) |
| **IV** | **บทที่ 12** | [การควบคุมทิศทางและลูป: while, for, until, if-then-else, case](th/ch12_control_flow_loops.md) | [Control Flow and Loops: while, for, until, if-then-else, case](en/ch12_control_flow_loops.md) |
| **IV** | **บทที่ 13** | [การแยกแยะ Scope: Subshell (...) vs Grouping {...}](th/ch13_subshells_and_scope.md) | [Scope Isolation: Subshells (...) vs Grouping {...}](en/ch13_subshells_and_scope.md) |
| **IV** | **บทที่ 14** | [เบื้องหลังคำสั่ง Builtins: ทำไม cd, export, read ต้องเขียนในตัว](th/ch14_builtin_commands.md) | [Behind Builtin Commands: Why cd, export, and read Must Be Internal](en/ch14_builtin_commands.md) |
| **V** | **บทที่ 15** | ["เร็วแต่ผลลัพธ์ผิด ย่อมไร้ค่า": ศาสตร์แห่ง Differential Testing](th/ch15_differential_testing.md) | ["Fast with Wrong Output Is Worthless": Differential Testing](en/ch15_differential_testing.md) |
| **V** | **บทที่ 16** | [ผ่ากายวิภาคชุดทดสอบ 1,000 ข้อ: 10 โมดูลประชัน macOS Bash](th/ch16_anatomy_of_1000_tests.md) | [Anatomy of the 1,000-Test Suite: 10 Modules vs macOS Bash](en/ch16_anatomy_of_1000_tests.md) |
| **V** | **บทที่ 17** | [ล่าบั๊กขอบเหว: เมมโมรี่รั่ว ลูปค้าง และการจัดการ Trap สัญญาณ](th/ch17_hunting_edge_cases.md) | [Hunting the Toughest Edge Cases: Memory Leaks and Hangs](en/ch17_hunting_edge_cases.md) |
| **VI** | **บทที่ 18** | [จาก Terminal สู่เว็บเบราว์เซอร์: การพอร์ต zbash สู่ WebAssembly](th/ch18_wasm_webos_porting.md) | [From Terminal to Browser: Porting zbash to WebAssembly & WebOS](en/ch18_wasm_webos_porting.md) |
| **VI** | **บทที่ 19** | [จิตวิญญาณแห่งผู้สร้าง: จากจุดเริ่มต้นสู่ผลงานระดับโลก](th/ch19_the_creators_mindset.md) | [The Creator's Mindset: Crafting Sovereign Systems Software](en/ch19_the_creators_mindset.md) |
| **VII** | **บทที่ 20** | [สร้าง Readline ไร้ Dependency: ระบบ Line Editor และ VT100](th/ch20_zero_dep_readline.md) | [Zero-Dependency Readline: VT100 Line Editing from Scratch](en/ch20_zero_dep_readline.md) |
| **VII** | **บทที่ 21** | [การควบคุมสัญญาณระดับเคอร์เนล: Job Control และ Signals](th/ch21_signals_and_job_control.md) | [Kernel Signal Management: Job Control and Signals](en/ch21_signals_and_job_control.md) |
| **VII** | **บทที่ 22** | [เวกเตอร์สแกนเนอร์ความเร็วสูง: การขยาย Pathname & Wildcards](th/ch22_pathname_globbing.md) | [High-Velocity Vector Scanner: Pathname Expansion & Wildcards](en/ch22_pathname_globbing.md) |
| **VII** | **บทที่ 23** | [ระบบ Autocompletion & Command Lookup: Tab และ Fast $PATH Hash](th/ch23_tab_completion_path_hashing.md) | [Interactive Autocompletion & Command Lookup: Tab & $O(1)$ Hash](en/ch23_tab_completion_path_hashing.md) |
| **VII** | **บทที่ 24** | [ศิลปะการรีดไบนารีระดับเอ็กซ์ตรีม: จาก C Source สู่ 86 KB Executable](th/ch24_binary_minimization_86kb.md) | [Extreme Binary Minimization: From C Source to 86 KB Executable](en/ch24_binary_minimization_86kb.md) |
| **VII** | **บทที่ 25** | [เครื่องยนต์ตรวจสอบและดีบั๊กสคริปต์: สร้าง Execution Tracing (-x, -v, -n)](th/ch25_script_debugger_tracing.md) | [Script Debugging & Execution Tracing: Building -x, -v, -n, $LINENO](en/ch25_script_debugger_tracing.md) |

---

## 🎯 กลุ่มเป้าหมายและพื้นฐานที่จำเป็น

หนังสือเล่มนี้เหมาะสำหรับ:
* **วิศวกรซอฟต์แวร์ระบบ (Systems Engineers):** ผู้เขียนภาษา C, C++, Rust, Zig หรือ Go ที่ต้องการเข้าใจการทำงานของ POSIX Execution Model ในระดับลึก
* **นักศึกษาและคณาจารย์ด้านวิทยาการคอมพิวเตอร์:** ใช้เป็นตำราอ้างอิงและกรณีศึกษาจริงในวิชา Operating Systems, Compilers และ Runtime Engineering
* **วิศวกร DevOps / SRE / Cloud:** ผู้ดูแล Shell Scripts ในระดับโครงสร้างพื้นฐาน ที่ต้องการเข้าใจการทำงานใต้ฝากระโปรงอย่างแท้จริง

**พื้นฐานที่ควรมี:** มีความคุ้นเคยกับภาษา C เบื้องต้น (Pointers, Structs, Memory Allocation) และการใช้งาน Command Line ทั่วไป

---

## 🔗 โค้ดต้นฉบับประกอบหนังสือ

ซอร์สโค้ดภาษา C และ ARM64 Assembly ทั้งหมดที่อ้างอิงในหนังสือเล่มนี้ พัฒนาและเปิดเป็นโอเพนซอร์สอยู่ที่:  
👉 **[https://github.com/chin6700x/zbash](https://github.com/chin6700x/zbash)**

---

## 📄 ลิขสิทธิ์และสัญญาอนุญาต (License)

สงวนลิขสิทธิ์ © 2026 โดย **วิโรจน์ ชูแก้ว ([`chin6700x`](https://github.com/chin6700x))**

เนื้อหาทางวรรณกรรม คำอธิบาย ผังไดอะแกรม และโครงสร้างการสอนทั้งหมดในหนังสือเล่มนี้ เผยแพร่ภายใต้สัญญาอนุญาตสากล **[Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International Public License (CC BY-NC-ND 4.0)](LICENSE)**

* **คุณมีอิสระในการ:** ทำซ้ำ แจกจ่าย และส่งต่อผลงานนี้ในสื่อหรือรูปแบบใดก็ได้ เพื่อวัตถุประสงค์ที่ไม่ใช่เชิงพาณิชย์
* **ภายใต้เงื่อนไขดังต่อไปนี้:**
  * **Attribution (แสดงที่มา):** ต้องให้เกียรติระบุชื่อผู้เขียน **วิโรจน์ ชูแก้ว (`chin6700x`)** แนบลิงก์ไปยังสัญญาอนุญาต และระบุว่ามีการเปลี่ยนแปลงใดๆ หรือไม่
  * **NonCommercial (ไม่ใช้เพื่อการค้า):** ห้ามนำผลงานนี้ไปใช้เพื่อแสวงหาผลประโยชน์ทางการค้าหรือค่าตอบแทนทางการเงิน โดยไม่ได้รับความยินยอมเป็นลายลักษณ์อักษรจากผู้เขียน
  * **NoDerivatives (ไม่ดัดแปลง):** หากคุณนำผลงานนี้ไปดัดแปลง แปลงรูป หรือต่อยอด คุณต้องไม่เผยแพร่ผลงานดัดแปลงเหล่านั้นต่อสาธารณะ
