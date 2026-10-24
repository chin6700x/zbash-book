# บทที่ 20: สถาปัตยกรรม Readline แบบ Zero-Dependency: ปั้นระบบ Line Editing จากศูนย์ด้วย VT100
> **"Pure C Readline: Mastering Termios Raw Mode, ANSI Escape Codes, and History Ring Buffers"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch20_zero_dep_readline.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"The terminal is not a text box; it is an interactive streaming canvas of byte sequences."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**  
> *คำแปล:* "เทอร์มินัลไม่ใช่กล่องข้อความ แต่มันคือผืนผ้าใบอินเทอร์แอคทีฟที่ขับเคลื่อนด้วยสายธารของลำดับไบต์"
> 
> **"Controlling the raw interface between human keystrokes and machine execution without intermediaries is the highest form of software independence."**  
> — **Ken Thompson**  
> *คำแปล:* "การควบคุมส่วนต่อประสานดิบระหว่างการกดแป้นพิมพ์ของมนุษย์กับการประมวลผลของเครื่องโดยปราศจากคนกลาง คือรูปแบบสูงสุดของอิสรภาพทางซอฟต์แวร์"

---

## 🧭 1. ทำไมเราต้องปฏิเสธ GNU Readline?

ในโลกของ Unix เชลล์ส่วนใหญ่มักพึ่งพาไลบรารีชื่อดังอย่าง **GNU Readline** เพื่อให้ผู้ใช้สามารถ:
* กดลูกศรซ้าย/ขวาเพื่อเลื่อนเคอร์เซอร์ไปแก้ไขคำที่พิมพ์ผิด
* กดลูกศรขึ้น/ลงเพื่อย้อนดูประวัติคำสั่งเก่า (Command History)
* กด `Ctrl+A` เพื่อไปต้นประโยค และ `Ctrl+E` เพื่อไปท้ายประโยค

ทว่า สำหรับโครงการที่ยึดมั่นในปรัชญา **86 KB, Zero-Dependency, และ Pure Open Engineering** แบบ `zbash` การนำ GNU Readline เข้ามาใช้คือสิ่งที่เป็นไปไม่ได้:
1. **ขนาดที่มหึมา (Library Bloat):** ตัวไลบรารี Readline และ Termcap/Ncurses มีขนาดหลายร้อยกิโลไบต์ ซึ่งจะกลืนกินพื้นที่ไบนารีของเราจนทะลุหลักเมกะไบต์ทันที
2. **พันธนาการของ GPL License:** GNU Readline ถูกผูกมัดด้วยสัญญาอนุญาต GPLv3 ซึ่งขัดแย้งกับสัญญาสิทธิเสรีภาพแบบ Apache License 2.0 ของเรา
3. **การสูญเสียการควบคุมระดับไมโครวินาที:** การครอบไลบรารีภายนอกทำให้เราไม่สามารถปรับแต่งการรับส่งข้อมูลให้เข้ากับ Unified Memory ของ Apple Silicon ได้อย่างสมบูรณ์

คำถามทางวิศวกรรมที่แท้จริงคือ:  
> *"เราจะสร้างระบบ Line Editing และ History ในภาษา C ขึ้นมาเองจากศูนย์ได้อย่างไร โดยใช้เพียง System Calls พื้นฐานของ POSIX และลำดับรหัส ANSI/VT100?"*

ในบทนี้ เราจะมาผ่าสถาปัตยกรรมอินเทอร์แอคทีฟ REPL และสร้าง **Zero-Dependency Readline Engine** ขึ้นมาด้วยมือของเราเอง!

---

## 🏗️ 2. กายวิภาคของ Terminal Modes: Canonical vs Raw Mode

โดยปกติ เมื่อโปรแกรมในภาษา C เรียกใช้ `getchar()` หรือ `getline(&line, &cap, stdin)` ใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 2491:

```c
/* โหมด Canonical เริ่มต้นใน src/core/engine.c บรรทัดที่ 2496 - 2504 */
while (1) {
    fputs("zbash$ ", stdout);
    fflush(stdout);
    nread = getline(&line, &cap, stdin);
    if (nread == -1) break;
    ...
}
```

ตัว Terminal Driver ของระบบปฏิบัติการจะทำงานอยู่ใน **Canonical Mode (Cooked Mode)**:
* เคอร์เนลจะกักเก็บตัวอักษรทุกตัวไว้ในบัฟเฟอร์ของระบบ และจะยอมส่งให้โปรแกรมของเราก็ต่อเมื่อผู้ใช้ **กดปุ่ม Enter (`\n`)** เท่านั้น
* หากผู้ใช้กดปุ่มลูกศร เคอร์เนลจะไม่รู้ว่าจะเลื่อนเคอร์เซอร์ไปทางไหน แต่จะพิมพ์โค้ดแปลกๆ เช่น `^[[A` ออกมาบนหน้าจอแทน!

