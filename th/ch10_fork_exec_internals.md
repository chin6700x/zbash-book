# บทที่ 10: ดำดิ่งระบบปฏิบัติการ: เข้าใจ fork(), execve(), และ Process Lifecycle
> **"Diving Deep into the Operating System: Mastering fork(), execve(), and Process Lifecycle"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch10_fork_exec_internals.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"The Unix programming environment is based on the idea of small, modular programs that work together cleanly through files and processes."**  
> — **W. Richard Stevens** (*Advanced Programming in the UNIX Environment*)  
> *คำแปล:* "สภาพแวดล้อมการเขียนโปรแกรมของ Unix วางอยู่บนรากฐานของโปรแกรมขนาดเล็กที่มีความเป็นโมดูล ซึ่งทำงานประสานกันอย่างหมดจดผ่านไฟล์และโปรเซส"
> 
> **"Talk is cheap. Show me the code."**  
> — **Linus Torvalds**  
> *คำแปล:* "คำพูดใครก็พูดได้ จงเปิดโค้ดให้ดูเดี๋ยวนี้"

---

## 🧭 1. มนตร์เสน่ห์ของกระบวนการ Unix: จากตัวอักษรสู่ Process

เมื่อคุณพิมพ์คำสั่งบนคีย์บอร์ด เช่น:
```bash
ls -la /tmp
```
แล้วกด Enter สิ่งที่เกิดขึ้นภายในเสี้ยววินาทีไม่ใช่แค่การอ่านสตริง แต่คือการที่โปรแกรมเชลล์ต้องขอร้องให้ **Kernel ของระบบปฏิบัติการ (Darwin / Linux)** ชุบชีวิตโปรแกรมใหม่ขึ้นมา ทำงานให้เสร็จ แล้วรายงานผลลัพธ์กลับมา

นี่คือแก่นแท้ของระบบปฏิบัติการ Unix ที่สืบทอดมายาวนานกว่า 50 ปี:
* **`fork()`:** การโคลนกระบวนการปัจจุบัน (Parent Process) ออกมาเป็นกระบวนการลูก (Child Process) ที่มีหน่วยความจำและตาราง File Descriptors เหมือนกันทุกประการ
* **`execve()` / `execvp()`:** การสลัดคราบภาพจำเดิมของโปรแกรมทิ้ง แล้วโหลดไบนารีไฟล์ใหม่เข้ามาทับพื้นที่หน่วยความจำของ Child Process เพื่อเริ่มต้นทำงานจากฟังก์ชัน `main()` ของโปรแกรมนั้น
* **`waitpid()`:** การที่ Parent Process เฝ้ารอคอยให้ลูกทำงานจนเสร็จ เพื่อรับรู้รหัสสถานะการสิ้นสุด (Exit Status) และเก็บกวาดศพโปรเซส (Reaping Zombie Processes) ไม่ให้คั่งค้างในตารางโปรเซสของเคอร์เนล

ในบทนี้ เราจะมาแกะรอยโค้ดจริงใน `zbash` เพื่อดูว่าเราควบคุมวงจรชีวิตของโปรเซสเหล่านี้ได้อย่างไรโดยไร้ Overhead และตรงตามมาตรฐาน POSIX 100%!

---

## 🏗️ 2. แผนผังวงจรชีวิตของโปรเซส (Process Lifecycle Sequence)

```
        Parent Process (zbash)
                 │
                 ├─── 1. fork() ─────────────────────────────┐
                 │                                           ▼
                 │                                  Child Process (Clone)
                 │                                           │
                 │                                           ├─── 2. apply_redirections()
                 │                                           │    (จัดทางเดิน I/O ด้วย dup2)
                 │                                           │
                 │                                           ├─── 3. execvp("ls", argv)
                 │                                           │    (แปลงร่างเป็นไบนารีเป้าหมาย)
                 │                                           ▼
                 │                                  External Binary Running...
                 │                                           │
                 ├─── 4. waitpid(pid, &status) ◄─────────────┘
                 │    (Parent รอจนกว่าลูกจะจบ)                 (Exit: exit code / signal)
                 ▼
         ดึง Exit Status ($?)
       (127 = Not Found, 126 = No Perm,
        128 + N = Signal Termination)
```

---

