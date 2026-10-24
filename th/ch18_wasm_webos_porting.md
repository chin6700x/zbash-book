# บทที่ 18: จาก Terminal สู่เบราว์เซอร์: พอร์ต zbash สู่ WebAssembly & WebOS
> **"From Bare-Metal Silicon to the Browser: Porting zbash to WebAssembly and WebOS"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch18_wasm_webos_porting.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"Always bet on the Web. The boundaries of the browser are expanding to engulf the operating system itself."**  
> — **Brendan Eich** (ผู้สร้างภาษา JavaScript)  
> *คำแปล:* "จงเชื่อมั่นในเว็บ ขอบเขตของเบราว์เซอร์กำลังขยายตัวออกไปจนกลืนกินระบบปฏิบัติการเข้ามาอยู่ข้างใน"
> 
> **"Code written in pure, standards-compliant C has no boundaries—from microcontrollers to bare-metal silicon, and from local terminals to WebAssembly in the cloud."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**  
> *คำแปล:* "โค้ดที่เขียนด้วยภาษา C อันบริสุทธิ์และเคารพมาตรฐานนั้นไร้ซึ่งพรมแดน—จากไมโครคอนโทรลเลอร์สู่ชิป Apple Silicon และจาก Terminal ท้องถิ่นสู่ WebAssembly บนก้อนเมฆ"

---

## 🧭 1. พรมแดนใหม่ของระบบปฏิบัติการ: เว็บบราวเซอร์

ตลอด 17 บทที่ผ่านมา เราได้เห็นการสร้าง `zbash` บนสถาปัตยกรรม Bare-Metal POSIX และ Apple Silicon ARM64 ซึ่งพึ่งพา Kernel System Calls อย่าง `fork()`, `execve()`, `pipe()`, และ `dup2()` อย่างลึกซึ้ง

แต่คำถามที่ท้าทายจินตนาการทางวิศวกรรมที่สุดคือ:
> *"จะเป็นไปได้หรือไม่ หากเราจะนำ Shell ตัวเดิมนี้ ข้ามมิติไปรันอยู่บนเว็บเบราว์เซอร์ (Chrome, Safari, Firefox) โดยไม่ต้องติดตั้งโปรแกรมใดๆ บนเครื่องคอมพิวเตอร์ของผู้ใช้เลย?"*

คำตอบคือ **เป็นไปได้อย่างสมบูรณ์แบบ ผ่านเทคโนโลยี WebAssembly (WASM)!**

ในบทนี้ เราจะมาแกะรอยการสร้างสะพานเชื่อมมิติ (Bridge) ใน `zbash/wasm/` เพื่อแปลงโค้ดภาษา C ของเราให้กลายเป็น WASM Binary และพาเชลล์ของเราไปโลดแล่นเป็นหัวใจหลักของ **WebOS** บนเบราว์เซอร์!

---

## 🏗️ 2. ความท้าทายอันยิ่งใหญ่: โลกที่ไร้ `fork()`

สำหรับนักพัฒนาทั่วไป การนำโปรแกรม C ไปคอมไพล์ด้วย Emscripten อาจดูเหมือนแค่การเปลี่ยนตัวคอมไพเลอร์  
ทว่าสำหรับ **Unix Shell** สภาพแวดล้อม Sandbox ของเบราว์เซอร์คือดินแดนที่ขัดแย้งกับธรรมชาติของเชลล์อย่างสิ้นเชิง:
1. **เบราว์เซอร์ไม่มี `fork()`:** ความปลอดภัยของเว็บไม่อนุญาตให้แท็บเบราว์เซอร์สร้าง Process ใหม่ในระบบปฏิบัติการได้
2. **เบราว์เซอร์ไม่มีฮาร์ดแวร์ Pipe:** ไม่มีทางเดิน I/O ระหว่างกระบวนการในระดับเคอร์เนล
3. **เบราว์เซอร์ไม่มีระบบไฟล์จริง:** มีเพียง Virtual Memory File System (MEMFS) หรือ Origin Private File System (OPFS)