หากเราต้องการแทรกแซงการกดแป้นพิมพ์ทุกตัวแบบ Real-time เราต้องสลับ Terminal เข้าสู่ **Raw Mode**!

---

## 🔍 3. การเปลี่ยนเข้าสู่ Raw Mode ด้วย `termios`

ดูโค้ดการปรับแต่งโหมดของเทอร์มินัลด้วย Header มาตรฐาน `<termios.h>`:

```c
#include <termios.h>
#include <unistd.h>

static struct termios orig_termios;

void zbash_enable_raw_mode(void) {
    /* 1. จำการตั้งค่าดั้งเดิมของ Terminal ไว้เพื่อกู้คืนตอนออกจากโปรแกรม */
    tcgetattr(STDIN_FILENO, &orig_termios);

    struct termios raw = orig_termios;
    /* 2. ปิด ECHO (ไม่ให้ Terminal สะท้อนตัวอักษรออกมาเอง) */
    /*    ปิด ICANON (ปิด Line Buffering รับข้อมูลทีละ 1 ไบต์ทันที) */
    /*    ปิด ISIG (ไม่ให้ส่งสัญญาณ SIGINT/SIGTSTP อัตโนมัติเมื่อกด Ctrl+C/Z) */
    raw.c_lflag &= ~(ECHO | ICANON | IEXTEN | ISIG);

    /* 3. ปิดการแปลงอักขระพิเศษในฝั่งขาเข้า */
    raw.c_iflag &= ~(IXON | ICRNL);

    /* 4. กำหนดให้อ่านได้ทันทีโดยไม่ต้องรอเวลา (Timeout = 0) */
    raw.c_cc[VMIN] = 1;
    raw.c_cc[VTIME] = 0;

    /* 5. บังคับใช้การตั้งค่าใหม่กับ STDIN ทันที */
    tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}

void zbash_disable_raw_mode(void) {
    /* กู้คืนโหมดเดิมกลับมาเสมอเมื่อโปรแกรมจบ */
    tcsetattr(STDIN_FILENO, TCSAFLUSH, &orig_termios);
}
```

เมื่อ Terminal อยู่ใน Raw Mode:
* เมื่อนิ้วของผู้ใช้กดคีย์ `a` ฟังก์ชัน `read(STDIN_FILENO, &c, 1)` จะได้รับไบต์ `0x61` ทันทีในเสี้ยวนาโนวินาทีโดยไม่ต้องรอปุ่ม Enter
* หน้าจอจะไม่แสดงอะไรออกมา จนกว่าโปรแกรมของเราจะเป็นผู้สั่ง `write(STDOUT_FILENO, ...)` ด้วยตนเอง!

---

## 🧭 4. ถอดรหัส ANSI / VT100 Escape Sequences

เมื่อผู้ใช้กดปุ่มพิเศษ เช่น ปุ่มลูกศร หรือปุ่มฟังก์ชัน คีย์บอร์ดไม่ได้ส่งไบต์เดี่ยวๆ มาให้เรา แต่ส่ง **ลำดับรหัสหลบ (Escape Sequence)** ที่ขึ้นต้นด้วยไบต์ `\x1b` (Escape หรือ ASCII 27) ตามด้วย `[`:

| ปุ่มบนคีย์บอร์ด | ไบต์ที่ถูกส่งเข้ามา (Hex / ASCII) | ความหมายของ Sequence |
| :--- | :--- | :--- |
| **ลูกศรขึ้น (Up)** | `\x1b [ A` (`1B 5B 41`) | เลื่อนเคอร์เซอร์ขึ้น / ขอดูประวัติก่อนหน้า |
| **ลูกศรลง (Down)** | `\x1b [ B` (`1B 5B 42`) | เลื่อนเคอร์เซอร์ลง / ขอดูประวัติถัดไป |
| **ลูกศรขวา (Right)** | `\x1b [ C` (`1B 5B 43`) | เลื่อนเคอร์เซอร์ไปทางขวา 1 ตำแหน่ง |
| **ลูกศรซ้าย (Left)** | `\x1b [ D` (`1B 5B 44`) | เลื่อนเคอร์เซอร์ไปทางซ้าย 1 ตำแหน่ง |
| **Home** | `\x1b [ H` หรือ `\x1b [ 1 ~` | ย้ายเคอร์เซอร์ไปต้นบรรทัด |
| **End** | `\x1b [ F` หรือ `\x1b [ 4 ~` | ย้ายเคอร์เซอร์ไปท้ายบรรทัด |
| **Backspace** | `\x7f` หรือ `\x08` | ลบอักขระทางซ้ายมือ |

### โค้ดดักจับลำดับรหัสใน C:
```c
int zbash_read_key(void) {
    char c;
    if (read(STDIN_FILENO, &c, 1) != 1) return -1;

    if (c == '\x1b') {
        char seq[3];
        if (read(STDIN_FILENO, &seq[0], 1) != 1) return '\x1b';
        if (read(STDIN_FILENO, &seq[1], 1) != 1) return '\x1b';

        if (seq[0] == '[') {
            switch (seq[1]) {
                case 'A': return KEY_ARROW_UP;
                case 'B': return KEY_ARROW_DOWN;
                case 'C': return KEY_ARROW_RIGHT;
                case 'D': return KEY_ARROW_LEFT;
                case 'H': return KEY_HOME;
                case 'F': return KEY_END;
            }
        }
    }
    return c;
}
```

