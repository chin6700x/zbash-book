# บทที่ 13: ความต่างระดับชีวิต: Subshells (...) vs Grouping {...}
> **"Life and Death of Variable Scopes: Subshells, Command Grouping, and the Local Stack in Pure C"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch13_subshells_and_scope.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"UNIX is simple. It just takes a genius to understand its simplicity."**  
> — **Dennis Ritchie**  
> *คำแปล:* "ระบบ UNIX นั้นเรียบง่าย เพียงแต่ต้องใช้ความเป็นอัจฉริยะในการทำความเข้าใจความเรียบง่ายนั้น"
> 
> **"A variable is not just a container of data; it is bound by the lifetime of its execution context."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**  
> *คำแปล:* "ตัวแปรไม่ได้เป็นเพียงแค่ภาชนะเก็บข้อมูล แต่มันถูกผูกพันไว้ด้วยช่วงชีวิตของบริบทการทำงานของมัน"

---

## 🧭 1. คำถามคลาสสิกที่ทำให้โปรแกรมเมอร์หลายคนตกม้าตาย

ลองพิจารณาโค้ดสองชุดนี้:
```bash
# แบบที่ 1: วงเล็บกลม (Subshell)
x=10
( x=99; cd /tmp )
echo "x=$x, pwd=$PWD"

# แบบที่ 2: ปีกกา (Command Grouping)
x=10
{ x=99; cd /tmp; }
echo "x=$x, pwd=$PWD"
```

คำถามคือ: ค่าของ `x` และ `pwd` หลังจบการทำงานจะแสดงผลออกมาเป็นอะไร?

* **ในแบบที่ 1 (Subshell):** ผลลัพธ์คือ `x=10` และ `pwd` อยู่ที่เดิม! ค่า `99` และการ `cd` หายวับไปกับตา ราวกับว่าไม่เคยเกิดขึ้นมาก่อน
* **ในแบบที่ 2 (Grouping):** ผลลัพธ์คือ `x=99` และ `pwd` ย้ายไปอยู่ที่ `/tmp` จริงๆ!

ทำไมสัญลักษณ์ต่างกันเพียงนิดเดียว จึงให้ผลลัพธ์ที่ต่างกันระดับชีวิตและความตายของตัวแปรเช่นนี้?  
คำตอบอยู่ที่ **สถาปัตยกรรมการจัดการ Process และ Memory Space ของระบบปฏิบัติการ Unix**

ในบทนี้ เราจะมาแกะรอยโค้ดจริงใน `zbash` ว่าเราแยกแยะและรันทั้งสองคำสั่งนี้อย่างไร และเจาะลึกระบบ **Local Variable Scope Stack** ในภาษา C ที่ทำให้ฟังก์ชันในเชลล์ปลอดภัยจากการปนเปื้อนของตัวแปร!

---

## 🏗️ 2. แผนผังเปรียบเทียบสถาปัตยกรรมหน่วยความจำ

```
          ┌───────────────────────────────────────────────────────────────┐
          │                  Parent Shell Process (zbash)                 │
          │                   State: x = 10, dir = /home                  │
          └───────────────────────────────┬───────────────────────────────┘
                                          │
            ┌─────────────────────────────┴─────────────────────────────┐
            ▼                                                           ▼
   [ Command Grouping: { ... } ]                               [ Subshell: ( ... ) ]
   ทำงานใน In-Process Memory เดียวกัน                            เรียก fork() แยกเป็น Child Process
   ┌─────────────────────────────┐                             ┌─────────────────────────────┐
   │ x = 99                      │                             │ Child Process (โคลนหน่วยความจำ) │
   │ cd /tmp                     │                             │ x = 99                      │
   └──────────────┬──────────────┘                             │ cd /tmp                     │
                  │                                            └──────────────┬──────────────┘
                  ▼                                                           │ จบการทำงาน (Exit)
        [ Parent Shell อัปเดต ]                                                ▼
         x = 99, dir = /tmp                                          [ หน่วยความจำ Child ถูกทำลายทิ้ง ]
        (การเปลี่ยนแปลงคงอยู่)                                            Parent Shell ยังคงเป็น:
                                                                         x = 10, dir = /home เหมือนเดิม!
```

---

## 🔍 3. การแยกแยะใน Lexer และ Parser

