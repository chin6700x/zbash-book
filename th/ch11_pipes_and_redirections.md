# บทที่ 11: ท่อไปป์และการเปลี่ยนทิศทาง: pipe(), dup2(), และ done < file
> **"Unix Plumbing: Pipes, Redirections, File Descriptors, and the Trailing Loop Trap"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch11_pipes_and_redirections.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"We should have some ways of connecting programs like garden hose--screw in another segment when it becomes necessary to massage data in another way."**  
> — **Doug McIlroy** (ผู้คิดค้นระบบ Unix Pipes ในปี 1973)  
> *คำแปล:* "เราควรจะมีวิธีเชื่อมต่อโปรแกรมต่างๆ เข้าด้วยกันเหมือนสายยางรดน้ำต้นไม้—ต่อข้อต่อท่อนใหม่เข้าไปเมื่อไหร่ก็ตามที่เราต้องการดัดแปลงข้อมูลในรูปแบบอื่น"
> 
> **"The UNIX system provides a convenient and uniform way for programs to communicate with each other: pipes."**  
> — **Brian W. Kernighan**  
> *คำแปล:* "ระบบ UNIX ได้มอบวิธีการที่สะดวกและเป็นอันหนึ่งอันเดียวกันเพื่อให้โปรแกรมต่างๆ สื่อสารกันได้ นั่นคือ 'ท่อไปป์'"

---

## 🧭 1. สายยางรดน้ำต้นไม้แห่งโลก Unix (The Garden Hose)

หากเปรียบระบบปฏิบัติการเป็นโรงงานอุตสาหกรรม **Unix Pipes** ก็คือสายพานลำเลียงที่ทรงอิทธิพลที่สุดในประวัติศาสตร์คอมพิวเตอร์  
ก่อนปี 1973 โปรแกรมส่วนใหญ่ทำงานแบบโดดเดี่ยว: อ่านไฟล์จากดิสก์ เขียนผลลัพธ์ลงเทปแม่เหล็ก แล้วจบงาน  
Doug McIlroy ได้เสนอแนวคิดปฏิวัติวงการ: ทำไมเราไม่สร้างท่อส่งข้อมูลระหว่างหน่วยความจำของกระบวนการสองตัวโดยตรง เพื่อให้ผลลัพธ์จากโปรแกรมหนึ่งไหลเข้าเป็นอินพุตของอีกโปรแกรมหนึ่งได้ทันทีโดยไม่ต้องเขียนลงดิสก์?

ใน Bash และสคริปต์ทั่วไป การสั่ง:
```bash
cat access.log | grep "404" | cut -d' ' -f1 | sort | uniq -c > summary.txt
```
ดูเหมือนเรื่องง่ายดาย แต่เบื้องหลังคำสั่งบรรทัดนี้ เคอร์เนลต้องเปิดตัวอธิบายไฟล์ (File Descriptors), ทำการสลับช่องทางรับส่งข้อมูลด้วย `dup2()`, สร้างท่อแบบทางเดียวด้วย `pipe()`, และจัดการการปิดท่ออย่างระมัดระวัง

ในบทนี้ เราจะมาแกะรอยระบบ **Plumbing & Redirection Engine** ใน `zbash` และเปิดเผยเบื้องหลังการแก้ **"บั๊ก Trailing Loop Redirection (`done < file`)"** ที่เคยเป็นปริศนาในการทดสอบ Parity!

---

## 🏗️ 2. ตารางตัวอธิบายไฟล์ (File Descriptors Table)

ในระดับระบบปฏิบัติการ Unix มาตรฐาน ทุกโปรเซสจะมีตาราง File Descriptor เริ่มต้น 3 ตัวเสมอ:
* **`0` (`STDIN_FILENO`):** ช่องทางรับข้อมูลมาตรฐาน (แป้นพิมพ์ หรือไฟล์อินพุต)
* **`1` (`STDOUT_FILENO`):** ช่องทางแสดงผลลัพธ์มาตรฐาน (หน้าจอ Terminal หรือไฟล์เอาต์พุต)
* **`2` (`STDERR_FILENO`):** ช่องทางแจ้งข้อผิดพลาดมาตรฐาน (หน้าจอ Terminal)