## 🔍 3. เจาะลึกการรันคำสั่งภายนอกแบบเดี่ยว (Single External Command)

ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 954 ถึง 970:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 954 - 970) */
    pid_t pid = fork();
    if (pid == 0) {
        /* โค้ดส่วนนี้ทำงานใน Child Process */
        if (apply_redirections(cmd) != 0) _exit(1);
        execvp(effective_argv[0], effective_argv);
        
        /* หากโค้ดหลุดมาถึงบรรทัดนี้ แสดงว่า execvp ล้มเหลว! */
        fprintf(stderr, "zbash: %s: %s\n", effective_argv[0], strerror(errno));
        _exit((errno == ENOENT) ? 127 : 126);
    } else if (pid < 0) {
        /* เกิดข้อผิดพลาด ไม่สามารถ fork ได้ (เช่น process limit เต็ม) */
        perror("zbash: fork");
        return 1;
    }

    /* โค้ดส่วนนี้ทำงานใน Parent Process (zbash หลัก) */
    int status = 0;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status)) return WEXITSTATUS(status);
    if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);
    return 1;
```

### รายละเอียดและเทคนิคเชิงลึกที่ต้องเข้าใจ:

#### 1. ทำไมต้องใช้ `_exit()` แทน `exit()` ในกระบวนการลูก?
สังเกตว่าในบรรทัดที่ 956 และ 959 เราเรียก `_exit()` ไม่ใช่ `exit()` ธรรมดา!  
* **`exit()`:** จะทำการล้างบัฟเฟอร์ I/O ของภาษา C (`stdio` buffer) และเรียกฟังก์ชันทำความสะอาดที่ลงทะเบียนไว้ผ่าน `atexit()` ของ Parent Process ซึ่งอาจส่งผลเสียต่อการทำงานของโปรเซสหลัก
* **`_exit()`:** เป็น System Call โดยตรงของระบบปฏิบัติการที่ปิดกระบวนการลูกทันทีที่ระดับเคอร์เนล โดยไม่แตะต้องบัฟเฟอร์หรือตัวจัดการของ Parent

#### 2. มาตรฐานรหัสข้อผิดพลาด POSIX: 127 vs 126
ในบรรทัดที่ 959:
```c
_exit((errno == ENOENT) ? 127 : 126);
```
นี่คือกฎเหล็กของ POSIX ที่เชลล์ชั้นดีต้องปฏิบัติตามอย่างเคร่งครัด:
* **Exit Code `127`:** เกิดขึ้นเมื่อ `errno == ENOENT` (Error: No such file or directory) หมายถึงคำสั่งนั้น **ไม่มีอยู่จริงใน `$PATH`**
* **Exit Code `126`:** เกิดขึ้นเมื่อพบไฟล์ แต่ **ไม่มีสิทธิ์ในการรัน (Permission denied / EACCES)** หรือไฟล์ไม่ได้อยู่ในรูปแบบไบนารีที่รันได้ (`ENOEXEC`)

#### 3. การแกะรหัสสถานะและการจัดการสัญญาณ (Signal Handling)
ในฝั่ง Parent:
```c
if (WIFEXITED(status)) return WEXITSTATUS(status);
if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);
```
* **`WIFEXITED(status)`:** โปรเซสจบการทำงานแบบปกติผ่าน `exit(code)` ค่าที่ดึงได้จาก `WEXITSTATUS(status)` คือตัวเลข $0-255$
* **`WIFSIGNALED(status)`:** โปรเซสถูกสั่งฆ่ากะทันหันด้วยสัญญาณจากระบบ (เช่น โดนกด `Ctrl+C` ซึ่งส่งสัญญาณ `SIGINT` หรือโดน `SIGKILL`) มาตรฐาน POSIX กำหนดให้ค่า `$?` ต้องเท่ากับ **$128 + \text{หมายเลขสัญญาณ}$**  
  เช่น สัญญาณ `SIGINT` มีหมายเลข 2 ดังนั้นเมื่อโปรแกรมโดนตัดจบ สถานะจะต้องเป็น $128 + 2 = 130$

---

## ⚡ 4. การข้าม `fork()` สำหรับ Builtin Commands (Fast Path)

การเรียก `fork()` แม้ว่าระบบปฏิบัติการสมัยใหม่จะใช้เทคนิค **Copy-on-Write (COW)** แต่ก็ยังมีค่าใช้จ่ายในการสลับบริบท (Context Switch) และการจัดสรร Page Table ในเคอร์เนล

สำหรับคำสั่งที่เป็น Builtin ของเชลล์ เช่น `cd`, `export`, `set`, `read` และ `echo` เราไม่ต้องการสร้าง Process ใหม่โดยไม่จำเป็น  
ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 920 ถึง 947:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 920 - 947) */
    /* จัดการคำสั่งที่เป็น Builtin ในตัว */
    if (zbash_is_builtin(effective_argv[0])) {
        /* กรณีที่ไม่มีการเปลี่ยนทิศทาง I/O (No Redirections): รันตรงในกระบวนการหลักทันที! */
        if (!cmd->in_file && !cmd->out_file && !cmd->err_file && !cmd->err_to_out && !cmd->out_to_err) {
            return zbash_exec_builtin(state, effective_argc, effective_argv);
        }

        /* กรณีที่มี Redirections เช่น echo "hello" > file.txt */
        int saved_in = dup(STDIN_FILENO);
        int saved_out = dup(STDOUT_FILENO);
        int saved_err = dup(STDERR_FILENO);

        if (apply_redirections(cmd) != 0) {
            close(saved_in);
            close(saved_out);
            close(saved_err);
            return 1;
        }
        int ret = zbash_exec_builtin(state, effective_argc, effective_argv);
        fflush(stdout);
        fflush(stderr);

        /* กู้คืน File Descriptors เดิมกลับมายัง Terminal */
        dup2(saved_in, STDIN_FILENO);
        dup2(saved_out, STDOUT_FILENO);
        dup2(saved_err, STDERR_FILENO);
        close(saved_in);
        close(saved_out);
        close(saved_err);
        return ret;
    }
```

