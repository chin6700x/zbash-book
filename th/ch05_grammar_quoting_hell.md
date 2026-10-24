# บทที่ 5: ปราบพยศไวยากรณ์: แกะรอย Quoting Hell (", ', \) และ Tokenizer
> **"Taming the Grammar: Surviving Quoting Hell (", ', \) and the Tokenizer"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch05_grammar_quoting_hell.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Quote your variables, or the shell will split your words into pieces you never intended."**  
> — **Shell Scripting Axiom**  
> *คำแปล:* "จงใส่เครื่องหมายคำพูดครอบตัวแปรของคุณ มิฉะนั้นเชลล์จะหั่นคำของคุณออกเป็นชิ้นๆ ในแบบที่คุณไม่ได้ตั้งใจ"
> 
> **"Clarity is better than cleverness."**  
> — **Tim Peters** (The Zen of Python)  
> *คำแปล:* "ความชัดเจนแจ่มแจ้ง มีค่ามากกว่าความฉลาดแกมโกง"

---

## 🧭 1. นรกแห่งเครื่องหมายคำพูด (Quoting Hell) คืออะไร?

หากคุณถามนักพัฒนาซอฟต์แวร์ที่เคยพยายามเขียน Shell ขึ้นมาเองว่า:  
**"อะไรคือส่วนที่ยาก น่าปวดหัว และเต็มไปด้วยบั๊กมากที่สุด?"**  
คำตอบ 99% ไม่ใช่เรื่องการ Fork Process หรือการต่อท่อไปป์ แต่คือ **"Quoting Hell (นรกของการจัดการเครื่องหมายคำพูด)"**!

ในภาษาการเขียนโปรแกรมทั่วไป เช่น C, Python, หรือ JavaScript กฎของสตริงมักจะตรงไปตรงมา: สตริงจะเริ่มด้วย `"` และจบด้วย `"` เสมอ  
แต่ใน Unix Shell เครื่องหมายคำพูดมีกฎหมาย 3 ฉบับซ้อนกันอยู่:

1. **Single Quotes (`'...'`):** สัจธรรมบริสุทธิ์ (Literal Mode) — ทุกสิ่งที่อยู่ข้างในจะไม่มีการแปลงค่าใดๆ ทั้งสิ้น ห้ามมีตัวแปร `$VAR` และแม้แต่ `\'` ก็ไม่สามารถหลบอักขระได้!
2. **Double Quotes (`"..."`):** โหมดกึ่งสอดแทรก (Interpolation Mode) — อนุญาตให้ขยายตัวแปร `$VAR`, ประเมินคณิตศาสตร์ `$((...))`, และแทนที่คำสั่ง `$(...)` ได้ แต่จะรักษาช่องว่าง (Whitespaces) ไม่ให้ถูกหั่นคำ (Word Splitting)
3. **Backslash (`\`):** การหลบอักขระเดี่ยว — ซึ่งจะมีพฤติกรรมแตกต่างกันอย่างสิ้นเชิงเมื่ออยู่นอกเครื่องหมายคำพูด กับเมื่ออยู่ข้างใน Double Quotes!

---

## 🔍 2. สเตตแมชชีนใน `src/parser/lexer.c`: จัดการ Backslash และ Quotes

เพื่อจัดการกับความซับซ้อนนี้โดยไม่ให้โค้ดกลายเป็นสปาเก็ตตี้ เราสร้าง **Finite State Machine (FSM)** ในฟังก์ชัน `zbash_expand_vars` ในไฟล์ [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 20 ถึง 56:

```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 20 - 56) */
    bool in_single = false;
    bool in_double = false;

    for (size_t i = 0; input[i] != '\0'; i++) {
        char c = input[i];

        /* 1. การจัดการ Backslash (\) */
        if (c == '\\' && !in_single && input[i + 1] != '\0') {
            char next = input[i + 1];
            if (in_double) {
                // กฎเหล็กของ POSIX ภายใน Double Quotes:
                // \ จะหลบอักขระได้เฉพาะ $, `, ", \, และ \n เท่านั้น!
                if (next == '$' || next == '`' || next == '"' || next == '\\' || next == '\n') {
                    i++;
                    c = next;
                } else {
                    // หากเป็นตัวอักษรอื่น เช่น \a, \b ให้เก็บตัว \ ไว้เหมือนเดิม!
                    if (len + 1 >= cap) { cap *= 2; buf = realloc(buf, cap); }
                    buf[len++] = '\\';
                    continue;
                }
            } else {
                // อยู่นอกเครื่องหมายคำพูด: \ สามารถหลบอักขระได้ทุกตัว
                i++;
                c = next;
            }
            if (len + 1 >= cap) { cap *= 2; buf = realloc(buf, cap); }
            buf[len++] = c;
            continue;
        }

        /* 2. การสลับสถานะ Single Quote */
        if (c == '\'' && !in_double) {
            in_single = !in_single;
            continue; // ไม่นำเครื่องหมาย ' เข้าไปในข้อความผลลัพธ์ (Quote Removal)
        }

        /* 3. การสลับสถานะ Double Quote */
        if (c == '"' && !in_single) {
            in_double = !in_double;
            continue; // ไม่นำเครื่องหมาย " เข้าไปในข้อความผลลัพธ์ (Quote Removal)
        }
```

### เจาะลึกกฎข้อยกเว้นของ POSIX (บรรทัดที่ 29–37):
ลองดูความแตกต่างที่แยบยลตรงนี้:
* หากผู้ใช้พิมพ์ `echo "hello\world"` 
  - ตัว `\w` ไม่ได้อยู่ในกลุ่ม `$` หรือ `"` ดังนั้นตามมาตรฐาน POSIX ตัว `\` จะต้อง **ไม่ถูกลบทิ้ง** ผลลัพธ์ต้องแสดงเป็น `hello\world`
* แต่หากผู้ใช้พิมพ์ `echo "hello\"world"`
  - ตัว `\"` ตรงกับกฎในบรรทัดที่ 30 ระบบจะข้าม `\` และนำเฉพาะตัว `"` เข้าไปในสตริง ผลลัพธ์จึงเป็น `hello"world`
* บั๊กในเชลล์มือสมัครเล่นส่วนใหญ่มักตกม้าตายตรงนี้ เพราะเผลอไปลบ `\` ทิ้งทั้งหมด!

---

## 🔄 3. การจัดการ Backticks (\`cmd\`) ซ้อนใน Quotes

อีกหนึ่งความท้าทายคือการรองรับคำสั่งยุคคลาสสิกอย่าง Backticks เช่น `echo "Today is `date`"`  
ดูโค้ดจริงใน [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 58 ถึง 83:

```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 58 - 83) */
        /* Backtick command substitution `cmd` */
        if (c == '`' && !in_single) {
            size_t start = i + 1;
            size_t j = start;
            while (input[j] != '\0' && input[j] != '`') {
                if (input[j] == '\\' && input[j + 1] != '\0') j += 2;
                else j++;
            }
            if (input[j] == '`') {
                size_t clen = j - start;
                char sub_cmd[1024];
                if (clen >= sizeof(sub_cmd)) clen = sizeof(sub_cmd) - 1;
                strncpy(sub_cmd, input + start, clen);
                sub_cmd[clen] = '\0';

                // รันคำสั่งใน Subshell ผ่าน zbash_subshell_exec
                char *out = zbash_subshell_exec(state, sub_cmd);
                if (out) {
                    size_t olen = strlen(out);
                    while (len + olen >= cap) { cap = (cap + olen) * 2; buf = realloc(buf, cap); }
                    memcpy(buf + len, out, olen);
                    len += olen;
                    free(out);
                }
                i = j;
                continue;
            }
        }