---

## 🎨 5. การจัดการ Line Buffer และการขยับเคอร์เซอร์บนหน้าจอ

การแก้ไขบรรทัดคำสั่งต้องทำงานร่วมกับ **บัฟเฟอร์ในหน่วยความจำ** สองตัว:
* `buf`: อาร์เรย์เก็บตัวอักษรจริง
* `pos`: ตำแหน่งเคอร์เซอร์ปัจจุบันภายในข้อความ ($0 \le \text{pos} \le \text{len}$)

```
             Buffer: [ 'e', 'c', 'h', 'o', ' ', 'f', 'o', 'o' ]
                                     ▲
                                    pos = 4 (เคอร์เซอร์อยู่ตรงช่องว่าง)
```

### การแทรกตัวอักษรกลางประโยค:
หากผู้ใช้กดปุ่ม `x` ขณะที่เคอร์เซอร์อยู่ตรงกลาง:
1. ใช้ `memmove()` ดันตัวอักษรตั้งแต่ `pos` ไปทางขวา 1 ช่อง
2. ใส่ตัวอักษรใหม่ลงที่ `buf[pos]`
3. สั่ง Terminal ล้างหน้าจอไปข้างหน้าด้วยคำสั่ง `\x1b[K` (Clear to end of line)
4. สตรีมตัวอักษรที่เหลือออกไป แล้วดึงเคอร์เซอร์กลับมาตำแหน่งเดิมด้วย `\x1b[%dD`

---

## 📚 6. ระบบ Command History Ring Buffer ในหน่วยความจำ

เพื่อความประหยัดหน่วยความจำระดับสูงสุด เราใช้โครงสร้าง **Circular Ring Buffer** เก็บประวัติคำสั่ง 1,000 รายการล่าสุด:

```c
#define ZBASH_HIST_SIZE 1024

typedef struct {
    char *entries[ZBASH_HIST_SIZE];
    int head;       /* ตำแหน่งสำหรับบันทึกรายการถัดไป */
    int count;      /* จำนวนคำสั่งที่มีในประวัติ */
    int cursor;     /* ตำแหน่งที่กำลังเรียกดูย้อนหลัง */
} zbash_history_t;

void zbash_hist_add(zbash_history_t *h, const char *line) {
    if (!line || !*line) return;
    /* ป้องกันการบันทึกคำสั่งซ้ำซ้อนติดกัน */
    if (h->count > 0 && strcmp(h->entries[(h->head - 1 + ZBASH_HIST_SIZE) % ZBASH_HIST_SIZE], line) == 0) {
        return;
    }
    if (h->entries[h->head]) free(h->entries[h->head]);
    h->entries[h->head] = strdup(line);
    h->head = (h->head + 1) % ZBASH_HIST_SIZE;
    if (h->count < ZBASH_HIST_SIZE) h->count++;
    h->cursor = h->head;
}
```

เมื่อผู้ใช้กด **ลูกศรขึ้น (Up)** เราเพียงแค่ถอย `cursor` ย้อนหลังหนึ่งตำแหน่ง ดึงสตริงออกมาวางทับบัฟเฟอร์ปัจจุบัน แล้ววาดหน้าจอใหม่ทันทีในระดับศูนย์มิลลิวินาที!

---

## 🎯 7. สรุปบทเรียนสำหรับนักพัฒนา

1. **อิสรภาพทางวิศวกรรมเกิดขึ้นเมื่อเราตัดตัวกลางทิ้ง:** การสร้างระบบ Readline ของตัวเองทำให้เราประหยัดพื้นที่ไบนารีไปได้กว่า 800 KB และขจัดปัญหาเรื่องข้อตกลงไลเซนส์ได้อย่างถาวร
2. **เข้าใจภาษาของ Terminal:** การรู้ว่าหน้าจอเทอร์มินัลสื่อสารด้วยลำดับไบต์ VT100 ทำให้เราสามารถควบคุม UI ของระบบปฏิบัติการได้อย่างสมบูรณ์แบบ
3. **ความเรียบง่ายคือหัวใจของความเร็ว:** ด้วยโค้ดภาษา C เพียงประมาณ 250 บรรทัด เราได้ระบบพิมพ์คำสั่งที่ไหลลื่น ตอบสนองในพริบตา และมีหน่วยความจำสะอาดปราศจาก Memory Leaks

ในบทที่ 21 เราจะไปควบคุมการขัดจังหวะของระบบ: **การจัดการสัญญาณระดับเคอร์เนล: Job Control, Trap Handling และปราบ `SIGINT`, `SIGTSTP`, `SIGCHLD`**!
