# Chapter 4: The Lexer Secret: 128-bit ARM64 NEON SIMD Acceleration
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch04_simd_vector_lexer.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"SIMD is not an optimization; it is how modern silicon was meant to process data."**  
> — **Casey Muratori** (Handmade Hero)  
> 
> **"Do not fear the kernel. Do not fear assembly. They are merely bytes waiting for your instructions."**  
> — **Wirot Chookeaw**

---

## 🧭 1. Why Conventional Lexers Become Bottlenecks

In shell execution pipelines, whenever an interactive command or large script is ingested, the primary workload initially falls on the **Lexer (Tokenizer)**.  
The responsibilities of the lexer include:
1. Skipping consecutive whitespace (spaces and tabs) separating command tokens.
2. Detecting token boundaries and delimiters, including whitespace, newlines (`\n`), pipes (`|`), semicolons (`;`), and quotes (`"`, `'`).

In traditional shells, lexical scanning is implemented using byte-by-byte scalar loops:

```c
/* Conventional scalar scanning: one byte at a time */
size_t skip_spaces_scalar(const char *str, size_t len) {
    size_t i = 0;
    while (i < len && (str[i] == ' ' || str[i] == '\t')) {
        i++;
    }
    return i;
}
```

If an automated script contains 32 characters of indentation or whitespace, the CPU executes 32 discrete branch comparisons and increment instructions. On modern Apple Silicon architectures, this scalar approach uses less than 10% of available vector execution bandwidth.

In `zbash`, the lexical scanner is implemented directly in **ARM64 Assembly using 128-bit NEON SIMD instructions** in [`src/asm/simd_lexer.s`](src/asm/simd_lexer.s), processing **16 bytes simultaneously in a single clock cycle**.

---

## ⚡ 2. Dissecting `zbash_simd_skip_spaces`: 16-Byte Acceleration

Consider the assembly implementation in [`src/asm/simd_lexer.s`](src/asm/simd_lexer.s#L22-L57):

```assembly
/* File: src/asm/simd_lexer.s (lines 22 - 57) */
.globl CSYM(zbash_simd_skip_spaces)
CSYM(zbash_simd_skip_spaces):
    cbz     x1, .Lskip_done_zero     // Return immediately if length (x1) == 0
    mov     x2, #0                   // x2 = current byte offset

    // Prepare 128-bit vector constants for Space (0x20) and Tab (0x09)
    mov     w3, #0x2020
    movk    w3, #0x2020, lsl #16
    dup     v4.4s, w3                // v4 = 16-byte vector of ' ' (0x20)

    mov     w3, #0x0909
    movk    w3, #0x0909, lsl #16
    dup     v5.4s, w3                // v5 = 16-byte vector of '\t' (0x09)

.Lskip_loop_16:
    sub     x3, x1, x2               // Remaining byte count
    cmp     x3, #16
    b.lo    .Lskip_scalar            // If less than 16 bytes remain, process via scalar tail

    ldr     q0, [x0, x2]             // Load 16 consecutive bytes into 128-bit register q0
    cmeq    v1.16b, v0.16b, v4.16b   // Compare 16 bytes simultaneously against ' '
    cmeq    v2.16b, v0.16b, v5.16b   // Compare 16 bytes simultaneously against '\t'
    orr     v1.16b, v1.16b, v2.16b   // v1 lane is set to 0xFF wherever whitespace matched

    // Evaluate whether all 16 bytes are whitespace in a single instruction
    uminv   b3, v1.16b
    umov    w4, v3.b[0]
    cmp     w4, #0xFF
    b.ne    .Lskip_find_non_space    // If any byte is non-whitespace, branch out

    add     x2, x2, #16              // Advance offset by 16 bytes
    b       .Lskip_loop_16
```

### Hardware Instruction Analysis:
1. **`dup v4.4s, w3` (Broadcast Register):**  
   Replicates the space character (`0x20`) across all sixteen byte lanes of the 128-bit vector register `v4` as a reference mask.
2. **`ldr q0, [x0, x2]` (128-bit Vector Load):**  
   Fetches 16 consecutive characters from system memory or L1 cache into vector register `q0` in a single operation.
3. **`cmeq` (Vector Compare Equal):**  
   Executes 16 byte comparisons in parallel. Lanes matching space or tab are set to `0xFF`.
4. **`uminv b3, v1.16b` (Unsigned Minimum Vector Reduction):**  
   Finds the minimum byte across all 16 lanes. If the minimum is `0xFF`, all 16 characters are guaranteed whitespace, allowing the engine to skip the entire 16-byte chunk without scalar branch overhead.

---

## 🔍 3. Pinpointing Non-Whitespace Transitions (`.Lskip_find_non_space`)

When a 16-byte vector contains non-whitespace characters, execution branches to find the exact transition boundary:

```assembly
/* File: src/asm/simd_lexer.s (lines 58 - 74) */
.Lskip_find_non_space:
    // At least one byte is non-whitespace; scan the remaining elements in this chunk
.Lskip_scalar:
    cmp     x2, x1
    b.hs    .Lskip_ret
    ldrb    w3, [x0, x2]
    cmp     w3, #' '
    b.eq    .Lskip_next
    cmp     w3, #'\t'
    b.ne    .Lskip_ret               // First non-whitespace byte encountered; return!
.Lskip_next:
    add     x2, x2, #1
    b       .Lskip_scalar

.Lskip_ret:
    mov     x0, x2                   // Return scanned offset in register x0
    ret
```

* **Tail Fallback:** Short inputs under 16 bytes or chunk boundary transitions transition smoothly into a compact scalar loop, guaranteeing vector speed on bulk text and 100% precision on arbitrary string lengths.

---

## 🌉 4. Bridging C and Assembly in `src/parser/lexer.c`

In the C subsystem, the assembly function is declared and integrated via [`include/zbash.h`](include/zbash.h#L128-L132):

```c
/* File: include/zbash.h (lines 128 - 132) */
#if defined(__arm64__) || defined(__aarch64__)
size_t zbash_simd_skip_spaces(const char *str, size_t len);
size_t zbash_simd_scan_token(const char *str, size_t len);
#endif
```

Within the lexical analyzer, preprocessor directives enable transparent compilation across all targets:

```c
static size_t skip_whitespace(const char *str, size_t len) {
#if defined(ZBASH_USE_NEON)
    return zbash_simd_skip_spaces(str, len);
#else
    /* Portable fallback for x86 and WebAssembly */
    size_t i = 0;
    while (i < len && (str[i] == ' ' || str[i] == '\t')) i++;
    return i;
#endif
}
```

* **Zero Abstraction Overhead:** Under the standard ARM64 Procedure Call Standard (AAPCS64), arguments pass directly in registers `x0` and `x1`, and the result returns in `x0`, incurring zero conversion overhead.

---

## 🏁 Summary

1. **Modern Hardware Capability:** 128-bit NEON registers enable processing 16 bytes per instruction cycle.
2. **Branch Reduction:** Bulk 16-byte steps eliminate up to 93% of branch evaluations compared to scalar scanning.
3. **Cross-Platform Resilience:** While native assembly delivers maximum throughput on Apple Silicon, portable C fallbacks ensure full compatibility for WebAssembly (WASM) and alternative CPU architectures.

In **[Chapter 5: Taming the Grammar: Deconstructing Quoting Hell and Tokenization](ch05_grammar_quoting_hell.md)**, we address one of the most intricate challenges in shell parser design: managing nested quotes (`"`, `'`), backslash escapes, and variable substitutions without buffer overflows or heap fragmentation.
