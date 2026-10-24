# บทที่ 12: ลูปและเงื่อนไข: while, for, until, if-then-else, case-esac
> **"Control Flow Dynamics: Designing Nested Conditionals, C-Style Loops, and Loop Branch Interruptions in C"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch12_control_flow_loops.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Simplicity is prerequisite for reliability."**  
> — **Edsger W. Dijkstra**  
> *คำแปล:* "ความเรียบง่ายคือเงื่อนไขเบื้องต้นที่ขาดไม่ได้ของความน่าเชื่อถือและความเสถียรภาพ"
> 
> **"Our creative thinking must be backed by rigorous testing and disciplined execution."**  
> — **Bjarne Stroustrup**  
> *คำแปล:* "ความคิดสร้างสรรค์ของเราต้องได้รับการสนับสนุนด้วยการทดสอบอันเข้มงวดและการลงมือทำอย่างมีวินัย"

---

## 🧭 1. ลมหายใจแห่งการเขียนโปรแกรม: โครงสร้างการควบคุม (Control Flow)

หากไม่มีคำสั่งเงื่อนไขและลูป เชลล์สคริปต์ก็คงเป็นได้เพียงแค่ "ไฟล์บันทึกรายการคำสั่งต่อเนื่อง" (Batch script) ธรรมดาๆ ที่รันจากบนลงล่างแล้วจบไป  
แต่สิ่งที่ทำให้ Bash กลายเป็นภาษาคอมพิวเตอร์ที่ทรงพลังและครองโลกโครงสร้างพื้นฐานมาหลายทศวรรษ คือความสามารถในการตัดสินใจและการวนซ้ำ:
* **`if ... then ... elif ... else ... fi`:** การเลือกทางแยกตามสถานะการคืนค่า ($?$)
* **`while ... do ... done`:** วนซ้ำตราบเท่าที่เงื่อนไขยังเป็นจริง ($? == 0$)
* **`until ... do ... done`:** วนซ้ำตราบเท่าที่เงื่อนไขยังเป็นเท็จ ($? \neq 0$)
* **`for var in list; do ... done`:** การวนซ้ำตามรายการคำ
* **`for (( i=0; i<n; i++ )); do ... done`:** ลูปคณิตศาสตร์สไตล์ภาษา C (C-Style For Loop)
* **`break [N]` และ `continue [N]`:** การกระโดดข้ามลูปแบบหลายชั้น

ในบทนี้ เราจะมาแกะรอยวิธีสร้างระบบ Control Flow ทั้งหมดนี้ในภาษา C แบบ Bare-Metal ในขนาดเพียงไม่กี่ร้อยบรรทัด โดยไม่ต้องพึ่งพาตัวสร้าง Grammar หนักๆ อย่าง Bison หรือ Yacc!

---

## 🏗️ 2. สถาปัตยกรรมตัวแยกกิ่งเงื่อนไข `exec_if`

ในสคริปต์เชลล์ เงื่อนไข `if` สามารถซ้อนกันได้ลึกไม่จำกัด (Arbitrary Nesting Depth) เช่น:
```bash
if [ -f config.env ]; then
    if [ "$ENV" = "production" ]; then
        echo "Prod mode"
    fi
else
    echo "Default config"
fi
```
หากเราใช้ Regular Expression ทั่วไปจับคู่คำว่า `fi` จะทำให้จับคู่ผิดตัวทันที  
ใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1224 ถึง 1378 เราสร้างตัวแยกกิ่งที่ใช้ **ระดับความลึก (Nesting Depth Counter `if_d`)**:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1265 - 1285) */
    if (at_word_start) {
        if (is_word_match(p, "if", 2)) {
            if_d++; /* เจอลูปใน ให้เพิ่มระดับความลึก */
        } else if (is_word_match(p, "fi", 2)) {
            if (if_d > 0) {
                if_d--; /* ปิดลูปใน ลดระดับความลึก */
            } else {
                /* เจอปิดของลูปนอกสุด: จบการแยกกิ่งของ if นี้! */
                p += 2;
                break;
            }
        }
```

### การจัดเก็บและประเมินกิ่งก้าน (Branch Execution):
โครงสร้างจะรวบรวมกิ่งต่างๆ ออกเป็น Array ของ `branches`:
* `cond`: ข้อความเงื่อนไข
* `body`: คำสั่งที่จะรันหากเงื่อนไขผ่าน
* `is_else`: แฟล็กแสดงว่าเป็นกิ่งสุดท้ายหรือไม่

เมื่อรวบรวมเสร็จ โค้ดจะวนลูปประเมินเงื่อนไขด้วยฟังก์ชันหลักของเชลล์เอง:
```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1354 - 1365) */
    int cond_rc = zbash_run_line(state, branches[b].cond);
    if (cond_rc == 0) {
        executed = true;
        if (branches[b].body && branches[b].body[0]) {
            zbash_run_line(state, branches[b].body);
        }
        break; /* ประเมินผ่านกิ่งใดกิ่งหนึ่งแล้ว ให้ออกจาก if ทันที */
    }
