# บทที่ 8: เครื่องคิดเลขในตัว: สร้างตัวประเมินคณิตศาสตร์ $(( a + b * c ))
> **"The Built-in Calculator: Building a Recursive Descent Arithmetic Evaluator"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch08_arithmetic_engine.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"First, solve the problem. Then, write the code."**  
> — **John Johnson**  
> *คำแปล:* "ขั้นแรก จงแก้ปัญหานั้นให้กระจ่างก่อน แล้วจึงลงมือเขียนโค้ด"
> 
> **"Science is what we understand well enough to explain to a computer. Art is everything else we do."**  
> — **Donald Knuth**  
> *คำแปล:* "วิทยาศาสตร์คือสิ่งที่เราเข้าใจดีพอที่จะอธิบายให้คอมพิวเตอร์ฟังได้ ส่วนศิลปะคือทุกสิ่งทุกอย่างที่เหลือนอกเหนือจากนั้น"

---

## 🧭 1. ทำไมเชลล์ต้องมีตัวประเมินคณิตศาสตร์ในตัวเอง?

ในยุคแรกเริ่มของ Unix หากคุณต้องการคำนวณเลขในเชลล์ คุณต้องเรียกโปรแกรมภายนอกอย่าง `expr` หรือ `bc`:
```bash
# วิธีการดั้งเดิมในยุค 1970: ต้อง fork process ใหม่ทุกครั้งที่บวกเลข!
result=$(expr $a + $b)
```
การเรียก `expr` หมายถึงเคอร์เนลต้อง `fork()` และ `execve()` โปรแกรมใหม่ขึ้นมาในหน่วยความจำ ซึ่งกินเวลาหลักหลายมิลลิวินาที หากคุณมีลูปที่ต้องบวกเลข 10,000 ครั้ง การใช้โปรแกรมภายนอกจะทำให้สคริปต์ช้าจนแทบหยุดนิ่ง

เพื่อแก้ปัญหานี้ POSIX และ Bash จึงกำหนดให้เชลล์ต้องมี **Arithmetic Expansion `$(( expression ))`** ฝังอยู่ในตัว  
ในบทนี้ เราจะมาดูการสร้าง **Recursive Descent Parser (ตัวแจงส่วนแบบสืบเชื้อสายเวียนเกิด)** ในภาษา C ที่รองรับ:
* การจัดลำดับความสำคัญของตัวดำเนินการ (Operator Precedence) เช่น คูณ/หาร ก่อนบวก/ลบ
* เลขฐานสิบหก (`0xFF`), เลขฐานแปด (`077`), และเลขฐานสิบ
* ตัวดำเนินการระดับบิต (`<<`, `>>`, `&`, `^`, `|`)
* การเพิ่ม/ลดค่าทั้งหน้าและหลัง (`++x`, `x++`, `--x`, `x--`)
* การยกกำลัง (`**`) และเงื่อนไขแบบสามส่วน (Ternary Operator `a ? b : c`)

ทั้งหมดนี้ถูกบรรจุไว้อย่างงดงามใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 197 ถึง 390!

---

## 🏗️ 2. ลำดับขั้นความสำคัญของตัวดำเนินการ (Precedence Cascade)

เคล็ดลับในการสร้างตัวคำนวณคณิตศาสตร์โดยไม่ต้องใช้ Library ภายนอก คือการเรียงฟังก์ชันเป็นชั้นๆ (Cascade Levels) จากตัวดำเนินการที่มีความสำคัญต่ำสุด ไปหาตัวที่มีความสำคัญสูงสุด:

```
eval_assign      (=, +=, -=, *=, /=)        [ความสำคัญต่ำสุด]
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
                                                                            └── eval_primary    (ตัวเลข, ตัวแปร, วงเล็บ) [ความสำคัญสูงสุด]
```

---