หากต้องการให้ `zbash` รันบนเว็บได้ เราต้อง **สร้างสถาปัตยกรรมแบบไฮบริด (Hybrid Execution Architecture)** ที่ปรับเปลี่ยนกลยุทธ์การทำงานตามสภาพแวดล้อมโดยอัตโนมัติ!

---

## 🔍 3. การจำลอง Pipeline บน WASM ด้วย Virtual Storage

ดูวิธีแก้ปัญหาใน [`src/core/engine.c`](src/core/engine.c) บรรทัดที่ 978 ถึง 1005:

```c
/* ไฟล์: src/core/engine.c (บรรทัดที่ 978 - 1005) */
#if defined(__EMSCRIPTEN__)
    /* สถาปัตยกรรม WASM: จำลองท่อไปป์ผ่าน Temporary File Storage ใน Memory */
    char prev_tmp[64] = {0};
    int last_st = 0;

    for (int i = 0; i < n; i++) {
        zbash_cmd_t *cmd = &pipeline->cmds[i];
        char next_tmp[64] = {0};

        /* หากไม่ใช่คำสั่งแรก ให้อ่านอินพุตจากเอาต์พุตของคำสั่งก่อนหน้า */
        if (i > 0) {
            cmd->in_file = strdup(prev_tmp);
        }
        /* หากไม่ใช่คำสั่งสุดท้าย ให้เปลี่ยนเอาต์พุตไปลงไฟล์ชั่วคราวถัดไป */
        if (i < n - 1) {
            snprintf(next_tmp, sizeof(next_tmp), "/tmp/wasm_pipe_%d.tmp", i);
            cmd->out_file = strdup(next_tmp);
        }

        /* รันคำสั่งตรงๆ ใน In-Memory Virtual Machine */
        last_st = execute_single_cmd(state, cmd);

        /* ทำลายไฟล์ชั่วคราวท่อนก่อนหน้าทิ้ง */
        if (prev_tmp[0]) unlink(prev_tmp);
        if (next_tmp[0]) strcpy(prev_tmp, next_tmp);
    }
    if (prev_tmp[0]) unlink(prev_tmp);
    return last_st;
#else
    /* สถาปัตยกรรม Native: ใช้ pipe() และ fork() แท้ของระบบ */
    ...
#endif
```

### ความแยบยลของกลยุทธ์นี้:
บนเบราว์เซอร์ แทนที่เราจะพยายามเลียนแบบ `fork()` ที่เป็นไปไม่ได้ เราเปลี่ยนมาใช้เทคนิค **Sequential Temporary Stream Piping**:
1. คำสั่งที่ 1 รันแล้วเขียนข้อมูลลงไฟล์เสมือน `/tmp/wasm_pipe_0.tmp` บนหน่วยความจำ (MEMFS)
2. คำสั่งที่ 2 รันต่อโดยอ่านข้อมูลจากไฟล์นั้น แล้วเขียนผลลัพธ์ลง `/tmp/wasm_pipe_1.tmp`
3. คำสั่งที่ 3 อ่านต่อและแสดงผลลัพธ์ออกหน้าจอ
4. สั่ง `unlink()` ทำลายไฟล์เสมือนทั้งหมดทิ้งทันที

ผลลัพธ์คือ: คำสั่งอย่าง `echo "hello" | tr 'a-z' 'A-Z' | wc -c` สามารถทำงานบนเบราว์เซอร์ได้ผลลัพธ์ตรงกับ Terminal บนเครื่องจริงแบบ 100%!

---

## ⚡ 4. สะพานเชื่อมระหว่าง C และ JavaScript (`wasm_bridge.c`)

ดูโค้ดจริงใน [`zbash/wasm/src/wasm_bridge.c`](wasm/src/wasm_bridge.c) บรรทัดที่ 30 ถึง 60:

```c
/* ไฟล์: zbash/wasm/src/wasm_bridge.c */
/* ส่งข้อความจาก C Engine ไปแสดงผลบน Terminal ของหน้าเว็บ */
EM_JS(void, js_on_terminal_output, (const char* str, int is_err), {
    if (typeof window !== 'undefined' && window.zbashOnOutput) {
        window.zbashOnOutput(UTF8ToString(str), is_err === 1);
    }
});

/* ส่งคำสั่งที่ C Engine ไม่รู้จัก (เช่น npm, git) ไปให้ JavaScript จัดการ */
EM_JS(int, js_on_external_command, (const char* cmd_name, const char* full_args), {
    if (typeof window !== 'undefined' && window.zbashOnExternal) {
        return window.zbashOnExternal(UTF8ToString(cmd_name), UTF8ToString(full_args));
    }
    return 127;
});

/* ฟังก์ชันหลักที่หน้าเว็บ JavaScript เรียกใช้เพื่อรันสคริปต์ */
EMSCRIPTEN_KEEPALIVE
int zbash_wasm_eval(const char *cmd_line) {
    if (!g_initialized) {
        zbash_init(&g_wasm_state);
        g_initialized = true;
    }
    return zbash_run_line(&g_wasm_state, cmd_line);
}
```

ด้วยโค้ดสะพานเพียงไม่กี่สิบบรรทัด ฝั่งหน้าบ้าน (Frontend) ใน JavaScript สามารถสั่งรันคำสั่งเชลล์ได้ง่ายดายเพียง:
```javascript
// ฝั่ง JavaScript บนเบราว์เซอร์
const exitCode = Module._zbash_wasm_eval(allocateUTF8("x=5; echo $(( x * 10 ))"));
```

---

## 🖥️ 5. สถาปัตยกรรม WebOS Desktop ในเบราว์เซอร์

ในโฟลเดอร์ [`zbash/wasm/webos/`](wasm/webos/) เราได้สร้างระบบปฏิบัติการจำลองบนเว็บ (WebOS) ขึ้นมาอย่างสมบูรณ์แบบ:
* **Terminal App:** หน้าต่างเทอร์มินัลสไตล์เรโทร รองรับสี ANSI, Cursor Navigation, และ History
* **App Finder / File Manager:** สำรวจไฟล์และโฟลเดอร์ใน Virtual File System
* **Desktop & Window Manager:** จัดการหน้าต่าง Drag & Drop, ย่อ/ขยายหน้าต่าง

ผู้ใช้สามารถเขียนสคริปต์, สร้างไฟล์, ทำการวนลูป และคำนวณคณิตศาสตร์ได้โดยตรงจากเบราว์เซอร์บนมือถือหรือแท็บเล็ต โดยที่เบื้องหลังการประมวลผลทั้งหมดถูกขับเคลื่อนด้วย **Engine ภาษา C ตัวเดียวกับที่รันบน Apple Silicon!**

---

## 🎯 6. สรุปบทเรียนสำหรับนักพัฒนา

1. **เขียนโค้ดตามมาตรฐาน ANSI C เพื่ออนาคต:** การหลีกเลี่ยงการพึ่งพาไลบรารีเฉพาะแพลตฟอร์ม ทำให้โค้ดของเรามีความยืดหยุ่นที่จะเดินทางไปรันบนแพลตฟอร์มใดๆ ในโลก รวมถึง WebAssembly
2. **ปรับกลยุทธ์ตามสภาพแวดล้อม (Graceful Degradation):** หากสภาพแวดล้อมไม่มี `fork()` จงอย่าล้มเลิก แต่จงออกแบบทางเดินข้อมูลแบบเสมือนขึ้นมาทดแทน
3. **เว็บคือผืนผ้าใบไร้ขีดจำกัด:** การนำระบบระดับล่างมาอยู่บนเว็บ ช่วยเปิดโอกาสให้ผู้คนนับล้านเข้าถึงและเรียนรู้เทคโนโลยีของเราได้ในคลิกเดียว

และในบทสุดท้าย **บทที่ 19: บทส่งท้าย: ปลดปล่อยพลังแห่งการสร้างซอฟต์แวร์ระดับโลก** เราจะมารวบรวมบทเรียนชีวิต จิตวิญญาณของผู้ประพันธ์ และส่งมอบคบเพลิงแห่งปัญญาแด่นักพัฒนารุ่นต่อไป!
