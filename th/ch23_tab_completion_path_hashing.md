# บทที่ 23: ระบบ Autocompletion & Command Lookup: การกด Tab เติมคำ และ Fast $PATH Hashing
> **"Intelligent Command Lookup: Programmable Tab-Completion and O(1) PATH Hash Tables in C"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch23_tab_completion_path_hashing.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Speed is not just raw compute; it is remembering what you have already found."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**  
> *คำแปล:* "ความเร็วไม่ได้เป็นเพียงแค่พลังการประมวลผลดิบ แต่มันคือการจดจำสิ่งที่คุณเคยค้นพบมาแล้ว"
> 
> **"An algorithm must be seen to be believed, and the best way to see what an algorithm is doing is to participate with it."**  
> — **Donald Knuth**  
> *คำแปล:* "อัลกอริทึมต้องได้รับการเห็นจึงจะเชื่อได้ และวิธีที่ดีที่สุดในการดูว่ามันทำอะไร คือการมีปฏิสัมพันธ์ร่วมกับมัน"

---

## 🧭 1. ลมหายใจแห่งประสบการณ์ผู้ใช้: ปุ่ม Tab และ $PATH

หากคุณลองถามนักพัฒนาซอฟต์แวร์และวิศวกรระบบว่า:
> *"ปุ่มใดบนคีย์บอร์ดที่คุณกดบ่อยที่สุดเมื่ออยู่ใน Terminal?"*

คำตอบที่ชนะขาดลอยเกือบ 100% ไม่ใช่ปุ่มตัวอักษรใดๆ แต่คือ **ปุ่ม Tab (`\t`)**!  
ความคล่องตัวและมนตร์เสน่ห์ของการใช้งาน Command Line อยู่ที่การพิมพ์ตัวอักษรเพียง 2-3 ตัว แล้วกด Tab เพื่อให้ระบบเติมคำที่เหลือให้อัตโนมัติ:
```bash
$ gi[Tab] ➔ $ git st[Tab] ➔ $ git status
```

และเมื่อผู้ใช้กด Enter เพื่อสั่งรัน `git status`:
* ตัวเชลล์ต้องวิ่งค้นหาในทุกโฟลเดอร์ของ `$PATH` หรือไม่?
* หากตัวแปร `$PATH` มีโฟลเดอร์อยู่ 15 แห่ง (เช่น `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`...) เชลล์ต้องส่ง System Call `access()` หรือ `stat()` ไปยังฮาร์ดดิสก์ถึง 15 ครั้งทุกรอบการพิมพ์คำสั่งหรือไม่?
* คำตอบคือ: หากทำเช่นนั้น เชลล์จะช้าและเปลืองพลังงาน I/O โดยเปล่าประโยชน์!

ในบทนี้ เราจะมาแกะรอยการสร้าง **ระบบเติมคำอัจฉริยะ (Contextual Tab Autocompletion)** ควบคู่กับ **ระบบแคชคำสั่ง $O(1)$ ($PATH Hash Table)** ในภาษา C เพื่อทำให้เชลล์ของเราตอบสนองเร็วระดับฟ้าแลบ!

---

## 🏗️ 2. บริบทของการกด Tab (Context-Aware Autocompletion)

เมื่อผู้ใช้กดปุ่ม Tab ใน Line Editor สิ่งแรกที่เชลล์ต้องทำไม่ใช่การไปค้นหาไฟล์ทันที แต่คือการ **วิเคราะห์บริบทของเคอร์เซอร์ (Contextual Analysis)**:

```
                            [ Keystroke: Tab (\t) ]
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 ▼                                           ▼
      [ คำแรกของบรรทัด (Command) ]              [ คำถัดมา (Arguments / Flags) ]
      เติมชื่อ Builtins และ $PATH                 เติมชื่อไฟล์ / โฟลเดอร์ในดิสก์
                 │                                           │
                 └─────────────────────┬─────────────────────┘
                                       ▼
                         [ นำหน้าด้วยเครื่องหมาย $ ? ]
                                       │
                                       ├──────► ใช่: เติมชื่อตัวแปร ($HOME, $PATH...)
                                       └──────► ไม่: ค้นหาใน File System ปกติ
```

