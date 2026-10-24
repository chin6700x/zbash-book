# บทที่ 3: สถาปัตยกรรมภาพรวม: จากคีย์บอร์ดสู่ OS Process Execution
> **"Architecture Overview: From Keyboard Strokes to OS Process Execution"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch03_architecture_overview.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"A shell is not just a language for writing scripts; it is a direct extension of your thoughts into the operating system."**  
> — **Stephen Bourne** (1979)  
> *คำแปล:* "เชลล์ไม่ได้เป็นเพียงแค่ภาษาสำหรับการเขียนสคริปต์ แต่มันคือการเชื่อมต่อความคิดของคุณเข้าสู่ระบบปฏิบัติการโดยตรง"
> 
> **"Data dominates. If you've chosen the right data structures and organized things well, the algorithms will almost always be self-evident."**  
> — **Rob Pike** (1989)  
> *คำแปล:* "โครงสร้างข้อมูลคือหัวใจ หากคุณเลือกโครงสร้างข้อมูลที่ถูกต้องและเป็นระเบียบ อัลกอริทึมจะปรากฏขึ้นมาเองอย่างชัดเจนแทบทุกครั้ง"

---

## 🧭 1. วงจรชีวิตของคำสั่งใน Shell (The Life of a Command)

เมื่อผู้ใช้พิมพ์คำสั่งบนเทอร์มินัล เช่น:
```bash
echo "Hello World" | grep Hello > result.txt
```
แล้วกดปุ่ม **[Enter]**... เกิดอะไรขึ้นบ้างในระดับแกนลึกของ `zbash`?

ในบทนี้ เราจะไม่พูดแบบทฤษฎีลอยๆ แต่เราจะมาผ่าสถาปัตยกรรมทีละขั้นตอน จากโค้ดจริงในโครงการ เพื่อให้เห็นว่าข้อมูลเดินทางผ่านระบบอย่างไร:

```
[ 1. Input Source ] ──> [ 2. SIMD Lexer ] ──> [ 3. AST Parser ] ──> [ 4. Variable Expander ] ──> [ 5. Execution Engine ]
 (stdin / -c / file)     (ตัดคำ 16 ไบต์/Tick)    (สร้าง Pipeline AST)   (ขยาย $VAR, $((..)))       (Fork, Pipe, Execvp)
```

---

## 🚀 2. จุดเริ่มต้นของการรับข้อมูล: ความลับของ `fmemopen` ใน `src/main.c`

การรับคำสั่งเข้ามาประมวลผลใน `zbash` รองรับ 3 รูปแบบหลัก:
1. การรันผ่าน Flag `-c`: เช่น `zbash -c "echo 123"`
2. การรันผ่านไฟล์สคริปต์: เช่น `zbash script.sh`
3. การรันแบบ Interactive: เมื่อเปิดหน้าต่าง Terminal เปล่าๆ

ลองมาดูโค้ดจริงในไฟล์ [`src/main.c`](src/main.c) บรรทัดที่ 70 ถึง 93:

```c
/* ไฟล์: src/main.c (บรรทัดที่ 70 - 93) */
    if (cmd_string) {
        state.argc = (argc > opt_idx) ? (argc - opt_idx + 1) : 1;
        state.argv = (argc > opt_idx) ? (argv + opt_idx - 1) : argv;
        FILE *mem_fp = fmemopen((char *)cmd_string, strlen(cmd_string), "r");
        if (mem_fp) {
            zbash_run_stream(&state, mem_fp);
            fclose(mem_fp);
        } else {
            zbash_run_line(&state, cmd_string);
        }
        return state.should_exit ? state.exit_code : state.last_status;
    }

    if (opt_idx < argc) {
        const char *script_file = argv[opt_idx];
        state.argc = argc - opt_idx;
        state.argv = argv + opt_idx;
        return zbash_run_file(&state, script_file);
    }

    state.argc = 1;
    state.argv = argv;
    return zbash_run_interactive(&state);
```

