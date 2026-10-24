# บทที่ 7: ศิลปะการขยายตัวแปร: ${VAR:-def}, ${#arr[@]}, ${VAR:offset:len}
> **"The Art of Parameter Expansion: ${VAR:-def}, ${#arr[@]}, and ${VAR:offset:len}"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch07_parameter_expansion.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Data dominates. If you've chosen the right data structures and organized things well, the algorithms will almost always be self-evident."**  
> — **Rob Pike** (Bell Labs)  
> *คำแปล:* "โครงสร้างข้อมูลคือหัวใจ หากคุณเลือกโครงสร้างข้อมูลที่ถูกต้องและเป็นระเบียบ อัลกอริทึมจะปรากฏขึ้นมาเองอย่างชัดเจนแทบทุกครั้ง"
> 
> **"A crown jewel is not forged in comfort; it is crystallized through thousands of rigorous tests until it reflects pure truth."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**

---

## 🧭 1. ลำดับขั้นการขยายตัวแปรในมาตรฐาน POSIX

หนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ Bash คือ **Parameter Expansion (การขยายตัวแปร)**  
ผู้ใช้ไม่ได้เพียงแค่สั่ง `$NAME` แล้วจบ แต่เชลล์สามารถ:
* กำหนดค่าเริ่มต้นเมื่อตัวแปรว่าง: `${VAR:-default}`
* ตั้งค่าและบันทึกกลับลง Environment: `${VAR:=default}`
* วัดความยาวของสตริงหรือจำนวนสมาชิกอาร์เรย์: `${#VAR}` และ `${#arr[@]}`
* ตัดแต่งสตริงแบบคำนวณตำแหน่ง (Substring Slicing): `${VAR:offset:length}`

ความท้าทายระดับมหากาพย์คือ: **เราจะเขียนตรรกะทั้งหมดนี้ในภาษา C อย่างไร โดยที่ตัวไบนารียังคงกะทัดรัด (86 KB) และไม่ทำให้หน่วยความจำรั่วไหล (Memory Leaks)?**

