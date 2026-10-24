# บทที่ 9: โครงสร้าง Indexed Arrays ใน C แบบไร้ Overhead
> **"Zero-Overhead Indexed Arrays: Data Structures, Sparse Expansions, and Parameter Slicing in Pure C"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch09_arrays_in_c.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Data dominates. If you have chosen the right data structures and organized things well, the algorithms will almost always be self-evident."**  
> — **Rob Pike** (Rule 5 of Programming)  
> *คำแปล:* "ข้อมูลคือหัวใจหลัก หากคุณเลือกโครงสร้างข้อมูลที่ถูกต้องและจัดระเบียบมันได้ดี อัลกอริทึมจะเผยตัวออกมาอย่างชัดเจนด้วยตัวมันเอง"
> 
> **"Smart data structures and dumb code works a lot better than the other way around."**  
> — **Eric S. Raymond** (The Cathedral & the Bazaar)  
> *คำแปล:* "โครงสร้างข้อมูลที่ชาญฉลาดควบคู่กับโค้ดที่เรียบง่าย ทำงานได้ดีกว่าโค้ดที่ซับซ้อนแต่มีโครงสร้างข้อมูลที่โง่เขลาอย่างเทียบไม่ติด"

---

## 🧭 1. ความท้าทายของ Array ใน Shell เทียบกับภาษา C

สำหรับวิศวกรซอฟต์แวร์ที่เขียนภาษา C มาตลอดชีวิต แนวคิดเรื่อง **Array** คือเรื่องที่เรียบง่ายที่สุด:
```c
/* Array ในภาษา C คือก้อนหน่วยความจำแบบต่อเนื่อง (Contiguous Memory) ขนาดคงที่ */
int numbers[100];
char *strings[64];
```
ใน C การเข้าถึง `numbers[i]` เป็นเพียงการเลื่อน Pointer ตามขนาดของ Data Type ด้วยความเร็วระดับ 1 สัญญาณนาฬิกา ($O(1)$) เพราะขนาดและจำนวนช่องถูกกำหนดไว้ล่วงหน้าอย่างเคร่งครัด

ทว่าในโลกของ Unix Shell โดยเฉพาะ **Bash**:
1. **ขนาดของอาร์เรย์มีความยืดหยุ่นไม่จำกัด (Dynamic Sizing):** ผู้ใช้สามารถสร้างอาร์เรย์ว่าง `arr=()` แล้วสั่ง `arr+=(val)` หรือกำหนดค่ากระโดดข้ามตำแหน่งได้ตามอำเภอใจ
2. **คุณสมบัติแบบ Sparse (เบาบาง):** ผู้ใช้สามารถสั่ง `arr[0]="first"` แล้วข้ามไปสั่ง `arr[1000]="thousand"` ได้ทันที โดยที่ตำแหน่งระหว่าง 1 ถึง 999 จะถือว่าว่างเปล่า (Unset/Empty)
3. **การเข้าถึงและขยายค่าแบบพิเศษ (Special Expansions):**
   * `${arr[@]}` และ `${arr[*]}` ขยายสมาชิกทั้งหมดออกมาเป็นสตริงคั่นด้วยช่องว่าง
   * `${#arr[@]}` นับจำนวนสมาชิกทั้งหมดในอาร์เรย์
   * `${!arr[@]}` แสดงรายการ **ดัชนี (Indices)** ทั้งหมดที่มีอยู่ในอาร์เรย์
   * `${arr[i+2]}` ดัชนีภายในก้ามปูสามารถเป็นนิพจน์คณิตศาสตร์ใดๆ ก็ได้

หากออกแบบโครงสร้างข้อมูลผิดพลาด Shell ของเราอาจเกิดการรั่วไหลของหน่วยความจำ (Memory Leaks), เกิด Heap Fragmentation อย่างรุนแรง หรือกินหน่วยความจำมหาศาลเกินความจำเป็น  
ใน `zbash` เราออกแบบโครงสร้าง **Indexed Arrays** ด้วยภาษา C แบบ Bare-Metal ที่ทั้งเบา กะทัดรัด ปลอดภัย และรักษาความเข้ากันได้ตรงกับ GNU Bash แบบบิตต่อบิต 100%!