```

### กลไกการประมวลผล:
1. **`!in_single`:** หาก Backtick อยู่ใน Single Quote (`'`...'`'`) ระบบจะมองเป็นตัวอักษรธรรมดา แต่หากอยู่ใน Double Quotes ระบบจะรันคำสั่งทันทีตามมาตรฐาน POSIX
2. **การกระโดดข้าม Backslash หลบคำสั่ง (`j += 2`):** หากภายใน Backticks มีการใช้ `\` เช่น `` `echo \` ` `` ลูปสแกนในบรรทัดที่ 62 จะไม่นับเป็นจุดสิ้นสุดคำสั่ง
3. **Zero-Leak Memory Expansion:** เมื่อคำสั่งใน Subshell คืนค่าผลลัพธ์ออกมา บัฟเฟอร์ `buf` จะทำการขยายขนาดหน่วยความจำแบบ Exponential Growth (`cap = (cap + olen) * 2`) เพื่อรองรับผลลัพธ์ขนาดใหญ่ได้โดยไม่เกิด Buffer Overflow

---

## 🧼 4. ศิลปะของ Quote Removal (การปลดเปลื้องเครื่องหมายคำพูด)

ผู้เริ่มต้นเขียน Shell มักสับสนว่า **"ทำไมเมื่อสั่ง `echo "hello"` บน Terminal ถึงไม่มีเครื่องหมาย `"` ปรากฏบนหน้าจอ?"**

คำตอบอยู่ที่บรรทัดที่ 49 และ 54 ใน [`src/parser/lexer.c`](src/parser/lexer.c):
```c
if (c == '\'' && !in_double) {
    in_single = !in_single;
    continue; // 👈 จุดสำคัญ: ข้ามตัวอักษรนี้ไป ไม่นำเข้าไปในบัฟเฟอร์ผลลัพธ์!
}
```
* กระบวนการนี้ในสเปกของ POSIX เรียกว่า **"Quote Removal"**
* เครื่องหมาย `'` และ `"` มีหน้าที่เพียงแค่เป็น "ธงส่งสัญญาณ (State Flags)" บอกตัว Lexer ว่าให้เปิดหรือปิดโหมด Literal เท่านั้น เมื่อมันทำหน้าที่กำกับขอบเขตคำเสร็จสิ้นแล้ว ตัวสัญลักษณ์เครื่องหมายคำพูดจะถูกปลดเปลื้องออกก่อนที่จะส่งไปยังคำสั่งจริง

---

## 🏁 สรุปบทเรียนที่ 5

1. **เครื่องหมายคำพูดไม่ใช่เรื่องง่าย:** การแยกแยะพฤติกรรมระหว่าง Single Quote, Double Quote, และ Backslash ต้องใช้ Finite State Machine ที่แม่นยำ
2. **เคารพในข้อยกเว้นของ POSIX:** Backslash ภายใน Double Quote มีกฎเกณฑ์พิเศษที่ต้องหลบเฉพาะอักขระควบคุมบางตัวเท่านั้น
3. **Quote Removal คือขั้นตอนสุดท้าย:** ปลดเครื่องหมายคำพูดทิ้งไปหลังจากที่มันกำหนดขอบเขตคำและควบคุมการขยายตัวแปรเรียบร้อยแล้ว

ใน **บทที่ 6** เราจะนำ Token ที่ตัดคำและปลดเครื่องหมายคำพูดอย่างหมดจดนี้ มาประกอบร่างสร้างเป็น **"Abstract Syntax Tree (AST) ขนาดกะทัดรัด (64-byte Node)"** เพื่อเตรียมส่งให้ Execution Engine รันต่อไปครับ! 🚀📖