## 🔍 3. เจาะลึกโค้ด `eval_primary`: จัดการตัวเลข ตัวแปร และ `++`/`--`

ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 206 ถึง 255:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 206 - 255) */
static long long eval_primary(zbash_state_t *state, const char **p) {
    skip_ws(p);
    if (**p == '\0') return 0;

    /* 1. วงเล็บ ( expr ) เพื่อเปลี่ยนลำดับความสำคัญ */
    if (**p == '(') {
        (*p)++;
        long long val = eval_expr(state, p);
        skip_ws(p);
        if (**p == ')') (*p)++;
        return val;
    }

    /* 2. Prefix ++var หรือ --var */
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
        val++; // เพิ่มค่าขึ้นทันทีก่อนส่งกลับ
        char nbuf[32]; snprintf(nbuf, sizeof(nbuf), "%lld", val);
        setenv(vname, nbuf, 1); // บันทึกกลับลงตัวแปร
        return val;
    }

    /* 3. เครื่องหมายยูนารี +, -, ~, ! */
    if (**p == '+') { (*p)++; return eval_primary(state, p); }
    if (**p == '-') { (*p)++; return -eval_primary(state, p); }
    if (**p == '~') { (*p)++; return ~eval_primary(state, p); }
    if (**p == '!') { (*p)++; return !eval_primary(state, p); }

    /* 4. ตัวเลข: รองรับฐาน 16 (0x), ฐาน 8 (0), และฐาน 10 */
    if (isdigit((unsigned char)**p)) {
        char *end;
        long long val = strtoll(*p, &end, 0); // 0 ทำให้ strtoll รู้จัก 0x และ 0 อัตโนมัติ!
        *p = end;
        return val;
    }
```

### รายละเอียดที่น่าประทับใจ:
1. **พลังของ `strtoll(*p, &end, 0)` (บรรทัดที่ 262):**  
   การใส่พารามิเตอร์ฐานเป็น `0` ทำให้ฟังก์ชันตรวจจับคำนำหน้าเอง: ถ้าขึ้นต้นด้วย `0x` จะอ่านเป็นเลขฐาน 16, ถ้าขึ้นต้นด้วย `0` จะอ่านเป็นฐาน 8, และถ้าไม่มีจะอ่านเป็นฐาน 10 ทันที โดยไม่ต้องเขียนโค้ดเพิ่มแม้แต่บรรทัดเดียว!
2. **Side Effect ของ `++var` (บรรทัดที่ 232–233):**  
   ใน Bash เมื่อเขียน `$(( ++x ))` ตัวแปร `x` ในหน่วยความจำต้องถูกเปลี่ยนค่าไปด้วย เราจึงคำนวณ `val++` แล้วเซ็ตกลับด้วย `setenv(vname, nbuf, 1)`

---

## ✖️ 4. การจัดการคูณ/หาร และการป้องกันหารด้วยศูนย์

ดูบรรทัดที่ 315 ถึง 345 ใน [`src/core/engine.c`](src/core/engine.c):

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 315 - 345) */
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
                // ป้องกัน SIGFPE: ห้ามแครชเมื่อหารด้วย 0!
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

* **Zero-Crash Protection (บรรทัดที่ 326–330):**  
  หากผู้ใช้สั่ง `$(( 10 / 0 ))` ในระดับฮาร์ดแวร์ ซีพียูจะยิงสัญญาณ `SIGFPE` (Floating Point Exception) ซึ่งหากไม่ดักจับไว้จะทำให้ตัวเชลล์แครชทันที! เราจึงดัก `if (right == 0)` พิมพ์เตือนข้อผิดพลาดตามมาตรฐาน Bash และคืนค่าอย่างปลอดภัย

---

## 🏁 สรุปบทเรียนที่ 8

1. **In-Process Arithmetic:** การมีเครื่องคิดเลขในตัวทำให้เชลล์สามารถคำนวณลูปนับหมื่นรอบได้ในเสี้ยววินาทีโดยไม่ต้อง fork process ย่อย
2. **ความงดงามของ Recursive Descent:** การแบ่งฟังก์ชันตามระดับความสำคัญของตัวดำเนินการทำให้โค้ดเป็นระเบียบ เข้าใจง่าย และขยายต่อได้ไม่รู้จบ
3. **ความทนทานต่อข้อผิดพลาด:** การจัดการหารด้วยศูนย์ และการรองรับเลขฐานต่างๆ (Hex/Octal) ช่วยให้ผ่านการทดสอบ Differential Parity ได้ 100%

ใน **บทที่ 9** เราจะมาศึกษาโครงสร้างข้อมูลที่ซับซ้อนขึ้นไปอีกขั้น: **"โครงสร้าง Indexed Arrays ใน C แบบไร้ Overhead"** เพื่อดูว่าเราเก็บ Sparse Array ในภาษา C ได้อย่างไรครับ! 🚀📖