```
        Standard Streams                 Redirection Operators
    ┌───────────────────────┐            ┌───────────────────────┐
    │  0: STDIN_FILENO      │ ◄───────── │ < file,  <<< "string" │
    ├───────────────────────┤            ├───────────────────────┤
    │  1: STDOUT_FILENO     │ ─────────► │ > file,  >> file      │
    ├───────────────────────┤            ├───────────────────────┤
    │  2: STDERR_FILENO     │ ─────────► │ 2> file, 2>> file     │
    └───────────────────────┘            ├───────────────────────┤
                                         │ 2>&1  (ERR ➔ OUT)     │
                                         │ >&2   (OUT ➔ ERR)     │
                                         └───────────────────────┘
```

---

## 🔍 3. เจาะลึกโค้ด `apply_redirections`: สลับท่อด้วย System Call

ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 716 ถึง 758:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 716 - 758) */
static int apply_redirections(zbash_cmd_t *cmd) {
    /* 1. จัดการ Redirection ขาเข้า: < input.txt */
    if (cmd->in_file) {
        int fd = open(cmd->in_file, O_RDONLY);
        if (fd < 0) {
            fprintf(stderr, "zbash: %s: %s\n", cmd->in_file, strerror(errno));
            return -1;
        }
        dup2(fd, STDIN_FILENO);
        close(fd);
    }

    /* 2. จัดการ Redirection ขาออก: > out.txt หรือ >> out.txt */
    if (cmd->out_file) {
        int flags = O_WRONLY | O_CREAT | (cmd->append_out ? O_APPEND : O_TRUNC);
        int fd = open(cmd->out_file, flags, 0666);
        if (fd < 0) {
            fprintf(stderr, "zbash: %s: %s\n", cmd->out_file, strerror(errno));
            return -1;
        }
        dup2(fd, STDOUT_FILENO);
        close(fd);
    }

    /* 3. จัดการ Redirection แจ้งข้อผิดพลาด: 2> err.log หรือ 2>> err.log */
    if (cmd->err_file) {
        int flags = O_WRONLY | O_CREAT | (cmd->append_err ? O_APPEND : O_TRUNC);
        int fd = open(cmd->err_file, flags, 0666);
        if (fd < 0) {
            fprintf(stderr, "zbash: %s: %s\n", cmd->err_file, strerror(errno));
            return -1;
        }
        dup2(fd, STDERR_FILENO);
        close(fd);
    }

    /* 4. รวมข้อผิดพลาดเข้ากับเอาต์พุตปกติ: 2>&1 */
    if (cmd->err_to_out) {
        dup2(STDOUT_FILENO, STDERR_FILENO);
    }

    /* 5. ส่งเอาต์พุตปกติไปยังช่องทางข้อผิดพลาด: >&2 */
    if (cmd->out_to_err) {
        dup2(STDERR_FILENO, STDOUT_FILENO);
    }

    return 0;
}
```

### ทำไมต้องสั่ง `close(fd)` หลัง `dup2` ทันที?
คำสั่ง `dup2(fd, STDOUT_FILENO)` ทำการโคลนตัวอธิบายไฟล์ `fd` ไปทับช่องทางหมายเลข 1 (`STDOUT`) หากเราไม่สั่ง `close(fd)` ตัวระบบจะมี File Descriptor เปิดค้างอยู่ 2 ตัวที่ชี้ไปยังไฟล์เดียวกัน ซึ่งเป็นการสิ้นเปลืองตารางทรัพยากรของเคอร์เนล และอาจทำให้โปรแกรมชนขีดจำกัด `ulimit -n` (Too many open files) เมื่อสคริปต์ทำงานยาวนาน

---

## ⚡ 4. การจัดการ Here-Strings (`<<<`) และ Here-Documents (`<<`)

ฟีเจอร์ที่นักพัฒนาชื่นชอบใน Bash คือการส่งข้อความตรงเข้า STDIN ผ่าน Here-String:
```bash
grep "needle" <<< "haystack with needle inside"
```
หรือการส่งข้อความหลายบรรทัดผ่าน Here-Document:
```bash
cat << 'EOF' > config.json
{
  "status": "active"
}
EOF
```

ใน `zbash` เราจัดการสตริงเหล่านี้โดยการสร้าง Anonymous Inode ชั่วคราวใน `/tmp/zbash_hs_XXXXXX` และเมื่อคำสั่งทำงานเสร็จสิ้น ฟังก์ชัน `free_cmd_strings` ใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 760 ถึง 776 จะทำลายไฟล์ทิ้งทันที:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 767 - 773) */
    if (cmd->in_file) {
        if (strncmp(cmd->in_file, "/tmp/zbash_hs_", 14) == 0) {
            unlink(cmd->in_file); /* ลบไฟล์ออกจากระบบไฟล์ทันที! */
        }
        free(cmd->in_file);
        cmd->in_file = NULL;
    }
```
การสั่ง `unlink()` ทันทีช่วยรับประกันว่าดิสก์ของเครื่องจะไม่เต็มไปด้วยไฟล์ขยะ แม้สคริปต์จะมีการส่ง Here-Strings เป็นแสนครั้งต่อวัน