### การคำนวณ Longest Common Prefix (LCP):
หากผู้ใช้พิมพ์ `ca` แล้วในโฟลเดอร์มีทั้ง `cat.c` และ `catalog.txt`:
* เชลล์จะยังไม่เติมจนจบคำ แต่จะเติมให้ถึงตัวอักษรที่เหมือนกันมากที่สุด นั่นคือ `cat`
* หากผู้ใช้กด Tab ซ้ำสองครั้งติดกัน เชลล์จะพิมพ์รายการตัวเลือกที่เป็นไปได้ทั้งหมดออกมาให้เลือกดูอย่างสวยงาม!

---

## 🔍 3. สถาปัตยกรรม Tab Completion Engine ในภาษา C

ดูโค้ดการตรวจจับและเติมคำในบัฟเฟอร์:

```c
typedef struct {
    char **matches;
    int count;
    int cap;
} zbash_completion_list_t;

static void add_completion(zbash_completion_list_t *list, const char *word) {
    if (list->count >= list->cap) {
        list->cap = (list->cap == 0) ? 32 : list->cap * 2;
        list->matches = realloc(list->matches, sizeof(char *) * list->cap);
    }
    list->matches[list->count++] = strdup(word);
}

void zbash_complete_word(zbash_state_t *state, const char *prefix, bool is_command, zbash_completion_list_t *list) {
    size_t plen = strlen(prefix);

    /* 1. กรณีเป็นคำสั่ง: ค้นหาในรายการ Builtins ของ zbash */
    if (is_command) {
        static const char *builtins[] = { "cd", "pwd", "echo", "exit", "export", "read", "set", "unset", NULL };
        for (int i = 0; builtins[i]; i++) {
            if (strncmp(builtins[i], prefix, plen) == 0) {
                add_completion(list, builtins[i]);
            }
        }
    }

    /* 2. ค้นหาไฟล์และโฟลเดอร์ในไดเรกทอรีปัจจุบัน */
    DIR *dir = opendir(".");
    if (dir) {
        struct dirent *ent;
        while ((ent = readdir(dir)) != NULL) {
            if (ent->d_name[0] == '.' && prefix[0] != '.') continue;
            if (strncmp(ent->d_name, prefix, plen) == 0) {
                add_completion(list, ent->d_name);
            }
        }
        closedir(dir);
    }
}
```

เมื่อได้รายการคำตอบมาแล้ว:
* หาก `list->count == 1`: ทำการแทรกข้อความที่เหลือลงใน Line Buffer ทันที พร้อมเติมเว้นวรรคให้อัตโนมัติ
* หาก `list->count > 1`: คำนวณหาจุดร่วมที่ยาวที่สุดเพื่อเติมให้มากที่สุด แล้วส่งเสียงกระดิ่ง (Bell `\a`) แจ้งเตือนผู้ใช้

---

## ⚡ 4. คอขวดของ `$PATH` และการเกิด $PATH Hashing

ลองดูตัวแปร `$PATH` บนเครื่องของวิศวกรสมัยใหม่:
```bash
PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:~/.cargo/bin:~/.local/bin
```
เมื่อผู้ใช้พิมพ์คำสั่ง `ls`:
1. ตรวจสอบ `/opt/homebrew/bin/ls` (ไม่พบ) ➔ Disk Access 1
2. ตรวจสอบ `/usr/local/bin/ls` (ไม่พบ) ➔ Disk Access 2
3. ตรวจสอบ `/usr/bin/ls` (ไม่พบ) ➔ Disk Access 3
4. ตรวจสอบ `/bin/ls` (พบ!) ➔ Disk Access 4

หากผู้ใช้รันสคริปต์ที่วนลูป 10,000 รอบ เคอร์เนลต้องออกไปแตะฮาร์ดดิสก์ถึง **40,000 ครั้ง** เพียงเพื่อจะหาว่าคำสั่ง `ls` อยู่ที่ไหน!