### เจาะลึกเทคนิค `fmemopen()`:
* ในบรรทัดที่ 73 แทนที่เราจะสร้างไฟล์ชั่วคราว (Temporary File) ลงดิสก์ หรือใช้ Buffer ช้าๆ เราใช้ System Call พิเศษของ POSIX คือ **`fmemopen()`**
* ฟังก์ชันนี้จะแปลงสตริง `cmd_string` ในหน่วยความจำ RAM ให้กลายเป็น `FILE* Stream` ทันทีโดยไม่ต้องแตะต้องฮาร์ดดิสก์แม้แต่น้อย ทำให้ฟังก์ชัน `zbash_run_stream` สามารถอ่านคำสั่งแบบสตรีมได้ด้วยความเร็วระดับ RAM Bus!

---

## 🏗️ 3. โครงสร้างข้อมูลหัวใจ: `zbash_cmd_t` และ `zbash_pipeline_t`

เชลล์ที่ดีต้องมีโครงสร้างข้อมูลที่กะทัดรัดและชัดเจน ในไฟล์ [`include/zbash.h`](include/zbash.h) บรรทัดที่ 73 ถึง 97 เรานิยามโครงสร้างคำสั่งและท่อไปป์ไว้ดังนี้:

### 1) โครงสร้างคำสั่งเดี่ยว (`zbash_cmd_t`)
```c
/* ไฟล์: include/zbash.h (บรรทัดที่ 73 - 84) */
typedef struct {
    char *argv[ZBASH_MAX_ARGS];   /* อาร์เรย์ของ Arguments (สูงสุด 64 ตัว) */
    int argc;                     /* จำนวน Arguments */
    char *in_file;                /* การเปลี่ยนทิศทาง Input (< file) */
    char *out_file;               /* การเปลี่ยนทิศทาง Output (> file) */
    bool append_out;              /* โหมดเขียนต่อท้าย (>> file) */
    char *err_file;               /* การเปลี่ยนทิศทาง Error (2> file) */
    bool append_err;              /* โหมดเขียนต่อท้าย Error (2>> file) */
    bool err_to_out;              /* ผสาน Error เข้ากับ Output (2>&1) */
    bool out_to_err;              /* ส่ง Output ไปที่ Error (>&2 หรือ 1>&2) */
} zbash_cmd_t;
```
* **Redirection Flags:** สังเกตว่าเราเก็บข้อมูลการเปลี่ยนทิศทาง I/O เช่น `2>&1` ไว้ในตัวแปรบูลีน `err_to_out` (บรรทัดที่ 81) ทำให้ตัว Execution Engine สามารถสลับ File Descriptors ได้ทันทีโดยไม่ต้อง Parse สตริงซ้ำซ้อน

### 2) โครงสร้างท่อไปป์และตัวเชื่อมตรรกะ (`zbash_pipeline_t`)
```c
/* ไฟล์: include/zbash.h (บรรทัดที่ 85 - 97) */
typedef enum {
    CONN_NONE,
    CONN_AND,     /* && (ทำงานเมื่อคำสั่งก่อนหน้าสำเร็จ) */
    CONN_OR,      /* || (ทำงานเมื่อคำสั่งก่อนหน้าล้มเหลว) */
    CONN_SEMI     /* ;  (ทำงานต่อกันโดยไม่สนผลลัพธ์) */
} zbash_conn_t;

typedef struct {
    zbash_cmd_t cmds[ZBASH_MAX_PIPELINE];  /* ชุดคำสั่งที่เชื่อมด้วย | (สูงสุด 16 คำสั่ง) */
    int cmd_count;                         /* จำนวนคำสั่งในท่อไปป์ */
    zbash_conn_t next_conn;                /* ตัวเชื่อมกับ Pipeline ถัดไป (&&, ||, ;) */
} zbash_pipeline_t;
```
* ด้วยโครงสร้างนี้ ประโยคซับซ้อนอย่าง `cat data.txt | grep error | wc -l && echo "Done"` จะถูกแตกเป็น:
  - **Pipeline 1:** ประกอบด้วย 3 คำสั่ง (`cat`, `grep`, `wc`) เชื่อมกันด้วย Pipe `|` และมี `next_conn = CONN_AND`
  - **Pipeline 2:** ประกอบด้วย 1 คำสั่ง (`echo "Done"`) มี `next_conn = CONN_NONE`