---

## 🪤 5. กับดักคลาสสิก: Trailing Loop Redirection (`done < file`)

ในการพัฒนาช่วงทดสอบ 1,000 ข้อ เราพบกรณีทดสอบสุดแสบใน `test_07_pipelines_redirections.sh` (ข้อที่ #77):
```bash
# อ่านข้อมูลทีละบรรทัดจากไฟล์จนจบ
while read line; do
    echo "Processing: $line"
done < input_data.txt

# คำสั่งถัดไปที่อ่านจากคีย์บอร์ดหรือสคริปต์หลัก
read next_cmd
```
### ปัญหาที่เกิดขึ้นในตอนแรก:
เมื่อลูป `while` จบลง คำสั่ง `read next_cmd` ด้านล่างกลับอ่านค่าไม่ได้และคืนค่า `EOF` ทันที!  
**สาเหตุ:** เพราะการเปลี่ยนทิศทาง `< input_data.txt` ไปทับ `STDIN_FILENO (0)` ของเชลล์หลัก ทำให้เมื่อไฟล์หมด เชลล์หลักก็สูญเสียช่องทางเชื่อมต่อกับอินพุตเดิมไปตลอดกาล!

### วิธีแก้ระดับตำนานใน `zbash`:
ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1446 ถึง 1465:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1446 - 1465) */
    zbash_cmd_t redir_cmd;
    memset(&redir_cmd, 0, sizeof(redir_cmd));
    const char *rp = p;
    while (*rp && isspace((unsigned char)*rp)) rp++;
    bool has_redir = false;
    int saved_in = -1, saved_out = -1, saved_err = -1;
    if (*rp) {
        zbash_parse_simple_cmd(state, rp, &redir_cmd);
        if (redir_cmd.in_file || redir_cmd.out_file || redir_cmd.err_file || 
            redir_cmd.err_to_out || redir_cmd.out_to_err) {
            has_redir = true;
            /* 1. จำ File Descriptors ดั้งเดิมของเชลล์ไว้ก่อนเข้าลูป! */
            saved_in = dup(STDIN_FILENO);
            saved_out = dup(STDOUT_FILENO);
            saved_err = dup(STDERR_FILENO);

            /* 2. สลับทางเดินข้อมูลเพื่อรันเฉพาะในลูป */
            if (apply_redirections(&redir_cmd) != 0) {
                close(saved_in); close(saved_out); close(saved_err);
                free_cmd_strings(&redir_cmd);
                return 1;
            }
        }
    }