### ทางออกทางวิศวกรรม: Hash Table $O(1)$ ในภาษา C
Bash มีกลไกภายในที่เรียกว่า **Command Hashing**:  
ครั้งแรกที่มีการเรียกใช้คำสั่งใด เชลล์จะค้นหาใน `$PATH` ตามปกติ แต่ทันทีที่พบ เชลล์จะบันทึกผลลัพธ์ลงใน **Hash Table** ทันที  
ในการเรียกครั้งที่ 2, 3, 4 เป็นต้นไป เชลล์จะข้ามการสแกนดิสก์ทั้งหมด แล้วดึง Path เต็มออกมาสั่ง `execve()` ได้ทันทีในเวลา $O(1)$!

---

## 🗄️ 5. โครงสร้างข้อมูล $PATH Hash Table ใน `zbash`

ดูการออกแบบ Hash Table ขนาดกะทัดรัด ไร้การรั่วไหลของหน่วยความจำ:

```c
#define ZBASH_HASH_BUCKETS 256

typedef struct path_hash_entry {
    char *cmd_name;                 /* เช่น "grep" */
    char *full_path;                /* เช่น "/usr/bin/grep" */
    int hits;                       /* จำนวนครั้งที่เคยเรียกใช้ */
    struct path_hash_entry *next;   /* Chaining สำหรับ Collision */
} path_hash_entry_t;

static path_hash_entry_t *g_path_table[ZBASH_HASH_BUCKETS];

/* ฟังก์ชันคำนวณ DJB2 Hash เร็วระดับ L1 Cache */
static unsigned int hash_cmd(const char *str) {
    unsigned int hash = 5381;
    int c;
    while ((c = *str++)) hash = ((hash << 5) + hash) + c;
    return hash % ZBASH_HASH_BUCKETS;
}

const char *zbash_lookup_cmd_hash(const char *cmd) {
    unsigned int bucket = hash_cmd(cmd);
    path_hash_entry_t *entry = g_path_table[bucket];
    while (entry) {
        if (strcmp(entry->cmd_name, cmd) == 0) {
            entry->hits++; /* บันทึกสถิติการใช้งาน */
            return entry->full_path; /* คืนพาธเต็มทันทีใน 0.001 ไมโครวินาที! */
        }
        entry = entry->next;
    }
    return NULL;
}
```

---

## 🛠️ 6. คำสั่ง Builtin `hash`

เมื่อผู้ใช้พิมพ์คำสั่ง `hash` เปล่าๆ ในเทอร์มินัล เชลล์จะแสดงสถิติ Hit Count ของคำสั่งทั้งหมดที่ถูกแคชไว้:

```bash
$ hash
hits    command
   4    /usr/bin/grep
  12    /bin/ls
   1    /usr/bin/find
```

### การล้างแคชด้วย `hash -r`:
หากผู้ใช้เพิ่งติดตั้งโปรแกรมใหม่ลงใน `/usr/local/bin` หรือแก้ไขตัวแปร `$PATH` แคชเดิมอาจชี้ไปยังไบนารีเก่า คำสั่ง `hash -r` จะทำหน้าที่วนลูป `free()` หน่วยความจำของทุก Bucket ในตาราง Hash เพื่อให้เชลล์เริ่มค้นหาใหม่จากดิสก์อย่างถูกต้อง

---

## 🎯 7. สรุปบทเรียนสำหรับนักพัฒนา

1. **UX คือหัวใจของเครื่องมือที่มีชีวิต:** ระบบ Tab Autocompletion เปลี่ยนเชลล์จากกล่องข้อความทื่อๆ ให้กลายเป็นเครื่องมือคู่ใจที่รู้ใจนักพัฒนา
2. **อย่าละเลยต้นทุนของ I/O ในระดับเคอร์เนล:** การลด System Call `access()` หรือ `stat()` ผ่าน Hash Table ช่วยประหยัดรอบการทำงานของซีพียูและลดเวลาการรันสคริปต์ซ้ำๆ ได้มหาศาล
3. **การออกแบบแคชที่ดีต้องมีกลไกล้างเสมอ:** การเปิดทางให้ผู้ใช้ล้างแคชได้ผ่านคำสั่ง `hash -r` คือหัวใจของความเสถียรภาพและความถูกต้องในการทำงาน

ในบทที่ 24 เราจะมาเปิดเผยความลับที่ทุกคนอยากรู้ที่สุด: **ศิลปะการรีดไบนารีระดับเอ็กซ์ตรีม: จาก C Source สู่ 86 KB Executable**!