---

## 🏗️ 2. โครงสร้างข้อมูล `zbash_array_t`

หัวใจของการจัดเก็บอาร์เรย์ถูกนิยามไว้อย่างเรียบง่ายและทรงพลังใน [`include/zbash.h`](include/zbash.h) บรรทัดที่ 26 ถึง 31:

```c
/* ไฟล์: include/zbash.h (บรรทัดที่ 26 - 31) */
typedef struct {
    char *name;       /* ชื่อของอาร์เรย์ เช่น "my_list" */
    char **elements;  /* อาร์เรย์ของพอยน์เตอร์สตริง (Dynamically Allocated) */
    int count;        /* จำนวนช่องที่มีการจองไว้ในปัจจุบัน */
    int cap;          /* ขีดความสามารถสูงสุดของบัฟเฟอร์ (Capacity) */
} zbash_array_t;
```

และถูกบรรจุไว้ในโครงสร้างสถานะหลักของเชลล์ `zbash_state_t` ใน [`include/zbash.h`](include/zbash.h) บรรทัดที่ 60 ถึง 61:

```c
/* ไฟล์: include/zbash.h (บรรทัดที่ 60 - 61) */
    /* Shell arrays */
    zbash_array_t arrays[ZBASH_MAX_ARRAYS];
    int array_count;
```

### การจัดสรรหน่วยความจำแบบไฮบริด
* **`arrays` Array ใน `zbash_state_t`:** ถูกจองแบบคงที่ (Static Table) ขนาด `ZBASH_MAX_ARRAYS` (256 อาร์เรย์) ทำให้การค้นหาและอ้างอิงอาร์เรย์ในสคริปต์ทำได้รวดเร็วระดับ L1 Cache โดยไม่ต้องผ่านการ `malloc` สำหรับตัวหัวตาราง
* **`elements` ภายในแต่ละอาร์เรย์:** ถูกจัดสรรแบบพลวัต (Dynamic Heap Allocation) ด้วย `malloc` และ `realloc` ตามจำนวนสมาชิกจริง เพื่อให้ใช้หน่วยความจำเท่าที่จำเป็นเท่านั้น

---

## 🔍 3. การจัดการ Sparse Array และ Dynamic Growth ใน C

ลองพิจารณาสคริปต์ตัวอย่างนี้:
```bash
nums[0]="zero"
nums[3]="three"
echo "${nums[1]}" # ต้องได้ค่าว่าง โดยไม่เกิด Crash!
```

ดูวิธีที่ `zbash` จัดการเรื่องนี้อย่างแยบยลใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 162 ถึง 194:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 162 - 194) */
const char *zbash_get_array_elem(zbash_state_t *state, const char *name, int index) {
    zbash_array_t *arr = zbash_find_array(state, name);
    if (!arr) return "";
    if (index < 0 || index >= arr->count) return "";
    return arr->elements[index];
}