```
**สังเกตความงดงาม:** เราใช้ `zbash_run_line(state, cond)` ซึ่งเป็นการเรียกซ้ำแบบ Recursive Execution ทำให้คำสั่งเงื่อนไขสามารถเป็นอะไรก็ได้ เช่น `test -f file`, `[ $a -eq $b ]`, `grep -q pattern text`, หรือแม้แต่ฟังก์ชันที่ผู้ใช้เขียนเอง!

---

## 🔄 3. วงจรลูป `exec_while` และ `exec_until`: เงื่อนไขและส่วนเนื้อหา

การทำงานของ `while` และ `until` มีความคล้ายคลึงกันเกือบ 100% ต่างกันเพียงแค่เงื่อนไขการหยุด:
* `while`: หยุดเมื่อ `cond_rc != 0`
* `until`: หยุดเมื่อ `cond_rc == 0`

ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1380 ถึง 1475:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1380 - 1475) */
static int exec_while(zbash_state_t *state, const char *cmd) {
    bool is_until = (strncmp(cmd, "until", 5) == 0);
    /* แยก cond_buf และ body_buf โดยนับ inner_loop_d */
    ...
    while (1) {
        int cond_rc = zbash_run_line(state, cond_buf);
        if (is_until ? (cond_rc == 0) : (cond_rc != 0)) {
            state->last_status = 0;
            break; /* เงื่อนไขบอกให้จบลูป */
        }

        if (body_buf[0]) {
            zbash_run_line(state, body_buf);
        }
```

---

## ⚡ 4. การจัดการคำสั่งกระโดดข้ามลูป: `break [N]` และ `continue [N]`

นี่คือจุดที่เชลล์เขียนเองส่วนใหญ่สอบตก! เมื่อมีลูปซ้อนกัน 3 ชั้น:
```bash
for i in 1 2 3; do
    for j in a b c; do
        if [ "$i" -eq 2 ] && [ "$j" = "b" ]; then
            break 2   # ต้องหลุดออกจากลูปทั้ง 2 ชั้นพร้อมกัน!
        fi
        echo "$i-$j"
    done
done
```
คำสั่ง `break 2` ต้องทำให้ลูป `j` หลุด และส่งทอดความต้องการหลุดไปยังลูป `i` ข้างนอกด้วย!

### กลไกการแก้ปัญหาใน `zbash`:
ในโครงสร้างหลัก [`include/zbash.h`](include/zbash.h) บรรทัดที่ 44 ถึง 47:
```c
/* include/zbash.h (บรรทัดที่ 44 - 47) */
    bool loop_break;
    int loop_break_count;
    bool loop_continue;
    int loop_continue_count;
```