### ทำไมสถาปัตยกรรมนี้ถึงทรงพลัง?
1. **ความเร็วระดับความเร็วแสง (Zero Fork Overhead):** คำสั่งอย่าง `x=1` หรือ `cd /tmp` หรือ `echo $count` ที่วนลูปเป็นล้านครั้ง สามารถประมวลผลเสร็จใน L1/L2 Cache โดยไม่มี System Call สร้างโปรเซสแม้แต่ครั้งเดียว
2. **การกู้คืน I/O อย่างปลอดภัย:** หาก Builtin มีการเปลี่ยนทิศทางไฟล์ เช่น `echo "data" > out.log` เราใช้ `dup()` เพื่อจำ Descriptor 0, 1, 2 เดิมไว้ก่อน จากนั้นรันฟังก์ชัน Builtin ล้างบัฟเฟอร์ด้วย `fflush()` แล้วใช้ `dup2()` ดึงท่อเดิมกลับมา ทำให้เชลล์หลักยังคงเชื่อมต่อกับ Terminal หน้าจอได้เหมือนเดิมอย่างสมบูรณ์แบบ

---

## 🌊 5. ท่อไปป์หลายขั้นตอน: การสร้าง Pipeline Process Group

เมื่อผู้ใช้สั่งรัน Pipeline เช่น:
```bash
cat large.txt | grep "error" | wc -l
```
เชลล์ต้องสร้าง Child Process ขึ้นมาพร้อมกัน 3 กระบวนการ และต่อท่อ `pipe()` ข้ามระหว่างกระบวนการเหล่านั้น  
ดูโค้ดจริงใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 1007 ถึง 1089:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 1007 - 1089) */
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
            /* กระบวนการลูกตัวที่ i */
            if (i > 0) {
                dup2(pipe_fds[i - 1][0], STDIN_FILENO);   /* ท่อขาเข้าจากคำสั่งก่อนหน้า */
            }
            if (i < n - 1) {
                dup2(pipe_fds[i][1], STDOUT_FILENO);     /* ท่อขาออกส่งต่อให้คำสั่งถัดไป */
            }

            /* สำคัญที่สุด: ปิดปลายท่อทั้งหมดที่ไม่ได้ใช้ในตัวลูก! */
            for (int j = 0; j < n - 1; j++) {
                close(pipe_fds[j][0]);
                close(pipe_fds[j][1]);
            }

            if (apply_redirections(cmd) != 0) {
                _exit(1);
            }

            /* สั่งรันคำสั่งเป้าหมาย */
            execvp(cmd->argv[0], cmd->argv);
            fprintf(stderr, "zbash: %s: %s\n", cmd->argv[0], strerror(errno));
            _exit((errno == ENOENT) ? 127 : 126);
        } else if (pids[i] < 0) {
            perror("zbash: fork");
            return 1;
        }
    }

    /* สำคัญที่สุด: Parent Process ต้องปิดปลายท่อทั้งหมดทันที! */
    for (int j = 0; j < n - 1; j++) {
        close(pipe_fds[j][0]);
        close(pipe_fds[j][1]);
    }

    /* วนลูปเก็บเกี่ยว (Reaping) โปรเซสลูกทั้งหมดเพื่อป้องกัน Zombie */
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

