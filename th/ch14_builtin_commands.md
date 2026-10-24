# บทที่ 14: ทำไมคำสั่ง cd, export, read จึงไม่สามารถเป็นโปรแกรมภายนอกได้?
> **"The Anatomy of Shell Builtins: Why cd, export, and read Must Mutate the Parent Process"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch14_builtin_commands.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Simple things should be simple, complex things should be possible."**  
> — **Alan Kay**  
> *คำแปล:* "สิ่งที่เรียบง่ายควรจะทำได้โดยง่าย และสิ่งที่ซับซ้อนก็ควรจะเป็นไปได้ที่จะทำ"
> 
> **"A shell is not just a command launcher; it is an environment keeper. What alters the keeper must live within its soul."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**  
> *คำแปล:* "เชลล์ไม่ได้เป็นเพียงแค่ตัวยิงคำสั่ง แต่มันคือผู้พิทักษ์สภาพแวดล้อม สิ่งใดก็ตามที่ต้องการเปลี่ยนแปลงตัวผู้พิทักษ์ สิ่งนั้นจะต้องสถิตอยู่ภายในจิตวิญญาณของมันเอง"

---

## 🧭 1. ปริศนาแห่งคำสั่ง `cd`: ทำไมเราสร้าง `/bin/cd` ไม่ได้?

สำหรับผู้เริ่มต้นเรียนรู้ระบบปฏิบัติการ Unix คำถามที่มักเกิดขึ้นเสมอคือ:
> *"ทำไมในระบบมีโปรแกรม `/bin/ls`, `/bin/cat`, `/bin/mkdir` แต่ทำไมเราถึงสร้างโปรแกรมภายนอกชื่อ `/bin/cd` เพื่อเปลี่ยนไดเรกทอรีไม่ได้?"*

ลองจินตนาการว่าหากมีโปรแกรม `/bin/cd` อยู่จริงบนเครื่อง:
1. ผู้ใช้พิมพ์คำสั่ง `cd /tmp` บนหน้าจอ
2. เชลล์หลักทำการเรียก `fork()` เพื่อสร้าง Child Process
3. Child Process ทำการเรียก `execve("/bin/cd", ["cd", "/tmp"])`
4. โปรแกรม `/bin/cd` เรียก System Call `chdir("/tmp")` สำเร็จอย่างงดงาม
5. โปรแกรม `/bin/cd` จบการทำงานและเรียก `exit(0)`
6. เชลล์หลักตรวจพบว่าลูกทำงานเสร็จสิ้นแล้ว... **แต่โฟลเดอร์ปัจจุบันของเชลล์หลักกลับยังอยู่ที่เดิมไม่ขยับเลยแม้แต่มิลลิเมตรเดียว!**

**เหตุผลทางสถาปัตยกรรม:** ในระบบ Unix กระบวนการลูกไม่สามารถเปลี่ยนแปลงสภาพแวดล้อม (Working Directory, Environment Variables, File Descriptor State) ของกระบวนการแม่ได้!  
ดังนั้น คำสั่งใดก็ตามที่มีหน้าที่เปลี่ยนแปลงสถานะของตัวเชลล์เอง **ต้องถูกสร้างเป็น Builtin Command ที่ทำงานอยู่ภายในตัวเชลล์เองเสมอ**

ในบทนี้ เราจะมาผ่าโครงสร้าง **Builtin Command Engine** ใน `zbash` ที่บรรจุคำสั่งหลัก 24 ตัวไว้ในขนาดกะทัดรัด และรักษาพฤติกรรมเทียบเท่า GNU Bash แบบ 100%!

---

## 🏗️ 2. สารบบ 24 คำสั่ง Builtin ใน `zbash`

ดูโค้ดตรวจสอบคำสั่งใน [`src/builtins/builtins.c`](src/builtins/builtins.c) บรรทัดที่ 16 ถึง 42:

```c
/* ไฟล์: src/builtins/builtins.c (บรรทัดที่ 16 - 42) */
bool zbash_is_builtin(const char *name) {
    if (!name) return false;
    return (strcmp(name, "cd") == 0 ||
            strcmp(name, "pwd") == 0 ||
            strcmp(name, "echo") == 0 ||
            strcmp(name, "exit") == 0 ||
            strcmp(name, "export") == 0 ||
            strcmp(name, "unset") == 0 ||
            strcmp(name, "set") == 0 ||
            strcmp(name, "true") == 0 ||
            strcmp(name, ":") == 0 ||
            strcmp(name, "false") == 0 ||
            strcmp(name, "type") == 0 ||
            strcmp(name, "shift") == 0 ||
            strcmp(name, "exec") == 0 ||
            strcmp(name, "read") == 0 ||
            strcmp(name, "test") == 0 ||
            strcmp(name, "[") == 0 ||
            strcmp(name, "source") == 0 ||
            strcmp(name, ".") == 0 ||
            strcmp(name, "eval") == 0 ||
            strcmp(name, "local") == 0 ||
            strcmp(name, "trap") == 0 ||
            strcmp(name, "break") == 0 ||
            strcmp(name, "continue") == 0 ||
            strcmp(name, "return") == 0);
}
```

เมื่อ Lexer พบว่าคำแรกของคำสั่งตรงกับรายชื่อนี้ Engine จะข้ามขั้นตอนการเรียก `fork()` ไปทันที (เว้นแต่จะอยู่ในท่อไปป์) และเรียกฟังก์ชันตัวจัดการโดยตรง

---

## 🔍 3. เจาะลึกการสร้าง Builtins สำคัญ

### 1. `builtin_cd`: การจัดการเส้นทางและ `$OLDPWD`
คำสั่ง `cd` ไม่ได้มีหน้าที่แค่เรียก `chdir()` แต่ต้องคอยอัปเดตตัวแปรระบบที่สคริปต์พึ่งพาเสมอ:

```c
/* โค้ดหลักของ builtin_cd ใน src/builtins/builtins.c */
static int builtin_cd(zbash_state_t *state, int argc, char **argv) {
    const char *target = NULL;
    char cwd_buf[PATH_MAX];
    getcwd(cwd_buf, sizeof(cwd_buf));

    if (argc < 2) {
        target = getenv("HOME"); /* cd เปล่าๆ ให้กลับบ้าน */
        if (!target) target = "/";
    } else if (strcmp(argv[1], "-") == 0) {
        target = getenv("OLDPWD"); /* cd - ให้กลับไปยังโฟลเดอร์ก่อนหน้า */
        if (!target) {
            fprintf(stderr, "zbash: cd: OLDPWD not set\n");
            return 1;
        }
        printf("%s\n", target); /* Bash กำหนดให้พิมพ์ path เมื่อใช้ cd - */
    } else {
        target = argv[1];
    }

    if (chdir(target) != 0) {
        fprintf(stderr, "zbash: cd: %s: %s\n", target, strerror(errno));
        return 1;
    }

    /* อัปเดตตัวแปร OLDPWD และ PWD ให้ตรงกับความเป็นจริง */
    setenv("OLDPWD", cwd_buf, 1);
    if (getcwd(cwd_buf, sizeof(cwd_buf))) {
        setenv("PWD", cwd_buf, 1);
    }
    return 0;
}
```

### 2. `builtin_export`: นำตัวแปรออกจากเชลล์สู่โปรเซสภายนอก
ใน Unix ตัวแปรที่ประกาศปกติ เช่น `FOO=bar` จะอยู่เฉพาะในหน่วยความจำของเชลล์  
หากผู้ใช้ต้องการให้โปรแกรมภายนอก (เช่น Node.js หรือ Python) มองเห็นตัวแปรนี้ด้วย ต้องส่งผ่าน Environment Block:

```c
static int builtin_export(int argc, char **argv) {
    for (int i = 1; i < argc; i++) {
        char *eq = strchr(argv[i], '=');
        if (eq) {
            *eq = '\0';
            setenv(argv[i], eq + 1, 1);
            *eq = '=';
        } else {
            /* หากไม่มีเครื่องหมาย = ให้คงค่าเดิมไว้แล้วส่งออก */
            char *val = getenv(argv[i]);
            if (val) setenv(argv[i], val, 1);
        }
    }
    return 0;
}
```