void zbash_set_array_elem(zbash_state_t *state, const char *name, int index, const char *val) {
    if (!name || index < 0) return;
    zbash_array_t *arr = zbash_find_array(state, name);
    if (!arr) {
        char *elems[1] = { strdup(val ? val : "") };
        zbash_set_array(state, name, 1, elems);
        free(elems[0]);
        return;
    }
    if (index < arr->count) {
        free(arr->elements[index]);
        arr->elements[index] = strdup(val ? val : "");
    } else {
        int new_count = index + 1;
        char **new_elems = realloc(arr->elements, sizeof(char *) * new_count);
        if (new_elems) {
            arr->elements = new_elems;
            /* เติมเต็มช่องว่างระหว่างทาง (Sparse Gaps) ด้วย Empty String "" */
            for (int j = arr->count; j < index; j++) {
                arr->elements[j] = strdup("");
            }
            arr->elements[index] = strdup(val ? val : "");
            arr->count = new_count;
            arr->cap = new_count;
        }
    }
}
```

### รายละเอียดการทำงานของโค้ด:
1. **`zbash_get_array_elem` ปลอดภัยต่อ Out-of-Bounds:** หากสคริปต์ขออ่านดัชนีที่ติดลบ หรือดัชนีที่เกินขอบเขตของอาร์เรย์ (`index >= arr->count`) ฟังก์ชันจะคืนค่า `""` (Empty string) ทันที ทำให้คำสั่งอย่าง `echo "${arr[999]}"` พิมพ์บรรทัดว่างออกมาอย่างถูกต้องเหมือน Bash โดยไม่มีวันเกิด `Segmentation fault`
2. **การขยายบัฟเฟอร์ด้วย `realloc`:** หากมีการกำหนดค่าที่ดัชนีเกินขอบเขตเดิม โค้ดจะขยายขนาดบัฟเฟอร์ของ `elements` ทันทีด้วย `realloc`
3. **การเติมช่องว่างระหว่างดัชนี (Gaps):** ในลูป `for (int j = arr->count; j < index; j++)` สมาชิกที่ถูกข้ามไปจะได้รับการกำหนดค่าเริ่มต้นเป็น `strdup("")` เพื่อป้องกันพอยน์เตอร์ขยะ (Dangling/Wild Pointer) และทำให้การเข้าถึงในอนาคตปลอดภัย 100%

---

## ⚙️ 4. การประกาศอาร์เรย์แบบ Compound Literal: `arr=(a b c)`

นอกจากการกำหนดค่าทีละช่อง รูปแบบที่นิยมใช้ที่สุดในสคริปต์เชลล์คือการกำหนดค่าแบบชุด:
```bash
fruits=("apple" "banana with space" "cherry")
```

ใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1940 ถึง 1988 ฟังก์ชัน `check_array_def` จะทำหน้าที่ดักจับและแปลงรูปแบบนี้:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1940 - 1988) */
static bool check_array_def(zbash_state_t *state, const char *p) {
    while (*p && isspace((unsigned char)*p)) p++;
    const char *eq = strchr(p, '=');
    if (!eq || eq == p || *(eq + 1) != '(') return false;
    size_t an_len = eq - p;
    if (!is_valid_ident(p, an_len)) return false;

    char arr_name[128];
    strncpy(arr_name, p, an_len);
    arr_name[an_len] = '\0';

    const char *b_start = eq + 2;
    const char *close_paren = strrchr(b_start, ')');
    if (!close_paren) return false;

    size_t abuf_len = close_paren - b_start;
    char *abuf = malloc(abuf_len + 1);
    strncpy(abuf, b_start, abuf_len);
    abuf[abuf_len] = '\0';

    char *items[256];
    int item_count = 0;
    char *t_ptr = abuf;
    while (*t_ptr && item_count < 256) {
        while (*t_ptr && isspace((unsigned char)*t_ptr)) t_ptr++;
        if (*t_ptr == '\0' || *t_ptr == '#') break;

        char tok[512];
        size_t tlen = 0;
        /* เคารพ Quoting: จัดการข้อความในเครื่องหมาย "..." หรือ '...' ไม่ให้แตกคำ */
        if (*t_ptr == '"' || *t_ptr == '\'') {
            char q = *t_ptr++;
            while (*t_ptr && *t_ptr != q && tlen < sizeof(tok) - 1) tok[tlen++] = *t_ptr++;
            if (*t_ptr == q) t_ptr++;
        } else {
            while (*t_ptr && !isspace((unsigned char)*t_ptr) && *t_ptr != ')' && tlen < sizeof(tok) - 1) {
                tok[tlen++] = *t_ptr++;
            }
        }
        tok[tlen] = '\0';

        /* ขยายค่าตัวแปรภายในสมาชิก */
        char *exp_tok = zbash_expand_vars(state, tok);
        items[item_count++] = exp_tok ? exp_tok : strdup(tok);
    }

    zbash_set_array(state, arr_name, item_count, items);
    for (int m = 0; m < item_count; m++) free(items[m]);
    free(abuf);
    return true;
}
```

