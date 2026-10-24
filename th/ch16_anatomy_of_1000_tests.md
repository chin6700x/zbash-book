# บทที่ 16: ผ่าโครงสร้างชุดทดสอบ 10 โมดูล 1,000 ข้อ vs macOS Bash
> **"Anatomy of the 1,000-Test Suite: Dissecting 10 Differential Parity Modules"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch16_anatomy_of_1000_tests.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"There are two ways of constructing a software design: One way is to make it so simple that there are obviously no deficiencies, and the other way is to make it so complicated that there are no obvious deficiencies."**  
> — **C.A.R. Hoare**  
> *คำแปล:* "มีสองวิธีในการสร้างการออกแบบซอฟต์แวร์: วิธีแรกคือทำให้มันเรียบง่ายจนเห็นชัดเจนว่าไม่มีจุดบกพร่อง และวิธีที่สองคือทำให้มันซับซ้อนจนมองไม่เห็นจุดบกพร่องอย่างชัดเจน"
> 
> **"Testing shows the presence, not the absence of bugs."**  
> — **Edsger W. Dijkstra**  
> *คำแปล:* "การทดสอบแสดงให้เห็นถึงการมีอยู่ของบั๊ก ไม่ใช่การไม่มีอยู่ของมัน"

---

## 🧭 1. กายวิภาคแห่งความสมบูรณ์แบบ: 1,000 การทดสอบ

เมื่อเราตั้งเป้าหมายว่า `zbash` จะต้องเป็น **True Drop-in Replacement** สำหรับ macOS Bash เราไม่สามารถพึ่งพาความรู้สึกหรือการทดสอบเพียงไม่กี่คำสั่งได้  
เราจึงได้สร้าง **ชุดทดสอบมาตรฐานระดับมหากาพย์ (The 1,000-Test Differential Suite)** ขึ้นมาในโฟลเดอร์ [`test/1000-test/`](test/1000-test/)

ชุดทดสอบนี้ถูกออกแบบให้เป็น **สถาปัตยกรรมแบบแยกโมดูล (Decoupled Modular Architecture)** แบ่งออกเป็น 10 โมดูล แต่ละโมดูลมี 100 ข้อทดสอบที่เจาะลึกเฉพาะทาง รันควบคู่กันระหว่าง `zbash` กับ `/bin/bash` ของระบบ macOS แบบข้อต่อข้อ

> [!TIP]
> **ระบบการทดสอบแบบหลายระดับ (Multi-Tier Testing Landscape):**  
> ชุดทดสอบ 1,000 ข้อในบทนี้คือ **Phase 1: Foundation Parity Suite** ที่สร้างรากฐานความเข้ากันได้แบบบิตต่อบิต ก่อนที่ในเวลาต่อมาโครงการจะขยายสู่ **Phase 2: Enterprise Advance Suite** (อีก 1,000 ข้อ ใน [`test/advance-test/`](test/advance-test/) ครอบคลุมงาน Log ETL, API Stream, และ Process Supervision) และ **GNU Conformance Suite** (4,600 ข้อ ใน [`test/GNU-TEST/`](test/GNU-TEST/)) ทำให้ `zbash` มีการทดสอบสะสมรวมกันกว่าหลายพันข้อ

ในบทนี้ เราจะมาผ่าโครงสร้างของทั้ง 10 โมดูลปฐมภูมิ เพื่อดูว่าการสร้างระบบทดสอบระดับนี้ ต้องครอบคลุมอะไรบ้าง!

---

## 🏗️ 2. แผนที่ 10 โมดูลการทดสอบ (The 10 Battlegrounds)

```
                            [ 1,000 PARITY TESTS ]
                                      │
        ┌──────────────┬──────────────┼──────────────┬──────────────┐
        ▼              ▼              ▼              ▼              ▼
   [ Module 1 ]   [ Module 2 ]   [ Module 3 ]   [ Module 4 ]   [ Module 5 ]
     Builtins      Arithmetic      Variables        Loops      Conditionals
   (100 Tests)    (100 Tests)    (100 Tests)    (100 Tests)    (100 Tests)
        │              │              │              │              │
        ▼              ▼              ▼              ▼              ▼
   [ Module 6 ]   [ Module 7 ]   [ Module 8 ]   [ Module 9 ]   [ Module 10 ]
    Functions      Pipelines      Subshells        Arrays      Stress/Unicode
   (100 Tests)    (100 Tests)    (100 Tests)    (100 Tests)    (100 Tests)
```

---

## 🔍 3. เจาะลึกรายละเอียดแต่ละโมดูล

### 📦 โมดูลที่ 1: `test_01_builtins_core.sh` (ข้อที่ #1 - #100)
* **หัวข้อ:** คำสั่งในตัวและฟังก์ชันพื้นฐาน
* **กรณีทดสอบหลัก:**
  - `echo` กับตัวเลือก `-n`, `-e`, escape sequences (`\n`, `\t`, `\\`)
  - `cd` สู่พาธปกติ, `cd -`, `cd ..`, การจัดการ `$PWD` และ `$OLDPWD`
  - `pwd` ตรวจสอบพาธปัจจุบัน
  - `export` และ `unset` สำหรับตัวแปรสภาพแวดล้อม
  - `true`, `false`, `:`, และ `exit [N]`

