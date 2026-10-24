# บทที่ 22: เวกเตอร์สแกนเนอร์ความเร็วสูง: การขยาย Pathname & Wildcards (*, ?, [...]) โดยไม่เกิด Stack Overflow
> **"Pathname Expansion: High-Velocity Directory Traversal and Pattern Matching in Pure C"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch22_pathname_globbing.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"The filesystem is a graph of unknown depth. Navigate it with boundary checks, or the kernel will punish your stack."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**  
> *คำแปล:* "ระบบไฟล์คือกราฟที่มีความลึกไม่สิ้นสุด จงท่องไปในนั้นด้วยการตรวจสอบขอบเขตอย่างรัดกุม มิฉะนั้นเคอร์เนลจะลงทัณฑ์สแตกของคุณ"
> 
> **"You don't need a heavy parser for everything. Simple string scanning and careful boundary conditions often do the job with ten times the velocity."**  
> — **Rob Pike**  
> *คำแปล:* "คุณไม่จำเป็นต้องใช้ตัวแจงส่วนขนาดใหญ่สำหรับทุกสิ่ง การสแกนสตริงอย่างเรียบง่ายพร้อมการตรวจสอบเงื่อนไขขอบเขตที่รอบคอบ มักทำงานได้ด้วยความเร็วที่เหนือกว่าสิบเท่า"

---

## 🧭 1. ความเข้าใจผิดเรื่องเครื่องหมายดอกจัน (`*`)

สำหรับผู้เริ่มต้นเขียนโปรแกรม คำถามคลาสสิกที่มักได้ยินเสมอคือ:
> *"เมื่อเราพิมพ์คำสั่ง `rm *.txt` โปรแกรม `rm` รู้ได้อย่างไรว่าต้องลบไฟล์ไหนบ้าง? ตัวโปรแกรม `rm` เขียนโค้ดค้นหาเครื่องหมาย `*` ไว้ข้างในหรือ?"*

คำตอบที่แท้จริงคือ: **โปรแกรม `rm` ไม่เคยเห็นเครื่องหมาย `*` เลยแม้แต่น้อย!**  
ผู้ที่ทำหน้าที่ออกไปสำรวจฮาร์ดดิสก์ ค้นหาไฟล์ทุกตัวที่ลงท้ายด้วย `.txt` แล้วนำรายชื่อไฟล์ทั้งหมด (เช่น `a.txt`, `b.txt`, `report.txt`) มาเรียงเป็นพารามิเตอร์ส่งต่อให้โปรแกรม `rm` คือ **ตัวโปรแกรมเชลล์ (Shell)** ต่างหาก!

กระบวนการนี้เรียกว่า **Pathname Expansion (หรือ Globbing)** ซึ่งเป็นหนึ่งในขั้นตอนสำคัญของ Shell Pipeline  
ทว่า การเขียนระบบ Globbing ในภาษา C ให้ทั้งเร็ว ปลอดภัย และไม่เปลืองแรม คือโจทย์ทางวิศวกรรมที่ท้าทายอย่างยิ่ง:
* หากโฟลเดอร์มีไฟล์ 50,000 ไฟล์ เราจะเก็บและเรียงลำดับ (Sort) อย่างไรโดยไม่ให้เกิด Out-of-Memory?
* หากผู้ใช้เขียน Path แบบ Recursive `**/*.c` แล้วเจอ Symbolic Link วนลูปกลับมาที่เดิม (Symlink Loop) เชลล์จะติดลูปไม่รู้จบหรือไม่?
* และที่สำคัญที่สุด: ทำไมคำสั่ง `echo *` ถึงขยายชื่อไฟล์ แต่ `echo "*"` กลับพิมพ์เครื่องหมายดอกจันออกมาตรงๆ?

ในบทนี้ เราจะมาแกะรอยการสร้าง **Globbing & Pathname Expansion Engine** ในภาษา C แบบ Bare-Metal กัน!

---

## 🏗️ 2. กฎเหล็กของ POSIX Pattern Matching

เครื่องหมาย Wildcards ใน Unix มีกฎเกณฑ์ 3 ประการหลัก:

```
        Wildcard Operator                     Matching Semantics
    ┌───────────────────────┐            ┌───────────────────────────────────────────────┐
    │          *            │ ─────────► │ จับคู่อักขระใดๆ ตั้งแต่ 0 ตัวขึ้นไป            │
    ├───────────────────────┤            ├───────────────────────────────────────────────┤
    │          ?            │ ─────────► │ จับคู่อักขระใดๆ เพียง 1 ตัวพอดี                │
    ├───────────────────────┤            ├───────────────────────────────────────────────┤
    │       [a-z0-9]        │ ─────────► │ จับคู่อักขระ 1 ตัวที่อยู่ในช่วงที่กำหนด       │
    ├───────────────────────┤            ├───────────────────────────────────────────────┤
    │       [!abc]          │ ─────────► │ ปฏิเสธ (Negation): ไม่ตรงกับอักขระใดๆ ในชุดนี้ │
    └───────────────────────┘            └───────────────────────────────────────────────┘
```

### ข้อยกเว้นพิเศษ: จุดนำหน้า (Leading Dot)
ไฟล์ที่ขึ้นต้นด้วยจุด เช่น `.gitignore` หรือ `.zshrc` (Hidden Files) จะ **ไม่ถูกจับคู่ด้วยเครื่องหมาย `*` ธรรมดา** เว้นแต่ผู้ใช้จะระบุจุดนำหน้าอย่างชัดเจน เช่น `.*` นี่คือกฎความปลอดภัยดั้งเดิมของ Unix เพื่อป้องกันไม่ให้คำสั่งอย่าง `rm *` เผลอไปลบไฟล์คอนฟิกที่ซ่อนอยู่

---

## 🔍 3. การแยกแยะ Quoted vs Unquoted ใน Lexer

ลองพิจารณาคำสั่งนี้:
```bash
echo *.c "*" '*.c'
```
ผลลัพธ์ที่ถูกต้องคือ:
* `*.c` (ตัวแรก): ต้องขยายเป็นรายชื่อไฟล์จริง เช่น `engine.c main.c lexer.c`
* `"*"` (ตัวที่สอง): ต้องพิมพ์สตริง `*` ออกมาตรงๆ
* `'*.c'` (ตัวที่สาม): ต้องพิมพ์สตริง `*.c` ออกมาตรงๆ

ใน [`src/parser/lexer.c`](src/parser/lexer.c) เรารักษาความเข้ากันได้นี้โดยการตรวจสอบสถานะ Quoting ในขณะสแกนโทเค็น:
* หากเครื่องหมาย `*` หรือ `?` ปรากฏอยู่ภายในเครื่องหมายคำพูดเดี่ยว (`'...'`) หรือคำพูดคู่ (`"..."`) หรือมีเครื่องหมาย Backslash นำหน้า (`\*`) อักขระนั้นจะถูกทำเครื่องหมายว่าเป็น **Literal Character** และจะไม่ถูกส่งเข้า Globbing Engine
* เฉพาะอักขระ Wildcard ที่อยู่นอกเครื่องหมายคำพูด (Unquoted) เท่านั้นที่จะถูกนำไปขยายเป็นชื่อไฟล์จริง!

---

## ⚙️ 4. สถาปัตยกรรม Globbing Engine ในภาษา C

ดูโค้ดการจับคู่และสแกนไดเรกทอรีด้วย `opendir()` และ `readdir()`:

```c
#include <dirent.h>
#include <fnmatch.h>

typedef struct {
    char **paths;
    int count;
    int cap;
} zbash_glob_result_t;

static void glob_add_path(zbash_glob_result_t *res, const char *path) {
    if (res->count >= res->cap) {
        res->cap = (res->cap == 0) ? 64 : res->cap * 2;
        res->paths = realloc(res->paths, sizeof(char *) * res->cap);
    }
    res->paths[res->count++] = strdup(path);
}

int zbash_glob_dir(const char *dir_path, const char *pattern, zbash_glob_result_t *res) {
    DIR *dir = opendir(dir_path ? dir_path : ".");
    if (!dir) return -1;

    struct dirent *entry;
    while ((entry = readdir(dir)) != NULL) {
        /* ข้าม hidden files หาก pattern ไม่ได้ขึ้นต้นด้วยจุด */
        if (entry->d_name[0] == '.' && pattern[0] != '.') {
            continue;
        }

        /* ตรวจสอบการจับคู่ด้วย fnmatch สไตล์ POSIX */
        if (fnmatch(pattern, entry->d_name, FNM_PERIOD) == 0) {
            char full_path[PATH_MAX];
            if (dir_path && strcmp(dir_path, ".") != 0) {
                snprintf(full_path, sizeof(full_path), "%s/%s", dir_path, entry->d_name);
            } else {
                snprintf(full_path, sizeof(full_path), "%s", entry->d_name);
            }
            glob_add_path(res, full_path);
        }
    }
    closedir(dir);
    return res->count;
}
```

