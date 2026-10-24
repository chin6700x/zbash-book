# บทที่ 4: ความลับของ Lexer: เร่งความเร็วด้วย ARM64 NEON SIMD 128-bit
> **"Inside the Lexer: Blazing-Fast Scanning with 128-bit ARM64 NEON SIMD"**  
> **ผู้เขียน:** วิโรจน์ ชูแก้ว (Wirot Chookeaw - `chin6700x`)  
> **ไฟล์ต้นฉบับ:** `zbash/docs/book/th/ch04_simd_vector_lexer.md`

---

> [!NOTE]
> ### 🏛️ คติธรรมประจำบท
> **"SIMD is not an optimization; it is how modern silicon was meant to process data."**  
> — **Casey Muratori** (Handmade Hero)  
> *คำแปล:* "SIMD ไม่ใช่แค่เทคนิคการ Optimize แต่มันคือวิธีที่ชิปซิลิคอนยุคใหม่ถูกออกแบบมาเพื่อประมวลผลข้อมูลตั้งแต่แรก"
> 
> **"Do not fear the kernel. Do not fear assembly. They are merely bytes waiting for your instructions."**  
> — **วิโรจน์ ชูแก้ว (Wirot Chookeaw)**

---

## 🧭 1. ทำไม Lexer ทั่วไปถึงเป็นคอขวด?

ในกระบวนการทำงานของเชลล์ เมื่อผู้ใช้ป้อนคำสั่งหรือรันสคริปต์ขนาดใหญ่ สิ่งแรกที่ต้องทำงานหนักที่สุดคือ **Lexer (ตัวแยกคำ / Tokenizer)**  
งานของ Lexer คือการกวาดอ่านสายอักขระ (Character Stream) เพื่อ:
1. ข้ามช่องว่าง (Spaces) และแท็บ (Tabs) ที่คั่นระหว่างคำสั่ง
2. ตรวจหาจุดสิ้นสุดของคำ (Token Delimiters) เช่น ช่องว่าง, Newline (`\n`), ไปป์ (`|`), เซมิโคลอน (`;`), หรือเครื่องหมายคำพูด (`"`, `'`)

ในเชลล์ยุคดั้งเดิม การสแกนนี้ถูกเขียนด้วยการวนลูปอ่านทีละ 1 ไบต์ (Byte-by-Byte Scalar Loop):
```c
/* วิธีการดั้งเดิม: อ่านทีละ 1 ไบต์ (ช้าและเกิด Branch หลายรอบ) */
size_t skip_spaces_scalar(const char *str, size_t len) {
    size_t i = 0;
    while (i < len && (str[i] == ' ' || str[i] == '\t')) {
        i++;
    }
    return i;
}
```

หากสคริปต์มีการเยื้องหน้า (Indentation) หรือมีช่องว่างคั่น 32 ตัวอักษร ซีพียูต้องวนลูปอ่านและตรวจสอบเงื่อนไขถึง 32 รอบ ซึ่งบนชิป Apple Silicon (M-Series) ถือเป็นการใช้ทรัพยากรฮาร์ดแวร์ได้ไม่ถึง 10% ของศักยภาพจริง

ใน `zbash` เราตัดสินใจเขียนเคอร์เนลตัวแยกคำด้วย **ARM64 Assembly และชุดคำสั่ง NEON Vector SIMD (128-bit)** โดยตรงในไฟล์ [`src/asm/simd_lexer.s`](src/asm/simd_lexer.s) เพื่อให้สามารถประมวลผลข้อมูลได้ **16 ไบต์พร้อมกันใน 1 สัญญาณนาฬิกา**

---

## ⚡ 2. ผ่าโค้ด `zbash_simd_skip_spaces`: ข้ามช่องว่าง 16 ไบต์ใน 1 สัญญาณนาฬิกา

ลองมาเปิดดูโค้ดจริงในไฟล์ [`src/asm/simd_lexer.s`](src/asm/simd_lexer.s) บรรทัดที่ 22 ถึง 57:

```assembly
/* ไฟล์: src/asm/simd_lexer.s (บรรทัดที่ 22 - 57) */
.globl CSYM(zbash_simd_skip_spaces)
CSYM(zbash_simd_skip_spaces):
    cbz     x1, .Lskip_done_zero     // ถ้าความยาว len (x1) == 0 ให้รีเทิร์นทันที
    mov     x2, #0                   // x2 = ตัวนับตำแหน่ง (Offset)

    // เตรียมเวกเตอร์คงที่ 128-bit สำหรับ Space (0x20) และ Tab (0x09)
    mov     w3, #0x2020
    movk    w3, #0x2020, lsl #16
    dup     v4.4s, w3                // v4 = เวกเตอร์ 16 ไบต์ของ ' ' (0x20)

    mov     w3, #0x0909
    movk    w3, #0x0909, lsl #16
    dup     v5.4s, w3                // v5 = เวกเตอร์ 16 ไบต์ของ '\t' (0x09)

.Lskip_loop_16:
    sub     x3, x1, x2               // คำนวณความยาวที่เหลือ
    cmp     x3, #16
    b.lo    .Lskip_scalar            // ถ้าน้อยกว่า 16 ไบต์ ให้ไปประมวลผลแบบธรรมดา

    ldr     q0, [x0, x2]             // 🚀 โหลดข้อมูล 16 ไบต์รวดเดียวลงรีจิสเตอร์ q0 (128-bit)
    cmeq    v1.16b, v0.16b, v4.16b   // เปรียบเทียบ 16 ไบต์พร้อมกันกับ ' '
    cmeq    v2.16b, v0.16b, v5.16b   // เปรียบเทียบ 16 ไบต์พร้อมกันกับ '\t'
    orr     v1.16b, v1.16b, v2.16b   // v1 จะมีบิตเป็น 0xFF ในตำแหน่งที่เป็นช่องว่าง

    // ตรวจสอบว่าทั้ง 16 ไบต์เป็นช่องว่างทั้งหมดหรือไม่ใน 1 คำสั่ง!
    uminv   b3, v1.16b
    umov    w4, v3.b[0]
    cmp     w4, #0xFF
    b.ne    .Lskip_find_non_space    // ถ้ามีแม้แต่ 1 ไบต์ที่ไม่ใช่ช่องว่าง ให้กระโดดออก

    add     x2, x2, #16              // ข้ามไปข้างหน้าทันที 16 ไบต์
    b       .Lskip_loop_16
```

### เจาะลึกการทำงานของชุดคำสั่งระดับฮาร์ดแวร์:
1. **`dup v4.4s, w3` (Broadcast Register):**  
   คำสั่งนี้ทำการกระจายค่าไบต์ของช่องว่าง (`0x20`) ซ้ำๆ กัน 16 ครั้งลงในเวกเตอร์รีจิสเตอร์ 128-bit `v4` เพื่อเตรียมเป็นแม่แบบเปรียบเทียบ
2. **`ldr q0, [x0, x2]` (128-bit Vector Load):**  
   คำสั่งเดียวนี้โหลดอักขระ 16 ตัวจากหน่วยความจำ RAM หรือ L1 Cache ขึ้นมาอยู่ในรีจิสเตอร์ `q0` ทันที
3. **`cmeq` (Vector Compare Equal):**  
   เปรียบเทียบทั้ง 16 ไบต์แบบขนาน ไบต์ใดที่ตรงกับช่องว่าง ฮาร์ดแวร์จะเซ็ตค่าเป็น `0xFF` (ทุกบิตเป็น 1) ทันที
4. **`uminv b3, v1.16b` (Unsigned Minimum Across Vector):**  
   นี่คือ "คำสั่งหมัดเด็ด" ของ ARM64! มันหาค่าต่ำสุดจาก 16 เลน หากค่าต่ำสุดยังคงเป็น `0xFF` นั่นแปลว่า **ทั้ง 16 ไบต์เป็นช่องว่างทั้งหมด 100%** เราจึงบวก Offset ขึ้นทีละ 16 ได้ในครั้งเดียวโดยไม่ต้องตรวจสอบทีละตัว!

---

## 🔍 3. การหาตำแหน่งตัวอักษรที่ไม่ใช่ช่องว่าง (`.Lskip_find_non_space`)

หากในบล็อก 16 ไบต์นั้น มีตัวอักษรปกติโผล่ขึ้นมา เราจะหาตำแหน่งของมันได้อย่างไรโดยไม่เสียเวลา?  
ดูบรรทัดที่ 58 ถึง 74 ใน [`src/asm/simd_lexer.s`](src/asm/simd_lexer.s):

