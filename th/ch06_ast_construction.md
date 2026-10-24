# บทที่ 6: การสร้าง Abstract Syntax Tree (AST) ขนาดกะทัดรัด (64-byte Node)
> **"Crafting a Compact Abstract Syntax Tree (AST) with 64-byte Nodes"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch06_ast_construction.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Conceptual integrity is the most important consideration in system design."**  
> — **Fred Brooks** (The Mythical Man-Month)  
> *คำแปล:* "ความเป็นเอกภาพทางแนวคิด (Conceptual Integrity) คือสิ่งสำคัญที่สุดในการออกแบบระบบขนาดใหญ่"
> 
> **"Keep it decoupled. The less your components know about each other, the more resilient your system becomes."**  
> — **David Parnas** (บิดาแห่ง Modular Programming)  
> *คำแปล:* "จงรักษาความเป็นอิสระระหว่างชิ้นส่วน ยิ่งคอมโพเนนต์รู้จักกันน้อยเท่าใด ระบบของคุณก็จะยิ่งยืดหยุ่นและแข็งแกร่งขึ้นเท่านั้น"

---

## 🧭 1. ทำไม AST ของเชลล์จึงไม่ควรเป็น Pointer Graph ที่ซับซ้อนเกินจำเป็น?

ในโลกของคอมไพเลอร์ทั่วไป เช่น Clang, GCC หรือตัวแปลภาษาอย่าง Python โครงสร้าง **Abstract Syntax Tree (AST)** มักถูกสร้างขึ้นเป็นโครงข่ายพอยน์เตอร์ขนาดมหึมา (Dynamic Pointer Graph) ที่เต็มไปด้วยโหนดประเภทต่างๆ นับร้อยชนิด  
โหนดเหล่านั้นต้องใช้การเรียก `malloc` และ `free` นับหมื่นครั้งต่อวินาที ซึ่งสร้างปัญหาใหญ่ 2 ประการสำหรับเชลล์:
1. **Memory Fragmentation:** หน่วยความจำแตกกระจายเป็นชิ้นเล็กชิ้นน้อยใน Heap
2. **CPU Cache Miss:** ซีพียูต้องกระโดดไล่ตามพอยน์เตอร์ในแรม (Pointer Chasing) ซึ่งช้ากว่าการอ่านข้อมูลที่วางเรียงกันในแคช

แต่เมื่อเราวิเคราะห์ไวยากรณ์ของ Unix Shell อย่างถ่องแท้ เราจะพบสัจธรรมที่สวยงามว่า **คำสั่งของเชลล์มีลำดับขั้นเพียง 3 ระดับที่ชัดเจนและแน่นอน**:

```
[ ลำดับขั้นที่ 1: Command Lists ] ──> เชื่อมด้วยเครื่องหมายตรรกะ (&&, ||, ;)
       │
       ▼
[ ลำดับขั้นที่ 2: Pipeline ]       ──> เชื่อมคำสั่งด้วยท่อไปป์ (|)
       │
       ▼
[ ลำดับขั้นที่ 3: Simple Command ] ──> ประกอบด้วย argv[] + Redirections (<, >, 2>&1)
```

ด้วยโครงสร้างที่ชัดเจนนี้ เราจึงสามารถออกแบบ AST ของ `zbash` ให้เป็น **Flat Memory Node ขนาดกะทัดรัด** ที่ไม่ต้องพึ่งพาพอยน์เตอร์ซับซ้อนเลยแม้แต่ตัวเดียว!

---

## 🏗️ 2. สถาปัตยกรรม 64-byte Flat Nodes ใน `include/zbash.h`

ลองเปิดดูไฟล์ [`include/zbash.h`](include/zbash.h) บรรทัดที่ 73 ถึง 97:

```c
/* ไฟล์: include/zbash.h (บรรทัดที่ 73 - 97) */
typedef struct {
    char *argv[ZBASH_MAX_ARGS];   /* อาร์เรย์เก็บ Arguments (ZBASH_MAX_ARGS = 64) */
    int argc;                     /* จำนวนพารามิเตอร์ */
    char *in_file;                /* < file */
    char *out_file;               /* > file */
    bool append_out;              /* >> file */
    char *err_file;               /* 2> file */
    bool append_err;              /* 2>> file */
    bool err_to_out;              /* 2>&1 */
    bool out_to_err;              /* >&2 หรือ 1>&2 */
} zbash_cmd_t;

typedef enum {
    CONN_NONE,
    CONN_AND,     /* && */
    CONN_OR,      /* || */
    CONN_SEMI     /* ; */
} zbash_conn_t;

typedef struct {
    zbash_cmd_t cmds[ZBASH_MAX_PIPELINE];  /* อาร์เรย์ของคำสั่งในไปป์ (สูงสุด 16 ตัว) */
    int cmd_count;                         /* จำนวนคำสั่งในท่อไปป์ */
    zbash_conn_t next_conn;                /* ตัวเชื่อมต่อไปยัง Pipeline ถัดไป */
} zbash_pipeline_t;
```

### ความฉลาดของการออกแบบ Flat Array:
* **ไม่มี Pointer Tree:** ในโครงสร้าง `zbash_pipeline_t` คำสั่งทั้งหมด 16 ตัว (`cmds[16]`) ถูกจองพื้นที่เรียงติดกันในหน่วยความจำแบบ **Contiguous Memory Block**
* **Cache-Locality สูงสุด:** เมื่อซีพียู Apple Silicon โหลดข้อมูล Pipeline ขึ้นมา ข้อมูลคำสั่งทั้งหมดจะถูกดึงเข้าสู่ **L1 Data Cache** พร้อมกันทันที ทำให้การประมวลผลท่อไปป์เร็วกว่าเชลล์แบบเดิมอย่างก้าวกระโดด