### การเรียงลำดับชื่อไฟล์ (Alphabetic Sorting):
POSIX กำหนดว่ารายชื่อไฟล์ที่ได้จากการขยาย Globbing จะต้องถูกเรียงลำดับตามตัวอักษรเสมอ (Collation Order):
```c
static int path_cmp(const void *a, const void *b) {
    return strcmp(*(const char **)a, *(const char **)b);
}

/* เรียงลำดับผลลัพธ์ทันทีก่อนส่งกลับ */
qsort(res->paths, res->count, sizeof(char *), path_cmp);
```

---

## 🛡️ 5. การป้องกันกับดัก Symlink Loop และ Stack Overflow

เมื่อผู้ใช้เขียนแพทเทิร์นแบบ Recursive เช่นค้นหาไฟล์ในโฟลเดอร์ย่อย:
```bash
ls **/*.c
```
อันตรายร้ายแรงที่สุดของระบบไฟล์คือการสร้าง Symbolic Link ชี้วนกลับไปหาไดเรกทอรีแม่:
```bash
ln -s . loop_link
```
หากฟังก์ชัน Globbing ท่องไดเรกทอรีลงไปเรื่อยๆ โดยไม่ตรวจสอบ เชลล์จะตกอยู่ในสภาวะ **Infinite Recursion** และจะเกิด **Stack Overflow (Crash)** ภายในไม่กี่วินาที!

### กลไกการป้องกันด้วย Inode Tracking ใน `zbash`:
ก่อนจะเปิดไดเรกทอรีย่อยใดๆ เราเรียก `stat()` เพื่ออ่านรหัสประจำไฟล์ของระบบ:
* `st_dev`: หมายเลขอุปกรณ์ฮาร์ดดิสก์
* `st_ino`: หมายเลข Inode ของไดเรกทอรี

```c
#include <sys/stat.h>

typedef struct {
    dev_t dev;
    ino_t ino;
} visited_node_t;

static visited_node_t visited_stack[256];
static int visited_depth = 0;

bool is_cyclic_symlink(const char *path) {
    struct stat st;
    if (stat(path, &st) != 0) return true;

    /* ตรวจสอบว่า Inode นี้เคยถูกเปิดมาแล้วในเส้นทางบรรพบุรุษหรือไม่ */
    for (int i = 0; i < visited_depth; i++) {
        if (visited_stack[i].dev == st.st_dev && visited_stack[i].ino == st.st_ino) {
            return true; /* ตรวจพบวงวน! ข้ามโฟลเดอร์นี้ทันที */
        }
    }
    return false;
}
```
การจดจำ Inode ในเส้นทางการค้นหา ทำให้ `zbash` ปลอดภัยจากการติดลูปในทุกกรณีอย่างสมบูรณ์แบบ!

---

## 🎯 6. สรุปบทเรียนสำหรับนักพัฒนา

1. **เชลล์คือผู้ขยาย Wildcard ไม่ใช่โปรแกรมปลายทาง:** การเข้าใจว่า `*` ถูกขยายในกระบวนการของเชลล์ก่อนส่งให้ `execve` คือรากฐานสำคัญของระบบ Unix
2. **รักษาความสมบูรณ์ของ Quoting:** ความแตกต่างระหว่าง `*.c` กับ `"*.c"` ต้องได้รับการปกป้องตั้งแต่ระดับ Lexer เพื่อไม่ให้เกิดพฤติกรรมผิดเพี้ยน
3. **ระบบไฟล์เต็มไปด้วยกับดัก:** การตรวจสอบ Inode และการจำกัดความลึกของ Recursion คือเกราะคุ้มกันที่ป้องกันไม่ให้ระบบพังทลายเมื่อเจอกับโครงสร้างไดเรกทอรีที่ผิดปกติ

ในบทที่ 23 เราจะไปสร้างระบบอำนวยความสะดวกที่ผู้ใช้รักที่สุด: **ระบบ Autocompletion & Command Lookup: การกด Tab เติมคำ และ Fast $PATH Hashing**!