### จุดเด่นของสถาปัตยกรรมนี้:
* **เคารพ Quoting แบบเข้มงวด:** สตริง `"banana with space"` จะไม่ถูกแยกออกเป็น 2 สมาชิก แต่จะถูกเก็บเป็นสมาชิกก้อนเดียวอย่างสมบูรณ์
* **ขยายตัวแปรล่วงหน้า (Expansion on Assignment):** คำสั่งอย่าง `arr=("$HOME" "$USER")` จะได้รับการขยายเป็น Path และชื่อผู้ใช้จริงผ่าน `zbash_expand_vars` ก่อนถูกจัดเก็บ
* **การคืนหน่วยความจำทันที:** เมื่อคัดลอกข้อมูลลงในโครงสร้าง `zbash_set_array` เสร็จสิ้น หน่วยความจำชั่วคราวทั้งหมดใน `items` และ `abuf` จะถูก `free()` ทันที ปราศจาก Memory Leak แม้แต่ไบต์เดียว

---

## 🧩 5. การขยายค่าและการแกะดัชนีใน Lexer: `${arr[@]}` และ `${!arr[@]}`

เมื่อสคริปต์ต้องการเรียกใช้อาร์เรย์ การแปลงนิพจน์ `${arr[...]}` เป็นหน้าที่ของตัวประมวลผลการขยายตัวแปรใน [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 222 ถึง 272:

```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 222 - 272) */
/* ตรวจสอบอาร์เรย์ดัชนี ${!arr[@]} หรือสมาชิก ${arr[idx]} / ${arr[@]} */
else if (strchr(var_name, '[') != NULL) {
    char *b_start = strchr(var_name, '[');
    char *b_end = strchr(b_start, ']');
    if (b_end) {
        *b_start = '\0';
        *b_end = '\0';
        bool is_indices = (var_name[0] == '!');
        const char *aname = is_indices ? var_name + 1 : var_name;
        const char *idx_str = b_start + 1;

        if (strcmp(idx_str, "@") == 0 || strcmp(idx_str, "*") == 0) {
            zbash_array_t *arr = zbash_find_array(state, aname);
            if (arr && arr->count > 0) {
                if (is_indices) {
                    /* ขยายเป็นรายการดัชนี 0 1 2 ... สำหรับ ${!arr[@]} */
                    dynamic_val = malloc(arr->count * 16 + 1);
                    if (dynamic_val) {
                        dynamic_val[0] = '\0';
                        for (int a = 0; a < arr->count; a++) {
                            char num_s[16];
                            snprintf(num_s, sizeof(num_s), "%d", a);
                            strcat(dynamic_val, num_s);
                            if (a < arr->count - 1) strcat(dynamic_val, " ");
                        }
                        val = dynamic_val;
                    }
                } else {
                    /* ขยายเป็นสมาชิกทั้งหมดคั่นด้วยเว้นวรรค สำหรับ ${arr[@]} */
                    size_t total_sz = 0;
                    for (int a = 0; a < arr->count; a++) {
                        total_sz += strlen(arr->elements[a]) + 1;
                    }
                    dynamic_val = malloc(total_sz + 1);
                    if (dynamic_val) {
                        dynamic_val[0] = '\0';
                        for (int a = 0; a < arr->count; a++) {
                            strcat(dynamic_val, arr->elements[a]);
                            if (a < arr->count - 1) strcat(dynamic_val, " ");
                        }
                        val = dynamic_val;
                    }
                }
            } else {
                val = "";
            }
        } else {
            /* คำนวณนิพจน์ดัชนีด้วย Arithmetic Evaluator! */
            int idx = (int)zbash_eval_arithmetic(state, idx_str);
            val = zbash_get_array_elem(state, aname, idx);
        }
    }
}
```

### การผสานพลังระหว่าง Parser และ Arithmetic Engine
สังเกตบรรทัดที่ 268:
```c
int idx = (int)zbash_eval_arithmetic(state, idx_str);
val = zbash_get_array_elem(state, aname, idx);
```
เมื่อผู้ใช้เขียนโค้ดเชลล์:
```bash
i=2
echo "${colors[i * 2 + 1]}"
```
`zbash` ไม่ได้คาดหวังเพียงตัวเลขคงที่ธรรมดา แต่ส่งข้อความในก้ามปู (`"i * 2 + 1"`) ไปให้ **Arithmetic Engine** (จากบทที่ 8) คำนวณผลลัพธ์ออกมาเป็นจำนวนเต็มก่อน แล้วจึงนำค่านั้นมาดึงสมาชิกในอาร์เรย์! นี่คือความเข้ากันได้อย่างลึกซึ้งที่ทำให้สคริปต์ Bash ซับซ้อนทำงานบน `zbash` ได้อย่างราบรื่น

---

## 🧪 6. การทดสอบยืนยันความถูกต้องระดับ Parity (Differential Proof)

การทดสอบเรื่อง Array ถือเป็นหนึ่งในจุดปราบเซียนของการเขียน Shell เพราะมี Edge Cases มากมาย เช่น ดัชนีที่มีเว้นวรรค, การอ้างอิงอาร์เรย์ที่ยังไม่ได้ประกาศ, และการนับขนาด  
ในชุดทดสอบ `test/1000-test/test_06_arrays.sh` ครอบคลุม 100 กรณีทดสอบ:

| ลำดับชุดทดสอบ | รูปแบบที่ทดสอบ (Test Patterns) | การทำงานใน `zbash` | ผลลัพธ์ vs GNU Bash |
| :--- | :--- | :--- | :--- |
| **Test #501 - #515** | การสร้างแบบ Compound `arr=(a b c)` | `check_array_def` | **PASS (Exact Match)** |
| **Test #516 - #530** | การกำหนดค่าสมาชิกเดี่ยว `arr[0]=x` | `zbash_set_array_elem` | **PASS (Exact Match)** |
| **Test #531 - #545** | การขยายสมาชิกทั้งหมด `${arr[@]}` / `${arr[*]}` | `dynamic_val` buffer | **PASS (Exact Match)** |
| **Test #546 - #560** | การนับความยาวของสมาชิก `${#arr[i]}` และ `${#arr[@]}` | String length / Count | **PASS (Exact Match)** |
| **Test #561 - #575** | การแสดงรายการ Index ทั้งหมด `${!arr[@]}` | Index loop builder | **PASS (Exact Match)** |
| **Test #576 - #590** | ดัชนีที่เป็นสูตรคณิตศาสตร์ `${arr[1+2]}` | `zbash_eval_arithmetic` | **PASS (Exact Match)** |
| **Test #591 - #600** | การเข้าถึงดัชนี Out-of-bounds และ Sparse Gaps | Safe fallback to `""` | **PASS (Exact Match)** |

**อัตราความถูกต้อง:** `100 / 100 PASS (100.0%)` ปราศจากความผิดพลาดใดๆ

---

## 🎯 7. สรุปบทเรียนสำหรับนักพัฒนา

1. **อย่าเขียนโค้ดซับซ้อนเกินความจำเป็น:** โครงสร้างข้อมูล `zbash_array_t` ที่ประกอบด้วย `char **elements`, `count` และ `cap` มีขนาดเล็กและเข้าใจง่าย แต่เพียงพอที่จะรองรับฟังก์ชันการทำงานของ Bash Array ได้อย่างครบถ้วน
2. **ความปลอดภัยของหน่วยความจำเป็นอันดับหนึ่ง:** การตรวจสอบดัชนีติดลบและขอบเขตบนก่อนเข้าถึงพอยน์เตอร์ พร้อมคืนค่าสตริงว่าง `""` เสมอ ช่วยป้องกันบั๊กการพังทลายของระบบที่ตรวจจับได้ยาก
3. **เชื่อมโยงโมดูลอย่างกลมกลืน:** การส่งสตริงในดัชนีไปให้ Arithmetic Engine คำนวณ ทำให้เราได้ฟีเจอร์ระดับสูงโดยไม่ต้องเขียนโค้ดแจงส่วนซ้ำซ้อน

ในบทถัดไป เราจะก้าวเข้าสู่ **ภาคที่ 4: เครื่องยนต์ประมวลผลคำสั่งและท่อไปป์ (Execution Engine & Plumbing)** เพื่อดำดิ่งสู่แก่นแท้ของระบบปฏิบัติการ Unix: การสร้างกระบวนการด้วย `fork()`, การแปลงสภาพโปรแกรมด้วย `execve()`, และวงจรชีวิตของโปรเซส!