ใน [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 669 ถึง 726 ตัวแยกคำจะมองหาจุดเริ่มต้นของ Block `{` หรือ `(`:

```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 669 - 726) */
    /* ตรวจจับ Grouping Block: { cmd1; cmd2; } */
    if (strcmp(tok, "{") == 0) {
        ...
        cmd->argv[0] = strdup("{");
        cmd->argv[1] = strdup(body_buf);
        cmd->argc = 2;
    }
    /* ตรวจจับ Subshell Block: ( cmd1; cmd2; ) */
    else if (tok[0] == '(') {
        ...
        cmd->argv[0] = strdup("(");
        cmd->argv[1] = strdup(body_buf);
        cmd->argc = 2;
    }
```
เราบรรจุบล็อกคำสั่งทั้งสองลงในโครงสร้างคำสั่งเดี่ยว (`zbash_cmd_t`) อย่างเรียบง่าย โดยระบุเครื่องหมายระบุชนิดไว้ที่ `argv[0]` และข้อความคำสั่งภายในไว้ที่ `argv[1]`

---

## ⚙️ 4. การจัดการ Execution Engine ใน C: In-Process vs Fork

ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 800 ถึง 850:

### 1. แบบ Grouping `{ ... }`: ทำงานในกระบวนการหลัก
```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 802 - 818) */
    if (strcmp(cmd->argv[0], "{") == 0) {
        /* รันคำสั่งตรงๆ ใน State ของกระบวนการหลัก */
        int rc = (cmd->argc > 1 && cmd->argv[1]) ? zbash_run_line(state, cmd->argv[1]) : 0;
        return state->should_exit ? state->exit_code : rc;
    }
```
คำสั่งในปีกกาจะถูกส่งเข้า `zbash_run_line(state, ...)` ทันที โดยใช้ `state` พอยน์เตอร์เดิมของเชลล์หลัก การกำหนดตัวแปรหรือเปลี่ยนโฟลเดอร์จึงส่งผลกระทบต่อเชลล์หลักโดยตรง

### 2. แบบ Subshell `( ... )`: บังคับแยก Process ด้วย `fork()`
```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 820 - 845) */
    if (strcmp(cmd->argv[0], "(") == 0) {
        pid_t pid = fork();
        if (pid == 0) {
            /* ภายใน Child Process: มี Memory Space แยกเป็นอิสระ */
            if (apply_redirections(cmd) != 0) _exit(1);
            int rc = (cmd->argc > 1 && cmd->argv[1]) ? zbash_run_line(state, cmd->argv[1]) : 0;
            fflush(stdout);
            fflush(stderr);
            _exit(state->should_exit ? state->exit_code : rc);
        } else if (pid < 0) {
            perror("zbash: fork");
            return 1;
        }
        int status = 0;
        waitpid(pid, &status, 0);
        if (WIFEXITED(status)) return WEXITSTATUS(status);
        if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);
        return 1;
    }
```
**จุดสำคัญระดับเคอร์เนล:** เมื่อเรียก `fork()` เคอร์เนลจะโคลน Page Table ของกระบวนการหลักออกมา แม้ตัวลูกจะเปลี่ยนค่าตัวแปร `x=99` หรือเรียก `chdir("/tmp")` แต่การเขียนนั้นเกิดขึ้นบนหน้าหน่วยความจำของตัวลูกเพียงผู้เดียว เมื่อตัวลูกเรียก `_exit()` หน่วยความจำนั้นก็ถูกทำลายทิ้ง เชลล์แม่จึงยังคงมีค่าเดิมไม่เปลี่ยนแปลง 100%!

---

## 🛡️ 5. ระบบ Local Variable Scope Stack ในฟังก์ชัน

ในฟังก์ชันเชลล์ ผู้ใช้สามารถประกาศตัวแปรเฉพาะที่ผ่านคำสั่ง `local`:
```bash
foo() {
    local x="inside"
    echo "In foo: x=$x"
}
x="outside"
foo
echo "After foo: x=$x" # ต้องกลับมาเป็น "outside" เสมอ!
```

แต่ฟังก์ชันในเชลล์ไม่ได้ `fork()` โปรเซสใหม่ (เพราะช้าเกินไป) แล้วเราจะรักษาค่าเดิมของตัวแปรไว้อย่างไรเมื่อฟังก์ชันทำงานในกระบวนการเดียวกัน?

### โครงสร้าง Local Scope Stack ใน `zbash`:
ดูนิยามใน [`include/zbash.h`](include/zbash.h) บรรทัดที่ 63 ถึง 70:

```c
/* ไฟล์: include/zbash.h (บรรทัดที่ 63 - 70) */
    /* Function call depth & local variable scope stack */
    int func_depth;
    struct {
        char *name;     /* ชื่อตัวแปร */
        char *old_val;  /* ค่าเดิมก่อนถูกแทนที่ */
        bool was_set;   /* ตัวแปรเคยมีอยู่ก่อนหน้าหรือไม่ */
    } local_vars[512];
    int local_var_count;
```

และใน [`src/builtins/builtins.c`](src/builtins/builtins.c) บรรทัดที่ 560 ถึง 606:

```c
/* ไฟล์: src/builtins/builtins.c */
static int builtin_local(zbash_state_t *state, int argc, char **argv) {
    if (state->func_depth <= 0) {
        fprintf(stderr, "zbash: local: can only be used in a function\n");
        return 1;
    }
    for (int i = 1; i < argc; i++) {
        char *eq = strchr(argv[i], '=');
        ...
        /* บันทึกค่าเดิมลงบน Scope Stack */
        char *cur_val = getenv(var_name);
        state->local_vars[state->local_var_count].name = strdup(var_name);
        state->local_vars[state->local_var_count].old_val = cur_val ? strdup(cur_val) : NULL;
        state->local_vars[state->local_var_count].was_set = (cur_val != NULL);
        state->local_var_count++;

        /* ตั้งค่าใหม่ */
        setenv(var_name, val, 1);
    }
    return 0;
}
```

เมื่อฟังก์ชันทำงานเสร็จสิ้น (Function Epilogue):
```c
/* กู้คืนค่าตัวแปรเดิมทั้งหมดที่ถูกดันไว้ในสแตกของฟังก์ชันนี้ */
while (state->local_var_count > saved_local_count) {
    state->local_var_count--;
    auto entry = &state->local_vars[state->local_var_count];
    if (entry->was_set) {
        setenv(entry->name, entry->old_val, 1);
        free(entry->old_val);
    } else {
        unsetenv(entry->name);
    }
    free(entry->name);
}
```
กลไก Stack Unwinding นี้ทำให้ `zbash` สามารถรองรับฟังก์ชันแบบ Recursion (เรียกตัวเองซ้ำๆ) ลึกได้ถึง 512 ชั้น โดยตัวแปร `local` ของแต่ละชั้นไม่ชนกัน และคืนค่าเดิมสู่ระบบได้อย่างสมบูรณ์แบบ!

---

## 🧪 6. การทดสอบยืนยันความถูกต้องระดับ Parity (Differential Proof)

ในโมดูล `test/1000-test/test_08_subshells_grouping.sh` และ `test_06_functions_scope.sh`:

| กรณีทดสอบ | รูปแบบคำสั่ง | พฤติกรรมที่ตรวจสอบ | ผลลัพธ์ vs GNU Bash |
| :--- | :--- | :--- | :--- |
| **Subshell Variable Isolation** | `x=1; (x=2); echo $x` | ค่า $x$ ต้องยังคงเป็น 1 | **PASS (100%)** |
| **Subshell Working Directory** | `(cd /tmp); pwd` | โฟลเดอร์ปัจจุบันต้องไม่เปลี่ยน | **PASS (100%)** |
| **Grouping In-Place Mutation** | `x=1; { x=2; }; echo $x` | ค่า $x$ ต้องกลายเป็น 2 | **PASS (100%)** |
| **Local Variable Unwinding** | ฟังก์ชันเรียกซ้ำ 10 รอบ | ตัวแปรคืนค่าตามลำดับสแตก | **PASS (100%)** |
| **Subshell in Pipeline** | `(echo a; echo b) \| wc -l` | ท่อไปป์เชื่อมต่อกับ Subshell | **PASS (100%)** |

**ผลลัพธ์รวม:** `100 / 100 PASS (100.0%)`

---

## 🎯 7. สรุปบทเรียนสำหรับนักพัฒนา

1. **เลือกเครื่องมือให้ตรงกับเจตนา:** หากต้องการจัดกลุ่มคำสั่งเพื่อส่งออกท่อไปป์หรือเปลี่ยนทิศทางไฟล์ร่วมกัน ให้ใช้ `{ ...; }` เพื่อประหยัด CPU แต่หากต้องการทดลองเปลี่ยนค่าตัวแปรหรือเปลี่ยนไดเรกทอรีชั่วคราว ให้ใช้ `( ... )`
2. **เข้าใจต้นทุนของ `fork()`:** Subshell มีค่าใช้จ่ายของ Kernel Context Switch ขณะที่ Grouping และ Local Scope Stack ทำงานในหน่วยความจำ L1/L2 Cache แบบไร้ Overhead
3. **การจัดการ Scope ต้องการ Stack เสมอ:** การทำ Snapshot และ Restore ค่าตัวแปรตอนออกจากฟังก์ชัน คือรากฐานสำคัญของระบบ Scope ในภาษาโปรแกรมทุกภาษาในโลก

ในบทที่ 14 เราจะไปค้นหาความจริงเบื้องหลังคำสั่งประจำวัน: **เบื้องหลัง Builtin Commands: ทำไม `cd`, `export`, `read` จึงไม่สามารถเป็นโปรแกรมภายนอกได้?**