```assembly
/* ไฟล์: src/asm/simd_lexer.s (บรรทัดที่ 58 - 74) */
.Lskip_find_non_space:
    // มีอย่างน้อย 1 ไบต์ที่ไม่ใช่ช่องว่าง ให้สแกนเฉพาะเศษในบล็อก 16 ไบต์นี้
.Lskip_scalar:
    cmp     x2, x1
    b.hs    .Lskip_ret
    ldrb    w3, [x0, x2]
    cmp     w3, #' '
    b.eq    .Lskip_next
    cmp     w3, #'\t'
    b.ne    .Lskip_ret               // พบตัวอักษรแรกที่ไม่ใช่ช่องว่าง หยุดทันที!
.Lskip_next:
    add     x2, x2, #1
    b       .Lskip_scalar

.Lskip_ret:
    mov     x0, x2                   // ส่งค่า Offset ที่ข้ามได้กลับไปในรีจิสเตอร์ x0
    ret
```

* **Tail Fallback:** สำหรับเศษที่เหลือน้อยกว่า 16 ไบต์ หรือช่วงรอยต่อที่เจอตัวอักษรแรก ระบบจะใช้ Scalar Loop สแกนต่อเพียงไม่กี่รอบ ทำให้ได้ทั้งความเร็วระดับเวกเตอร์และความแม่นยำระดับบิตตรงกัน 100%

---

## 🌉 4. การเชื่อมต่อระหว่าง C และ Assembly ใน `src/parser/lexer.c`

ในฝั่งภาษา C เราเรียกใช้ฟังก์ชัน Assembly นี้อย่างไร?  
ลองเปิดดูไฟล์ [`include/zbash.h`](include/zbash.h) บรรทัดที่ 128 ถึง 132:

```c
/* ไฟล์: include/zbash.h (บรรทัดที่ 128 - 132) */
#if defined(__arm64__) || defined(__aarch64__)
size_t zbash_simd_skip_spaces(const char *str, size_t len);
size_t zbash_simd_scan_token(const char *str, size_t len);
#endif
```

และในโค้ด C ของตัวตัดคำ เราใช้ Preprocessor ป้องกันเพื่อให้โค้ดคอมไพล์ได้บนทุกสถาปัตยกรรม:
```c
static size_t skip_whitespace(const char *str, size_t len) {
#if defined(ZBASH_USE_NEON)
    return zbash_simd_skip_spaces(str, len);
#else
    /* Fallback แบบพกพาสำหรับซีพียู x86 หรือ WebAssembly */
    size_t i = 0;
    while (i < len && (str[i] == ' ' || str[i] == '\t')) i++;
    return i;
#endif
}
```

* **Zero Abstraction Overhead:** บน Apple Silicon (ARM64) ตัวคอมไพเลอร์จะเรียกฟังก์ชันใน `simd_lexer.s` โดยตรงผ่านการส่งพารามิเตอร์ในรีจิสเตอร์ `x0` และ `x1` ตามมาตรฐาน ARM64 ABI โดยไม่มีต้นทุนการแปลงข้อมูลใดๆ ทั้งสิ้น

---

## 🏁 สรุปบทเรียนที่ 4

1. **ฮาร์ดแวร์ยุคใหม่มีพลังมหาศาล:** รีจิสเตอร์ 128-bit ของชิป ARM64 สามารถกวาดอ่านข้อมูลได้ 16 ไบต์ต่อคำสั่ง
2. **การลด Branch Misprediction:** การข้ามบล็อกละ 16 ไบต์ช่วยลดจำนวนคำสั่ง Jump ในซีพียูลงถึง 93%
3. **การออกแบบที่ยืดหยุ่น:** แม้แกนหลักจะใช้ Pure Assembly เพื่อความเร็วสูงสุดบน Mac แต่ก็มี Fallback C รองรับเมื่อต้องคอมไพล์ไปเป็น WebAssembly (WASM)

ใน **บทที่ 5** เราจะมาเผชิญหน้ากับความท้าทายที่ยากที่สุดของคนเขียน Shell Parser: **"ปราบพยศไวยากรณ์: แกะรอย Quoting Hell (", ', \) และ Tokenizer"** เพื่อดูว่าเราจัดการเครื่องหมายคำพูดที่ซ้อนกันอย่างบ้าคลั่งได้อย่างไรครับ! 🚀📖