### 3. `builtin_read`: รับข้อมูลและแยกคำตาม `$IFS`
คำสั่ง `read` ทำหน้าที่อ่านข้อมูลจากอินพุต (คีย์บอร์ดหรือท่อไปป์) แล้วนำมาแยกเป็นตัวแปร:
* รองรับตัวเลือก `-r` เพื่อห้ามตีความเครื่องหมาย Backslash (`\`)
* แยกคำตามอักขระ `$IFS` (Internal Field Separator ซึ่งค่าเริ่มต้นคือช่องว่าง, แท็บ, และขึ้นบรรทัดใหม่)
* หากผู้ใช้พิมพ์ข้อมูลยาวเกินจำนวนตัวแปรที่รับ ข้อมูลที่เหลือทั้งหมดจะถูกเทรวมลงในตัวแปรตัวสุดท้ายตามมาตรฐาน POSIX

### 4. `builtin_source` หรือ `.`: การรันสคริปต์ในกระบวนการเดิม
```c
/* ไฟล์: src/builtins/builtins.c (บรรทัดที่ 641 - 647) */
    if (strcmp(cmd, "source") == 0 || strcmp(cmd, ".") == 0) {
        if (argc < 2) {
            fprintf(stderr, "zbash: %s: filename argument required\n", cmd);
            return 2;
        }
        return zbash_source_file(state, argv[1]);
    }
```
เมื่อผู้ใช้สั่ง `. script.sh` ตัวเชลล์จะไม่ `fork()` โปรเซสใหม่ แต่จะเปิดไฟล์แล้วส่งแต่ละบรรทัดเข้า `zbash_run_line(state, line)` ในตัวเชลล์เองทันที ทำให้ค่าตัวแปรหรือฟังก์ชันที่สคริปต์นั้นประกาศ กลายเป็นสมบัติของเชลล์หลักทันที!

---

## 🧪 4. การทดสอบยืนยันความถูกต้องระดับ Parity (Differential Proof)

ในโมดูล `test/1000-test/test_01_builtins_core.sh` มีชุดทดสอบ 100 ข้อ:

| คำสั่ง Builtin | การทดสอบพฤติกรรม | ผลลัพธ์ vs GNU Bash |
| :--- | :--- | :--- |
| **`cd`** | `cd`, `cd -`, `cd ..`, ไดเรกทอรีที่ไม่มีจริง | **PASS (Exact Match)** |
| **`pwd`** | ตรวจสอบความถูกต้องของ Working Directory | **PASS (Exact Match)** |
| **`echo`** | `echo -n`, `echo -e`, escapes (`\n`, `\t`) | **PASS (Exact Match)** |
| **`export`** | การส่งต่อตัวแปรสู่โปรแกรมภายนอก | **PASS (Exact Match)** |
| **`read`** | `read -r`, การแบ่งคำด้วย `$IFS` | **PASS (Exact Match)** |
| **`source`** | การโหลดค่าตัวแปรและฟังก์ชันข้ามไฟล์ | **PASS (Exact Match)** |
| **`test` / `[`** | การทดสอบไฟล์ `-f`, `-d`, `-z`, `-n`, ตัวเลข `-eq` | **PASS (Exact Match)** |

**อัตราความผ่าน:** `100 / 100 PASS (100.0%)`

---

## 🎯 5. สรุปบทเรียนสำหรับนักพัฒนา

1. **เข้าใจขอบเขตอำนาจของโปรเซส:** การรู้ว่าสิ่งใดที่กระบวนการภายนอกทำไม่ได้ เป็นกุญแจสำคัญในการตัดสินใจว่าฟีเจอร์ใดต้องถูกสร้างเป็น Builtin
2. **รักษาความสมบูรณ์ของตัวแปรระบบ:** คำสั่งอย่าง `cd` ต้องดูแลตัวแปรแฝงอย่าง `$PWD` และ `$OLDPWD` ให้ถูกต้องเสมอ เพราะสคริปต์ระดับโปรดักชันจำนวนมากพึ่งพาตัวแปรเหล่านี้
3. **ประสิทธิภาพที่เหนือกว่า:** คำสั่ง Builtin ไม่ต้องผ่านวงจรชีวิต `fork()` และ `execve()` จึงทำความเร็วได้เหนือกว่าโปรแกรมภายนอกนับร้อยเท่า

จบสิ้น **ภาคที่ 4: เครื่องยนต์ประมวลผลคำสั่งและท่อไปป์** อย่างงดงาม!  
ในบทถัดไป เราจะก้าวเข้าสู่ **ภาคที่ 5: มาตรฐานการทดสอบระดับโลก (1,000 Parity Tests)** เพื่อเปิดเผยปรัชญาเบื้องหลัง: **"เร็วแต่ผลลัพธ์ผิด คือไร้ค่า": ปรัชญาการทดสอบระดับ Differential**!