### 📦 โมดูลที่ 2: `test_02_arithmetic_compound.sh` (ข้อที่ #101 - #200)
* **หัวข้อ:** นิพจน์คณิตศาสตร์ `$(( ... ))` และ `(( ... ))`
* **กรณีทดสอบหลัก:**
  - ลำดับความสำคัญของตัวดำเนินการ (`+`, `-`, `*`, `/`, `%`, `**`)
  - การดำเนินการระดับบิต (`<<`, `>>`, `&`, `^`, `|`, `~`)
  - ลอจิกและการเปรียบเทียบ (`&&`, `||`, `==`, `!=`, `<`, `<=`, `>`, `>=`)
  - Prefix/Postfix Increment (`++x`, `x++`, `--x`, `x--`)
  - เลขฐาน 16 (`0xFF`), เลขฐาน 8 (`077`), และ Ternary (`a ? b : c`)

### 📦 โมดูลที่ 3: `test_03_variables_expansion.sh` (ข้อที่ #201 - #300)
* **หัวข้อ:** การขยายตัวแปรและการตัดแต่งสตริง
* **กรณีทดสอบหลัก:**
  - ตัวแปรพื้นฐาน `$VAR`, `${VAR}`
  - ค่าเริ่มต้น `${VAR:-default}`, `${VAR:=assign}`
  - การตรวจสอบค่า `${VAR:+alternate}`, `${VAR:?error}`
  - การนับความยาวสตริง `${#VAR}`
  - Substring Slicing `${VAR:offset:len}`
  - ตัวแปรพิเศษ `$?`, `$$`, `$#`, `$0`, `$@`, `$*`

### 📦 โมดูลที่ 4: `test_04_control_flow_loops.sh` (ข้อที่ #301 - #400)
* **หัวข้อ:** ลูปและโครงสร้างควบคุม
* **กรณีทดสอบหลัก:**
  - `for var in words; do ... done`
  - C-Style For Loop: `for (( i=0; i<10; i++ )); do ... done`
  - `while [ condition ]; do ... done`
  - `until [ condition ]; do ... done`
  - การกระโดดข้ามลูป: `break`, `continue`, `break 2`, `continue 2`
  - ลูปซ้อนกัน 3 ชั้นพร้อมเงื่อนไขซับซ้อน

### 📦 โมดูลที่ 5: `test_05_conditionals_test.sh` (ข้อที่ #401 - #500)
* **หัวข้อ:** ตัวประเมินเงื่อนไข `if`, `test`, และ `[`
* **กรณีทดสอบหลัก:**
  - `if ... then ... elif ... else ... fi` ซ้อนกันลึก 4 ชั้น
  - ตัวดำเนินการไฟล์: `-f`, `-d`, `-e`, `-s`, `-r`, `-w`, `-x`
  - ตัวดำเนินการสตริง: `-z`, `-n`, `=`, `!=`
  - ตัวดำเนินการตัวเลข: `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`
  - ลอจิกผสม: `-a` (AND), `-o` (OR), `!` (NOT)

### 📦 โมดูลที่ 6: `test_06_functions_scope.sh` (ข้อที่ #501 - #600)
* **หัวข้อ:** ฟังก์ชัน ขอบเขตตัวแปร และ Call Stack
* **กรณีทดสอบหลัก:**
  - การประกาศฟังก์ชันรูปแบบ `fname() { ... }` และ `function fname { ... }`
  - พารามิเตอร์ของฟังก์ชัน `$1`, `$2`, `$@`, `$#`
  - การคืนค่าด้วย `return [N]`
  - ตัวแปรเฉพาะที่ `local var=val`
  - ฟังก์ชันเรียกตัวเอง (Recursion) ลึก 15 ระดับ
  - การกู้คืนค่าตัวแปรใน Stack หลังฟังก์ชันจบการทำงาน

### 📦 โมดูลที่ 7: `test_07_pipelines_redirections.sh` (ข้อที่ #601 - #700)
* **หัวข้อ:** ท่อไปป์ การเปลี่ยนทิศทางไฟล์ และ Here-Strings
* **กรณีทดสอบหลัก:**
  - Simple Pipes: `cmd1 | cmd2`
  - Multi-Stage Pipes: `cmd1 | cmd2 | cmd3 | cmd4`
  - Redirections: `>`, `>>`, `<`, `2>`, `2>>`, `2>&1`, `>&2`
  - Here-Strings: `cmd <<< "text"`
  - Trailing Loop Redirections: `while read line; do ... done < file`
  - การส่งข้อมูลผ่านท่อขนาดใหญ่ระดับ 100,000 บรรทัด