```
และเมื่อลูปวนซ้ำจนเสร็จสิ้น (บรรทัดที่ 1505):
```c
    /* 3. กู้คืนทางเดินข้อมูลดั้งเดิมกลับมา 100% */
    if (has_redir) {
        dup2(saved_in, STDIN_FILENO);
        dup2(saved_out, STDOUT_FILENO);
        dup2(saved_err, STDERR_FILENO);
        close(saved_in);
        close(saved_out);
        close(saved_err);
        free_cmd_strings(&redir_cmd);
    }
```
การใช้กลไก **Save ➔ Redirect ➔ Execute ➔ Restore** ทำให้ลูปทำงานได้อย่างสมบูรณ์แบบ และอินพุตของระบบหลักกลับคืนสู่สภาวะปกติอย่างไร้รอยต่อ!

---

## 🧪 6. การทดสอบยืนยันความถูกต้องระดับ Parity (Differential Proof)

ในโมดูล `test/1000-test/test_07_pipelines_redirections.sh` มีชุดทดสอบเข้มข้น 100 ข้อ:

| หมวดการทดสอบ | รูปแบบคำสั่ง | พฤติกรรมที่ตรวจสอบ | ผลลัพธ์ vs GNU Bash |
| :--- | :--- | :--- | :--- |
| **Simple Redirections** | `echo foo > out.txt`, `cat < in.txt` | การสร้างและตัดขนาดไฟล์ | **PASS (100%)** |
| **Append Mode** | `echo bar >> out.txt` | การต่อท้ายโดยไม่ลบของเดิม | **PASS (100%)** |
| **Error Stream Merging** | `cmd 2>&1`, `cmd >&2` | ลำดับการสลับ File Descriptor | **PASS (100%)** |
| **Multi-Stage Pipes** | `seq 100 \| grep 5 \| wc -l` | การส่งต่อข้อมูลข้าม 3 โปรเซส | **PASS (100%)** |
| **Trailing Loop Redirection** | `while read; do ... done < data` | การจำและกู้คืนช่องทาง STDIN | **PASS (100%)** |
| **Here-String & Quotes** | `grep needle <<< "$VAR"` | การขยายตัวแปรใน Here-string | **PASS (100%)** |

**ผลลัพธ์รวมของโมดูล:** `100 / 100 PASS (100.0%)`

---

## 🎯 7. สรุปบทเรียนสำหรับนักพัฒนา

1. **`dup2()` คือเครื่องมือเปลี่ยนทิศทางจักรวาลของ Unix:** การเข้าใจความต่างระหว่าง `dup()` (คัดลอก Descriptor ไปยังช่องว่างแรก) กับ `dup2()` (แทนที่ช่องเป้าหมายอย่างแม่นยำ) คือหัวใจของการเขียน System Software
2. **อย่าลืม State Preservation:** เมื่อใดก็ตามที่คำสั่งต้องการเปลี่ยนทิศทาง I/O ของกระบวนการหลัก ต้องมีสติเสมอที่จะ `dup()` เก็บช่องเดิมไว้ แล้ว `dup2()` กู้คืนกลับมาเสมอ
3. **การปิดท่อที่ไม่ได้ใช้ไม่ใช่แค่เรื่องความสะอาด แต่คือเรื่องชีวิตและความตาย:** การลืมปิดปลายท่อเขียนในตัวส่ง จะทำให้ตัวรับรอข้อมูลไปตลอดกาลจนระบบค้างสนิท

ในบทที่ 12 เราจะไปสำรวจโครงสร้างการควบคุมอันทรงพลัง: **ลูปและเงื่อนไข: `while`, `for`, `until`, `if-then-else`, และ `case-esac`**!