คำตอบทั้งหมดอยู่ในฟังก์ชัน `zbash_expand_vars` ในไฟล์ [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 183 ถึง 380!

---

## 🛠️ 2. การกำหนดค่าเริ่มต้น: `${VAR:-default}` และ `${VAR:=default}`

ลองมาดูโค้ดจริงใน [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 301 ถึง 326:

```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 301 - 326) */
                    } else if (op_char == ':') {
                        const char *env_v = getenv(vname_id);
                        if (rest[1] == '-') {
                            /* ${VAR:-default} */
                            const char *def_val = rest + 2;
                            if (env_v && *env_v) {
                                val = env_v;
                            } else {
                                // หากค่า default มีการอ้างอิงตัวแปรอื่น ให้ขยายตัวแปรซ้อนได้!
                                if (strchr(def_val, '$')) {
                                    dynamic_val = zbash_expand_vars(state, def_val);
                                    val = dynamic_val;
                                } else {
                                    val = def_val;
                                }
                            }
                        } else if (rest[1] == '=') {
                            /* ${VAR:=default} */
                            const char *def_val = rest + 2;
                            if (!env_v || *env_v == '\0') {
                                char *exp_def = strchr(def_val, '$') ? zbash_expand_vars(state, def_val) : strdup(def_val);
                                setenv(vname_id, exp_def, 1); // บันทึกลง OS Environment ทันที!
                                dynamic_val = exp_def;
                                val = exp_def;
                            } else {
                                val = env_v;
                            }
                        }
```

### รายละเอียดที่สร้างความแตกต่าง:
1. **Nested Variable Resolution (บรรทัดที่ 309):**  
   หากผู้ใช้เขียน `${VAR:-$HOME/backup}` ตัวแปร `$HOME` ที่ซ้อนอยู่ข้างในจะต้องถูกคลี่คลายออกมาด้วย ไม่ใช่ส่ง `$HOME` กลับไปดื้อๆ เราจึงเรียก `zbash_expand_vars` แบบ Recursive ทันที
2. **Side Effect ของ `${VAR:=}` (บรรทัดที่ 321):**  
   ความต่างระหว่าง `:-` กับ `:=` คือตัว `:=` ต้องบันทึกค่ากลับลงระบบปฏิบัติการผ่าน `setenv(vname_id, exp_def, 1)` เพื่อให้คำสั่งถัดไปมองเห็นค่านั้นด้วย

---

## ✂️ 3. การตัดแต่งสตริงแบบคำนวณทางคณิตศาสตร์: `${VAR:offset:length}`

ฟีเจอร์นี้ยากเป็นพิเศษเพราะ **Offset และ Length สามารถเป็นนิพจน์คณิตศาสตร์หรือตัวแปรอื่นได้** เช่น `${VAR:1+1:2*2}` หรือ `${VAR:$off:$len}`  
ดูโค้ดจริงใน [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 341 ถึง 366:

```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 341 - 366) */
                        } else {
                            /* Substring slicing: ${VAR:offset} หรือ ${VAR:offset:length} */
                            if (!env_v) env_v = "";
                            int vlen = (int)strlen(env_v);
                            char *sep2 = strchr(rest + 1, ':');
                            int offset = 0, slen = -1;
                            if (sep2) {
                                *sep2 = '\0';
                                // คำนวณ offset และ length ผ่านเครื่องยนต์คณิตศาสตร์ zbash!
                                offset = (int)zbash_eval_arithmetic(state, rest + 1);
                                slen = (int)zbash_eval_arithmetic(state, sep2 + 1);
                            } else {
                                offset = (int)zbash_eval_arithmetic(state, rest + 1);
                            }

                            // การป้องกัน Boundary ปลอดภัย 100% ไม่เกิด Buffer Overflow
                            if (offset < 0) offset = vlen + offset;
                            if (offset < 0) offset = 0;
                            if (offset > vlen) offset = vlen;
                            if (slen < 0 || offset + slen > vlen) slen = vlen - offset;
                            if (slen < 0) slen = 0;

                            dynamic_val = malloc(slen + 1);
                            if (dynamic_val) {
                                strncpy(dynamic_val, env_v + offset, slen);
                                dynamic_val[slen] = '\0';
                                val = dynamic_val;
                            } else {
                                val = "";
                            }
                        }
```

### การคิดค้นที่แยบยล:
* **การคำนวณผ่าน Arithmetic Evaluator (บรรทัดที่ 348–349):**  
  แทนที่จะใช้ `atoi()` ธรรมดา เราส่งสตริงของ offset และ length ไปให้ `zbash_eval_arithmetic(state, ...)` ประมวลผล ทำให้ `${VAR:start+1:len*2}` ทำงานได้อย่างถูกต้องไร้ที่ติ
* **การจัดการ Offset ติดลบ (บรรทัดที่ 353):**  
  หาก offset มีค่าติดลบ เช่น `${VAR: -2}` ในสเปกของ Bash จะหมายถึงการนับถอยหลังจากท้ายสตริง โค้ดในบรรทัด 353 ทำการแปลงเป็น `offset = vlen + offset` โดยอัตโนมัติ

---

## 📏 4. การวัดความยาวและจำนวนสมาชิก: `${#VAR}` และ `${#arr[@]}`

ดูบรรทัดที่ 198 ถึง 220 ใน [`src/parser/lexer.c`](src/parser/lexer.c):

```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 198 - 220) */
                /* ตรวจสอบเครื่องหมาย # นำหน้า */
                else if (var_name[0] == '#') {
                    // กรณีที่ 1: นับจำนวนสมาชิกอาร์เรย์ทั้งหมด ${#arr[@]} หรือ ${#arr[*]}
                    if (strstr(var_name, "[@]") != NULL || strstr(var_name, "[*]") != NULL) {
                        char *b_start = strchr(var_name, '[');
                        *b_start = '\0';
                        zbash_array_t *arr = zbash_find_array(state, var_name + 1);
                        snprintf(num_buf, sizeof(num_buf), "%d", arr ? arr->count : 0);
                        val = num_buf;
                    } 
                    // กรณีที่ 2: นับความยาวของสมาชิกอาร์เรย์ตัวใดตัวหนึ่ง ${#arr[idx]}
                    else if (strchr(var_name, '[') != NULL) {
                        char *b_start = strchr(var_name, '[');
                        char *b_end = strchr(b_start, ']');
                        if (b_end) {
                            *b_start = '\0'; *b_end = '\0';
                            int idx = atoi(b_start + 1);
                            const char *elem = zbash_get_array_elem(state, var_name + 1, idx);
                            snprintf(num_buf, sizeof(num_buf), "%zu", elem ? strlen(elem) : 0);
                            val = num_buf;
                        }
                    } 
                    // กรณีที่ 3: นับความยาวสตริงของตัวแปรทั่วไป ${#VAR}
                    else {
                        const char *env_v = getenv(var_name + 1);
                        snprintf(num_buf, sizeof(num_buf), "%zu", env_v ? strlen(env_v) : 0);
                        val = num_buf;
                    }
                }
```

* **Zero-Allocation Stack Buffer:** สังเกตว่าเราใช้ `num_buf[64]` บน Stack เพื่อแปลงตัวเลขจำนวนนับเป็นสตริงด้วย `snprintf()` ทำให้ไม่ต้องเสียเวลาเรียก `malloc` แม้แต่น้อย

---

## 🏁 สรุปบทเรียนที่ 7

1. **Parameter Expansion คือพลังที่แท้จริงของ Shell:** การเข้าใจพฤติกรรมของ `:-`, `:=`, และ `${#...}` ช่วยให้เราออกแบบตัวแปลงสตริงที่สมบูรณ์แบบ
2. **การผสานพลังกับ Arithmetic Engine:** การใช้ตัวประเมินคณิตศาสตร์มาคำนวณ Offset ทำให้ `${VAR:off:len}` รองรับนิพจน์ขั้นสูงได้ทุกรูปแบบ
3. **การป้องกันหน่วยความจำล้น:** การตรวจสอบ Boundary อย่างรัดกุมช่วยป้องกันบั๊กประเภท Segfault และ Out-of-bounds Read ได้ 100%

ใน **บทที่ 8** เราจะมาเปิดดูสมองกลที่เป็นเครื่องคิดเลขในตัวของเชลล์: **"เครื่องคิดเลขในตัว: สร้างตัวประเมินคณิตศาสตร์ `$(( a + b * c ))`"** พร้อมรองรับตัวดำเนินการระดับบิตและ Pre/Post-increment กันครับ! 🚀📖