### 📦 โมดูลที่ 8: `test_08_subshells_grouping.sh` (ข้อที่ #701 - #800)
* **หัวข้อ:** Subshells `( ... )` และ Command Grouping `{ ...; }`
* **กรณีทดสอบหลัก:**
  - การแยกขาดของหน่วยความจำใน Subshell
  - การเปลี่ยนแปลงค่าตัวแปรแบบ In-Place ใน Grouping
  - Subshell ภายในท่อไปป์ `(echo a; echo b) | grep a`
  - Subshell ซ้อนกันหลายชั้น `(( (echo deep) ))`
  - การดักจับ Exit Code ของ Subshell

### 📦 โมดูลที่ 9: `test_09_arrays_quoting.sh` (ข้อที่ #801 - #900)
* **หัวข้อ:** Indexed Arrays และกลไก Quoting
* **กรณีทดสอบหลัก:**
  - การสร้างอาร์เรย์ `arr=(one two "three with spaces")`
  - การกำหนดค่าดัชนีเดี่ยว `arr[0]=val`, `arr[10]=val` (Sparse Array)
  - การขยายสมาชิก `${arr[@]}` และ `${arr[*]}`
  - การนับสมาชิก `${#arr[@]}` และความยาวสมาชิก `${#arr[i]}`
  - การแสดงรายการดัชนี `${!arr[@]}`
  - Single Quotes, Double Quotes, และ Backslash Escapes ในอาร์เรย์

### 📦 โมดูลที่ 10: `test_10_edge_cases_stress.sh` (ข้อที่ #901 - #1000)
* **หัวข้อ:** กรณีขอบ อักขระ Unicode ภาษาไทย และ Stress Testing
* **กรณีทดสอบหลัก:**
  - สตริงภาษาไทย UTF-8: `echo "สวัสดีชาวโลก"`, ตัวแปรภาษาไทย
  - ช่องว่างหลายรูปแบบในบรรทัดเดียวกัน (Multiple spaces, tabs, empty lines)
  - คำสั่งเปล่าและคอมเมนต์ล้วน
  - การรันลูป 10,000 รอบเพื่อตรวจจับ Memory Leak
  - คำสั่งที่เกิดข้อผิดพลาดรุนแรง เช่น Command Not Found (127), Permission Denied (126)

---

## 🏆 4. ผลการทดสอบระดับ Master Orchestrator

เมื่อเรารันสคริปต์ Master Runner [`test/1000-test/run_1000_differential.sh`](test/1000-test/run_1000_differential.sh):

```
================================================================================
🚀 ZBASH: 1,000 DIFFERENTIAL PARITY TEST MASTER SUITE
   Bare-Metal ARM64 Silicon vs macOS BSD /bin/bash (Bit-for-Bit Exact Proof)
================================================================================

▶ RUNNING MODULE: test_01_builtins_core.sh (Core Builtins & Basic Commands)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_02_arithmetic_compound.sh (Compound Math & Arithmetic)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_03_variables_expansion.sh (Variable Expansions & Math)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_04_control_flow_loops.sh (Loops & Control Flow)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_05_conditionals_test.sh (Conditionals & Test Operators)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_06_functions_scope.sh (Functions, Local Scope & Stack)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_07_pipelines_redirections.sh (Pipelines & Redirections)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_08_subshells_grouping.sh (Subshells & Grouping)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_09_arrays_quoting.sh (Arrays & Quoting Mechanics)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

▶ RUNNING MODULE: test_10_edge_cases_stress.sh (Edge Cases, Unicode, Stress)...
  RESULTS: 100 PASS, 0 FAIL (100.0%)

================================================================================
🎉 MASTER SUITE COMPLETE: 1,000 / 1,000 TESTS PASSED (100.0%)
   ELAPSED TIME: 1.842 SECONDS ⚡
================================================================================
```

---

## 🎯 5. สรุปบทเรียนสำหรับนักพัฒนา

1. **แบ่งปัญหาใหญ่ออกเป็นโมดูลอิสระ:** การแบ่ง 1,000 ข้อเป็น 10 โมดูล ทำให้การตรวจจับและโฟกัสแก้ไขบั๊กทำได้เป็นหมวดหมู่ ไม่สับสน
2. **ทดสอบสิ่งที่ไม่คาดคิดเสมอ:** การทดสอบไม่ได้มีไว้สำหรับโค้ดที่ทำงานได้ดี แต่มีไว้สำหรับเคสแปลกๆ เช่น ภาษาไทย, ช่องว่างประหลาด, และคำสั่งที่ไม่มีอยู่จริง
3. **ความเร็วของการรันชุดทดสอบคือหัวใจของการพัฒนา:** การที่ชุดทดสอบ 1,000 ข้อสามารถรันเสร็จใน 1.8 วินาทีบน Apple Silicon ทำให้นักพัฒนาสามารถรันซ้ำได้ทุกครั้งที่มีการแก้โค้ดเพียงบรรทัดเดียว!

ในบทที่ 17 เราจะมาเปิดบันทึกภาคสนาม: **บันทึกการล่าบั๊กสุดแสบ: Memory Leaks, Infinite Redirections, และ Call Stack Overflow**!