---

## ⚙️ 3. การแยกแยะ Redirections และ Grouping ใน `src/parser/lexer.c`

การแปลงสตริงคำสั่งเดี่ยวให้กลายเป็นโหนด `zbash_cmd_t` เกิดขึ้นในฟังก์ชัน `zbash_parse_simple_cmd` ในไฟล์ [`src/parser/lexer.c`](src/parser/lexer.c) บรรทัดที่ 660 ถึง 750:

### 1) การตรวจจับ Compound Blocks `{ ... }` และ `( ... )`
```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 669 - 696) */
    /* ตรวจจับ Compound Brace { ... } */
    if (cmd_str[i] == '{' && (i + 1 == len || isspace((unsigned char)cmd_str[i + 1]))) {
        size_t b_start = i + 1;
        int depth = 1;
        bool in_s = false, in_d = false;
        size_t j = b_start;
        while (j < len && depth > 0) {
            char c = cmd_str[j];
            if (c == '\'' && !in_d) in_s = !in_s;
            else if (c == '"' && !in_s) in_d = !in_d;
            else if (c == '{' && !in_s && !in_d) depth++;
            else if (c == '}' && !in_s && !in_d) {
                depth--;
                if (depth == 0) break;
            }
            j++;
        }
        if (depth == 0) {
            cmd->argv[0] = strdup("{");
            // แยกเนื้อหาคำสั่งภายในวงเล็บปีกกาออกมาเป็น Argument ที่ 1
            size_t inner_len = j - b_start;
            char *inner = malloc(inner_len + 1);
            strncpy(inner, cmd_str + b_start, inner_len);
            inner[inner_len] = '\0';
            cmd->argv[1] = inner;
            cmd->argv[2] = NULL;
            cmd->argc = 2;
            i = j + 1;
        }
    }
```
* **Bracket Depth Tracking (บรรทัดที่ 671–684):** สังเกตว่าระบบนับระดับความลึก (`depth`) ของวงเล็บปีกกา `{ ... }` พร้อมทั้งตรวจเช็คสถานะ Quotes (`in_s`, `in_d`) ไปด้วย เพื่อป้องกันไม่ให้วงเล็บที่อยู่ในเครื่องหมายคำพูด (เช่น `echo "{}"`) ถูกนับเป็น Compound Block ผิดพลาด!

### 2) การสแกน Redirections แบบ Zero-Latency
```c
/* ไฟล์: src/parser/lexer.c (บรรทัดที่ 735 - 750) */
        /* ตรวจสอบการผสาน Error เข้ากับ Output (2>&1) */
        if (cmd_str[i] == '2' && i + 3 < len && cmd_str[i + 1] == '>' && cmd_str[i + 2] == '&' && cmd_str[i + 3] == '1') {
            cmd->err_to_out = true;
            i += 4;
            continue;
        }
        /* ตรวจสอบการส่ง Output ไปที่ Error (1>&2 หรือ >&2) */
        if (cmd_str[i] == '1' && i + 3 < len && cmd_str[i + 1] == '>' && cmd_str[i + 2] == '&' && cmd_str[i + 3] == '2') {
            cmd->out_to_err = true;
            i += 4;
            continue;
        }
        if (cmd_str[i] == '>' && i + 2 < len && cmd_str[i + 1] == '&' && cmd_str[i + 2] == '2') {
            cmd->out_to_err = true;
            i += 3;
            continue;
        }
```
* **Token Lookahead (บรรทัดที่ 736–749):** ตัว Parser ใช้วิธีมองไปข้างหน้า (Lookahead) เพียง 3–4 ไบต์ เพื่อตรวจจับโทเค็นยอดนิยมอย่าง `2>&1` และ `>&2` ทันที แล้วเปิดบูลีนแฟล็กในโหนด `cmd` พร้อมข้าม Index ไปข้างหน้าทันที ทำให้การ Parse มีความเร็วระดับ $O(N)$ เสมอ

---

## 🏁 สรุปบทเรียนที่ 6

1. **อย่าสร้างความซับซ้อนเกินจำเป็น:** เชลล์ไม่ต้องการ AST ที่เป็นต้นไม้ยักษ์ แต่ต้องการโครงสร้างแถวลำดับที่แบนราบ (Flat Structs)
2. **การจัดกลุ่มคำสั่งมีผลต่อประสิทธิภาพ:** การรวม `zbash_cmd_t` เป็นอาร์เรย์ใน `zbash_pipeline_t` ช่วยให้หน่วยความจำอยู่ในแคช L1 และตัด Overhead ของการจองหน่วยความจำแบบไดนามิกออกไปทั้งหมด
3. **การแกะวงเล็บที่รัดกุม:** การนับความลึกของวงเล็บ `{` และ `(` ร่วมกับการตรวจ Quotes ช่วยให้ระบบรองรับสคริปต์ที่ซับซ้อนได้อย่างแม่นยำ

ใน **บทที่ 7** เราจะก้าวเข้าสู่ **"ศิลปะการขยายตัวแปร: `${VAR:-def}`, `${#arr[@]}`, `${VAR:offset:len}`"** เพื่อดูว่าเราแกะสตริงและคำนวณความยาวอาร์เรย์ในระดับภาษา C ได้อย่างไรครับ! 🚀📖