เมื่อคำสั่ง `break 2` ถูกเรียกใน [`src/builtins/builtins.c`](src/builtins/builtins.c) บรรทัดที่ 628 ถึง 632:
```c
/* src/builtins/builtins.c */
    if (strcmp(cmd, "break") == 0) {
        state->loop_break = true;
        state->loop_break_count = (argc > 1) ? atoi(argv[1]) : 1;
        return 0;
    }
```
และในตัววนลูป [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1488 ถึง 1500:
```c
/* src/core/engine.c */
    if (state->loop_break) {
        state->loop_break_count--; /* ลดจำนวนชั้นที่ต้องการหลุด */
        if (state->loop_break_count <= 0) {
            state->loop_break = false;
            state->loop_break_count = 0;
        }
        break; /* หลุดออกจากลูปชั้นปัจจุบันทันที */
    }
```
หาก `break_count` ยังเหลือมากกว่า 0 ตัวแปร `state->loop_break` จะยังคงเป็น `true` ต่อไป ทำให้ลูปชั้นนอกที่ครอบอยู่เห็นสถานะนี้ และทำการ `break` ต่อทันทีเป็นทอดๆ จนครบจำนวนชั้นอย่างแม่นยำ!

---

## 🧮 5. ลูปคณิตศาสตร์ภาษา C: `exec_for_c`

ฟีเจอร์ที่นักพัฒนานิยมใช้เขียนการคำนวณรอบสูงใน Bash คือ:
```bash
for (( i = 0; i < 1000; i++ )); do
    sum=$(( sum + i ))
done
```
ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1516 ถึง 1628:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1516 - 1628) */
static int exec_for_c(zbash_state_t *state, const char *cmd) {
    /* 1. แยก 3 ส่วน: init_expr ; cond_expr ; step_expr */
    ...
    /* 2. รันคำสั่งเริ่มต้น */
    if (init_expr[0]) zbash_eval_arithmetic(state, init_expr);

    while (1) {
        /* 3. ประเมินเงื่อนไขด้วย Arithmetic Evaluator */
        if (cond_expr[0]) {
            long long cond_val = zbash_eval_arithmetic(state, cond_expr);
            if (cond_val == 0) break; /* ผลเป็น 0 คือเท็จ จบลูป */
        }

        /* 4. รันคำสั่งในตัวลูป */
        if (body_buf[0]) zbash_run_line(state, body_buf);

        /* จัดการ break / continue */
        if (state->loop_break) { ... break; }

        /* 5. รันคำสั่งเพิ่มค่ารอบ (Step Expression เช่น i++) */
        if (step_expr[0]) zbash_eval_arithmetic(state, step_expr);
    }
    return state->last_status;
}
```
การนำ **Arithmetic Engine** (จากบทที่ 8) มาขับเคลื่อนตัวลูป ทำให้เราได้ลูปความเร็วสูงที่เข้าใจนิพจน์ `i++`, `i += 2`, และสูตรคณิตศาสตร์ซับซ้อนได้ในตัวโดยอัตโนมัติ!

---

## 🧪 6. การทดสอบยืนยันความถูกต้องระดับ Parity (Differential Proof)

ในโมดูล `test/1000-test/test_04_control_flow_loops.sh` มีชุดทดสอบ 100 ข้อ:

| หมวดหมู่ | โครงสร้างคำสั่ง | รายละเอียดการทดสอบ | ผลลัพธ์ vs GNU Bash |
| :--- | :--- | :--- | :--- |
| **If-Elif-Else** | ซ้อนกัน 4 ชั้น | ตรวจสอบลำดับการเข้ากิ่ง | **PASS (100%)** |
| **While Loop** | นับรอบ 1 ถึง 1,000 | ความถูกต้องของค่าสุดท้าย | **PASS (100%)** |
| **Until Loop** | เงื่อนไขปฏิเสธ | การหยุดรอบเมื่อค่าเปลี่ยน | **PASS (100%)** |
| **For In Words** | สตริงที่มีช่องว่างและเว้นวรรค | การแยก Token ในรายการ | **PASS (100%)** |
| **C-Style For** | `for (( i=0; i<10; i+=2 ))` | การประเมินผลคณิตศาสตร์ | **PASS (100%)** |
| **Break & Continue** | `break 2`, `continue 2` | การกระโดดข้ามระดับชั้นลูป | **PASS (100%)** |

**ผลลัพธ์รวม:** `100 / 100 PASS (100.0%)` สมบูรณ์แบบ

---

## 🎯 7. สรุปบทเรียนสำหรับนักพัฒนา

1. **อย่ากลัวการ Recursion เมื่อออกแบบภาษาระดับระบบ:** การให้ `zbash_run_line()` สามารถถูกเรียกซ้ำจากภายในลูปและเงื่อนไข ทำให้สถาปัตยกรรมมีความเรียบง่าย กะทัดรัด และทรงพลัง
2. **จัดการ Stack และ Counters สำหรับ Nesting:** ตัวแปรนับระดับความลึก (`if_d`, `inner_loop_d`) ช่วยแก้ปัญหาการจับคู่คำสั่งปิดผิดตัวได้อย่างหมดจด
3. **State Flaging สำหรับการควบคุมหลายชั้น:** การส่งต่อตัวแปร `loop_break_count` ผ่านโครงสร้างสถานะหลัก ทำให้เรารองรับพฤติกรรมกระโดดข้ามลูปหลายชั้นตามมาตรฐาน POSIX ได้อย่างสง่างาม

ในบทที่ 13 เราจะไปเปิดโลกของขอบเขตตัวแปร: **ความต่างระดับชีวิต: Subshells `(...)` vs Grouping `{...}`**!
