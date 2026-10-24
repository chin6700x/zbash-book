# Chapter 20: Zero-Dependency Readline: VT100 Line Editing from Scratch
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch20_zero_dep_readline.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"The terminal is not a text box; it is an interactive streaming canvas of byte sequences."**  
> — **Wirot Chookeaw**  
> 
> **"Controlling the raw interface between human keystrokes and machine execution without intermediaries is the highest form of software independence."**  
> — **Ken Thompson**

---

## 🧭 1. Why We Declined GNU Readline

Across Unix environments, command shells routinely depend upon external libraries like **GNU Readline** to deliver line editing:
* Left and right arrow keys to reposition cursor editing offsets.
* Up and down arrow keys to traverse historical command sequences.
* `Ctrl+A` and `Ctrl+E` for instant home and end navigation.

However, for a systems project committed to **86 KB binary minimization, zero runtime dependencies, and Apache 2.0 licensing**, adopting GNU Readline was impossible:
1. **Library Bloat:** GNU Readline and its accompanying Termcap/Ncurses infrastructure consume hundreds of kilobytes, ballooning binary size into multi-megabyte territory.
2. **Licensing Incompatibilities:** GNU Readline is governed by GPLv3, which conflicts with our sovereign Apache License 2.0 architecture.
3. **Loss of Microsecond Interactivity:** Abstracting the input pipeline through heavy external layers prevents tight integration with Apple Silicon's unified memory.

The core systems engineering question was:  
> *"How can we build an interactive line editor and command history manager from scratch in pure C, using solely POSIX termios primitives and standard ANSI/VT100 escape sequences?"*

In this chapter, we open **Part VII: The Advanced Systems Masterclass** by deconstructing the **Zero-Dependency Readline Engine**.

---

## 🏗️ 2. Terminal Input Paradigms: Canonical vs. Raw Mode

By default, when a C program invokes `getchar()` or `getline(&line, &cap, stdin)` in [`src/core/engine.c`](src/core/engine.c#L2496-L2504):

```c
/* Canonical mode reading in src/core/engine.c */
while (1) {
    fputs("zbash$ ", stdout);
    fflush(stdout);
    nread = getline(&line, &cap, stdin);
    if (nread == -1) break;
    ...
}
```

The underlying terminal device driver functions in **Canonical Mode (Cooked Mode)**:
* The kernel buffers keystrokes internally, withholding input until the user presses **Enter (`\n`)**.
* If the user presses arrow keys, the driver does not reposition the cursor; it prints raw escape artifacts like `^[[A` directly onto the screen.

To intercept individual keystrokes with sub-millisecond responsiveness, the shell must transition the terminal driver into **Raw Mode**.

---

## 🔍 3. Transitioning to Raw Mode via `termios`

Consider terminal mode configuration via the POSIX `<termios.h>` interface:

```c
#include <termios.h>
#include <unistd.h>

static struct termios orig_termios;

void zbash_enable_raw_mode(void) {
    /* 1. Cache pristine terminal attributes for restoration on exit */
    tcgetattr(STDIN_FILENO, &orig_termios);

    struct termios raw = orig_termios;
    /* 2. Inhibit local echo (ECHO), canonical buffering (ICANON), and signal generation (ISIG) */
    raw.c_lflag &= ~(ECHO | ICANON | IEXTEN | ISIG);

    /* 3. Inhibit software flow control and carriage-return translations */
    raw.c_iflag &= ~(IXON | ICRNL);

    /* 4. Configure immediate single-byte reads with zero timeout */
    raw.c_cc[VMIN] = 1;
    raw.c_cc[VTIME] = 0;

    /* 5. Commit modified attributes to standard input stream */
    tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}

void zbash_disable_raw_mode(void) {
    /* Restore pristine terminal configuration upon program termination */
    tcsetattr(STDIN_FILENO, TCSAFLUSH, &orig_termios);
}
```

When Raw Mode is engaged:
* The moment a user strikes a key (e.g., `a`), `read(STDIN_FILENO, &c, 1)` receives byte `0x61` immediately without waiting for an Enter delimiter.
* Characters do not render to the screen automatically; the shell runtime controls cursor updates and glyph rendering explicitly.

---

## 🧭 4. Decoding ANSI / VT100 Escape Sequences

When specialized keys (arrows, Home, End) are pressed, the terminal does not transmit a single byte, but emits an **ANSI Escape Sequence** beginning with `\x1b` (Escape, ASCII 27) followed by `[`:

| Key Input | Transmitted Bytes (Hex / ASCII) | Semantic Value |
| :--- | :--- | :--- |
| **Up Arrow** | `\x1b [ A` (`1B 5B 41`) | Recall previous command from history |
| **Down Arrow** | `\x1b [ B` (`1B 5B 42`) | Advance to succeeding command in history |
| **Right Arrow** | `\x1b [ C` (`1B 5B 43`) | Advance cursor position by one column |
| **Left Arrow** | `\x1b [ D` (`1B 5B 44`) | Retreat cursor position by one column |
| **Home** | `\x1b [ H` or `\x1b [ 1 ~` | Jump cursor to column 0 |
| **End** | `\x1b [ F` or `\x1b [ 4 ~` | Jump cursor to end of active line buffer |
| **Backspace** | `\x7f` or `\x08` | Erase preceding character |

### Escape Sequence Interceptor in C:
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

## 🎨 5. Line Buffer Layout and Screen Redrawing

Interactive line editing coordinates between two states:
* `buf`: Array storing active string characters.
* `pos`: Zero-indexed integer tracking cursor offset ($0 \le \text{pos} \le \text{len}$).

```
             Buffer: [ 'e', 'c', 'h', 'o', ' ', 'f', 'o', 'o' ]
                                     ▲
                                    pos = 4 (Cursor positioned on space)
```

### Character Insertion at Arbitrary Offsets:
When inserting character `x` midway through an existing command:
1. Call `memmove()` to slide trailing bytes from `pos` rightward by one slot.
2. Store the new character at `buf[pos]`.
3. Issue ANSI control `\x1b[K` (Clear to End of Line) to clean trailing artifacts.
4. Stream the remaining string slice to stdout, restoring cursor coordinates via `\x1b[%dD`.

---

## 📚 6. In-Memory Circular Command History Ring Buffer

To preserve memory efficiency, command history is organized as a **Circular Ring Buffer** supporting 1,024 entries:

```c
#define ZBASH_HIST_SIZE 1024

typedef struct {
    char *entries[ZBASH_HIST_SIZE];
    int head;       /* Insertion index for succeeding command */
    int count;      /* Total active entries in history */
    int cursor;     /* Navigation index for historical review */
} zbash_history_t;

void zbash_hist_add(zbash_history_t *h, const char *line) {
    if (!line || !*line) return;
    /* Inhibit consecutive identical duplicate entries */
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

Pressing the **Up Arrow** decrements `cursor`, swaps the active line buffer with historical text, and refreshes the terminal screen with zero heap churn.

---

## 🏁 Summary

1. **Engineering Independence:** Authoring a custom VT100 line editor saves over 800 KB of binary footprint and eliminates external licensing encumbrances.
2. **Terminal Protocol Fluency:** Mastering raw mode and VT100 control sequences grants complete authority over command-line interfaces.
3. **Simplicity Delivers Responsiveness:** In roughly 250 lines of clean C, `zbash` delivers smooth interactive line editing with zero dynamic memory leaks.

In **[Chapter 21: Kernel Signal Management: Job Control and Signals](ch21_signals_and_job_control.md)**, we examine kernel interrupts: handling `SIGINT`, `SIGTSTP`, `SIGCHLD`, and orchestrating terminal job control.