---

## ⚙️ 4. กลไกการกระจายคำสั่ง: Dispatcher Engine ใน `src/core/engine.c`

เมื่อ Parser สร้าง `zbash_pipeline_t` เสร็จแล้ว ตัวเชลล์จะส่งไปให้ Execution Engine ในไฟล์ [`src/core/engine.c`](src/core/engine.c)

ลองมาดูจุดสำคัญที่สุดในกระบวนการ Fork Process ย่อย ในบรรทัดที่ 1037 ถึง 1070:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1037 - 1070) */
// 1. ตรวจสอบ Command Grouping เช่น { echo 1; echo 2; }
if (strcmp(cmd->argv[0], "{") == 0) {
    int rc = (cmd->argc > 1 && cmd->argv[1]) ? zbash_run_line(state, cmd->argv[1]) : 0;
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 2. ตรวจสอบ Subshell เช่น ( cd /tmp && ls )
if (strcmp(cmd->argv[0], "(") == 0) {
    int rc = (cmd->argc > 1 && cmd->argv[1]) ? zbash_run_line(state, cmd->argv[1]) : 0;
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 3. ตรวจสอบ Shell Function ที่ผู้ใช้ประกาศไว้
zbash_func_t *fn = zbash_find_func(state, cmd->argv[0]);
if (fn) {
    int rc = zbash_exec_func(state, fn, cmd->argc, cmd->argv);
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 4. ตรวจสอบ Shell Builtin เช่น cd, export, echo, read
if (zbash_is_builtin(cmd->argv[0])) {
    int rc = zbash_exec_builtin(state, cmd->argc, cmd->argv);
    fflush(stdout); fflush(stderr);
    _exit(state->should_exit ? state->exit_code : rc);
}

// 5. หากไม่ใช่ทั้งหมดข้างต้น ให้เรียกโปรแกรมภายนอกของระบบปฏิบัติการ
execvp(cmd->argv[0], cmd->argv);
fprintf(stderr, "zbash: %s: %s\n", cmd->argv[0], strerror(errno));
_exit((errno == ENOENT) ? 127 : 126);
```

### ลำดับความสำคัญในการค้นหาคำสั่ง (Execution Precedence):
1. **Compound Groups (`{` หรือ `(`):** หากเป็นวงเล็บ ให้เรียกตัวประมวลผลคำสั่งย่อย
2. **User Functions:** ค้นหาในตารางฟังก์ชัน `state->funcs` ก่อนเสมอ
3. **Builtin Commands:** หากชื่อตรงกับคำสั่ง Builtin เช่น `echo`, `cd`, `test`, `export` ให้รันฟังก์ชัน C ภายในตัวเชลล์เองทันที
4. **External Programs (`execvp`):** หากไม่พบในเชลล์ จึงส่งต่อให้เคอร์เนลค้นหาใน `$PATH`
5. **Exact POSIX Exit Code:** หากไม่พบไฟล์ (`ENOENT`) ต้องคืนค่า **`127`** และหากไม่มีสิทธิ์รัน (`EACCES`) ต้องคืนค่า **`126`**

---

## 🚰 5. ศิลปะของท่อไปป์ (Pipeline Plumbing) และการดึง Exit Status

ในระบบ Unix เมื่อเราต่อคำสั่งด้วยท่อไปป์ `cmd1 | cmd2` Process ลูกทุกตัวต้องถูก Fork ขึ้นมาทำงานพร้อมกันแบบคู่ขนาน (Concurrency)

ลองดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1007 ถึง 1031:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1007 - 1031) */
    int pipe_fds[64][2];
    for (int i = 0; i < n - 1; i++) {
        if (pipe(pipe_fds[i]) < 0) {
            perror("zbash: pipe");
            return 1;
        }
    }

    pid_t pids[64];
    for (int i = 0; i < n; i++) {
        zbash_cmd_t *cmd = &pipeline->cmds[i];
        pids[i] = fork();

        if (pids[i] == 0) {
            // หากไม่ใช่คำสั่งแรก ให้ดึง Input มาจาก Read-End ของท่อก่อนหน้า
            if (i > 0) {
                dup2(pipe_fds[i - 1][0], STDIN_FILENO);
            }
            // หากไม่ใช่คำสั่งสุดท้าย ให้ส่ง Output ไปที่ Write-End ของท่อถัดไป
            if (i < n - 1) {
                dup2(pipe_fds[i][1], STDOUT_FILENO);
            }

            // ปิด File Descriptors ของท่อทั้งหมดใน Process ลูก เพื่อไม่ให้ค้าง
            for (int j = 0; j < n - 1; j++) {
                close(pipe_fds[j][0]);
                close(pipe_fds[j][1]);
            }
```

### และจุดตายที่สำคัญที่สุด: การปิดท่อใน Process แม่
ในบรรทัดที่ 1076 ถึง 1089:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1076 - 1089) */
    for (int j = 0; j < n - 1; j++) {
        close(pipe_fds[j][0]);
        close(pipe_fds[j][1]);
    }

    int last_status = 0;
    for (int i = 0; i < n; i++) {
        int status = 0;
        waitpid(pids[i], &status, 0);
        if (i == n - 1) {
            if (WIFEXITED(status)) last_status = WEXITSTATUS(status);
            else if (WIFSIGNALED(status)) last_status = 128 + WTERMSIG(status);
        }
    }
    return last_status;
```

#### ทำไมลูป `close()` ในบรรทัด 1076 จึงเป็นเรื่องคอขาดบาดตาย?
* หาก Process แม่ลืมปิด `pipe_fds[j][1]` (Write-End) ปลายท่อของลูกตัวถัดไปจะไม่ได้รับสัญญาณ **EOF (End-Of-File)** และจะทำให้โปรแกรมอย่าง `grep` หรือ `wc` นั่งรอข้อมูลค้างอยู่ในเทอร์มินัลตลอดกาล!
* ในบรรทัดที่ 1085 สังเกตว่าเราจะเก็บ `last_status` เฉพาะของคำสั่งสุดท้ายในท่อไปป์ (`i == n - 1`) เพื่อให้ตัวแปร `$?` มีค่าตรงตามมาตรฐาน POSIX

---

## 🏁 สรุปบทเรียนที่ 3

ในบทนี้ เราได้เห็นภาพรวมสถาปัตยกรรมทั้งระบบของ `zbash`:
1. การเริ่มต้นรับคำสั่งแบบ Zero-Disk ด้วย `fmemopen()` ใน `src/main.c`
2. โครงสร้างข้อมูล `zbash_cmd_t` และ `zbash_pipeline_t` ใน `include/zbash.h` ที่เรียบง่ายแต่ครอบคลุมทุก Flag
3. การกระจายงาน (Dispatching) ใน `src/core/engine.c` ที่จัดลำดับความสำคัญอย่างถูกต้อง
4. กลไกการต่อท่อไปป์ `pipe()`, `dup2()`, และการรอเก็บ Exit Code ด้วย `waitpid()`

ใน **ภาคที่ 2: สถาปัตยกรรม Lexer & Parser (บทที่ 4)** เราจะดำดิ่งลงลึกสู่ระดับบิตและไบต์: ผ่าโค้ด **ARM64 NEON Assembly 128-bit** เพื่อดูว่าเราใช้ฮาร์ดแวร์เวกเตอร์เร่งความเร็วการตัดคำได้อย่างไรครับ! 🚀📖