### กฎข้อห้ามอันตราย: การปิดท่อใน Parent
หาก Parent Process ลืมสั่งปิดปลายท่อ `close(pipe_fds[j][1])` หลังจากสร้างลูกเสร็จสิ้น กระบวนการตัวรับ (เช่น `wc -l`) จะ **ไม่มีวันได้รับสัญญาณสิ้นสุดไฟล์ (End-of-File / EOF)** เพราะเคอร์เนลเห็นว่ายังมีตัวส่ง (คือ Parent) ถือปลายท่อเขียนเปิดค้างอยู่ ผลลัพธ์คือ **เชลล์จะค้างสนิท (Hang) ไปตลอดกาล!**  
การปิดท่อทั้งใน Child และ Parent อย่างเคร่งครัดตามโค้ดด้านบน จึงเป็นหัวใจสำคัญที่ทำให้ `zbash` ไหลลื่นและเสถียรภาพสูงสุด

---

## 🧪 6. การทดสอบยืนยันความถูกต้องระดับ Parity (Differential Proof)

ในชุดทดสอบ `test/1000-test/test_07_pipelines_redirections.sh` และชุดทดสอบกระบวนการ เราตรวจสอบพฤติกรรมของ Process Lifecycle เทียบกับ macOS Bash:

| สถานการณ์ทดสอบ | คำสั่งทดสอบ | Exit Code ที่ได้ | ผลการเปรียบเทียบ |
| :--- | :--- | :--- | :--- |
| **Command Not Found** | `non_existent_command_xyz` | `127` | **Exact Match (100%)** |
| **Permission Denied** | `/tmp/no_exec_file.sh` (mode 0644) | `126` | **Exact Match (100%)** |
| **Signal Interruption** | `kill -9 $$` หรือ `kill -TERM` | `128 + SIG` | **Exact Match (100%)** |
| **Pipeline Exit Code** | `false \| true` vs `true \| false` | `0` vs `1` | **Exact Match (100%)** |
| **Zombie Prevention** | สร้างกระบวนการ 5,000 โปรเซสติดต่อกัน | ไม่มี Zombie ค้าง | **Exact Match (100%)** |

---

## 🎯 7. สรุปบทเรียนสำหรับนักพัฒนา

1. **เข้าใจสัญญาระหว่างระบบ (POSIX Contract):** การคืน Exit Code 127 เมื่อไม่พบคำสั่ง และ 126 เมื่อไม่มีสิทธิ์รัน รวมถึง $128 + \text{Signal}$ คือเครื่องหมายการค้าของซอฟต์แวร์ระดับโปรดักชัน
2. **รักษาความสะอาดของ File Descriptors:** ทุกท่อที่เปิดขึ้นมาด้วย `pipe()` หรือ `dup()` ต้องได้รับการปิด (`close()`) ในเวลาและลำดับที่ถูกต้องเสมอ
3. **เลี่ยง `fork()` เมื่อทำได้:** การรัน Builtin Command ในตัว Parent Process โดยตรง ช่วยประหยัดเวลาและทรัพยากรของ CPU ได้อย่างมหาศาล

ในบทที่ 11 เราจะเจาะลึกต่อไปในรายละเอียดของ **ท่อไปป์และการเปลี่ยนทิศทาง: `pipe()`, `dup2()`, และ `done < file`** เพื่อแกะรอยเทคนิคการส่งผ่านข้อมูลระดับเคอร์เนลและบั๊กการเปลี่ยนทิศทางท้ายลูปที่เราเอาชนะมาได้!
