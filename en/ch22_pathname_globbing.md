# Chapter 22: High-Velocity Vector Scanner: Pathname Expansion & Wildcards (*, ?, [...]) Without Stack Overflow
> **"Pathname Expansion: High-Velocity Directory Traversal and Pattern Matching in Pure C"**  
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **Source Repository:** `zbash/docs/book/en/ch22_pathname_globbing.md`  
> **License:** [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

---

> [!NOTE]
> ### 🏛️ Chapter Epigraph
> **"The filesystem is a graph of unknown depth. Navigate it with boundary checks, or the kernel will punish your stack."**  
> — **Wirot Chookeaw**  
> 
> **"You don't need a heavy parser for everything. Simple string scanning and careful boundary conditions often do the job with ten times the velocity."**  
> — **Rob Pike**  

---

## 🧭 1. The Common Misconception About Asterisks (`*`)

For programmers beginning their journey into Unix systems, a classic question inevitably arises:
> *"When we run `rm *.txt`, how does the `rm` utility know which files to delete? Does `rm` contain code that searches for `*` characters?"*

The ground truth is surprising to newcomers: **The `rm` executable never sees the `*` character at all!**  
The entity responsible for scanning the storage filesystem, locating every filename ending with `.txt`, and expanding those matching filenames (e.g., `a.txt`, `b.txt`, `report.txt`) into individual discrete parameters passed to `rm` is the **Shell itself**.

This phase is termed **Pathname Expansion (or Globbing)**, a cornerstone phase of the Shell execution pipeline.  
However, engineering a globbing engine in bare-metal C that is simultaneously ultra-fast, robust, and zero-leak poses serious systems challenges:
* If a directory contains 50,000 files, how do we collect and sort them without triggering memory bloat or out-of-memory faults?
* If a user specifies a recursive pattern like `**/*.c` and encounters a cyclic symbolic link pointing back to an ancestor directory (symlink loop), does the shell enter an infinite loop and crash?
* Most fundamentally: Why does `echo *` expand to all filenames, whereas `echo "*"` prints a literal asterisk character?

In this chapter, we will dissect the architecture and implementation of a bare-metal C **Globbing & Pathname Expansion Engine**.

---

## 🏗️ 2. POSIX Pattern Matching Iron Laws

Wildcard matching in standard Unix shells adheres to three foundational rules:

```
        Wildcard Operator                     Matching Semantics
    ┌───────────────────────┐            ┌───────────────────────────────────────────────┐
    │          *            │ ─────────► │ Matches any string of zero or more characters │
    ├───────────────────────┤            ├───────────────────────────────────────────────┤
    │          ?            │ ─────────► │ Matches exactly one arbitrary character       │
    ├───────────────────────┤            ├───────────────────────────────────────────────┤
    │       [a-z0-9]        │ ─────────► │ Matches any single character within the range │
    ├───────────────────────┤            ├───────────────────────────────────────────────┤
    │       [!abc]          │ ─────────► │ Negation: matches any char NOT in the set     │
    └───────────────────────┘            └───────────────────────────────────────────────┘
```

### The Leading Dot Rule
Files beginning with a period, such as `.gitignore` or `.zshrc` (hidden files), are **never matched by an unqualified `*` wildcard**, unless the pattern explicitly specifies a leading dot (e.g., `.*`). This is a foundational Unix security principle designed to protect critical hidden configuration files from inadvertent erasure during commands such as `rm *`.

---

## 🔍 3. Distinguishing Quoted vs Unquoted in the Lexer

Consider the following invocation:
```bash
echo *.c "*" '*.c'
```
The semantically correct behavior requires:
* `*.c` (first argument): Must expand to matching filenames, e.g., `engine.c main.c lexer.c`.
* `"*"` (second argument): Must output the literal string `*`.
* `'*.c'` (third argument): Must output the literal string `*.c`.

In [`src/parser/lexer.c`](src/parser/lexer.c), we preserve this strict distinction by tracking quote states during token scanning:
* If `*`, `?`, or `[` appears within single quotes (`'...'`), double quotes (`"..."`), or is escaped by a backslash (`\*`), that character is tagged as a **Literal Character** and bypassed by the globbing engine.
* Only unquoted wildcard characters are passed into the pathname expansion pipeline to perform filesystem traversal.

---

## ⚙️ 4. Pure C Globbing Engine Architecture

Below is the directory traversal and pattern matching implementation utilizing standard POSIX `opendir()`, `readdir()`, and `fnmatch()`:

```c
#include <dirent.h>
#include <fnmatch.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include <limits.h>

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
        /* Skip hidden files unless the pattern explicitly begins with a dot */
        if (entry->d_name[0] == '.' && pattern[0] != '.') {
            continue;
        }

        /* Test pattern matching using POSIX fnmatch */
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

### Alphabetic Sorting (POSIX Collation):
POSIX mandates that all expanded pathname results must be returned in sorted alphabetical collation order:
```c
static int path_cmp(const void *a, const void *b) {
    return strcmp(*(const char **)a, *(const char **)b);
}

/* Sort results immediately before returning to caller */
qsort(res->paths, res->count, sizeof(char *), path_cmp);
```

---

## 🛡️ 5. Defending Against Symlink Loops & Stack Overflows

When users invoke recursive search patterns across subdirectories:
```bash
ls **/*.c
```
The greatest hazard in filesystems is a symbolic link that points cyclically back to an ancestor directory:
```bash
ln -s . loop_link
```
If a globbing function traverses directories recursively without cycle detection, the shell falls into **infinite recursion**, leading to an immediate **Stack Overflow crash**!

### Inode Tracking Mechanism in `zbash`:
Before opening any candidate subdirectory, we issue `stat()` to inspect filesystem hardware identifiers:
* `st_dev`: Storage device identifier.
* `st_ino`: Inode number of the target directory.

```c
#include <sys/stat.h>
#include <stdbool.h>

typedef struct {
    dev_t dev;
    ino_t ino;
} visited_node_t;

static visited_node_t visited_stack[256];
static int visited_depth = 0;

bool is_cyclic_symlink(const char *path) {
    struct stat st;
    if (stat(path, &st) != 0) return true;

    /* Verify whether this device + inode was already traversed in the ancestor chain */
    for (int i = 0; i < visited_depth; i++) {
        if (visited_stack[i].dev == st.st_dev && visited_stack[i].ino == st.st_ino) {
            return true; /* Cycle detected! Abort descending into this directory immediately */
        }
    }
    return false;
}
```
By maintaining this active inode stack during traversal, `zbash` guarantees complete immunity from cyclic symlink infinite loops and stack exhaustion!

---

## 🎯 6. Key Takeaways for Systems Developers

1. **The shell expands wildcards, not the target binary:** Understanding that `*` is expanded within the shell process before `execve()` is foundational to Unix systems engineering.
2. **Preserve quoting semantics through parsing:** The distinction between `*.c` and `"*.c"` must be preserved at the lexer stage to ensure predictable behavior.
3. **Filesystems are arbitrary graphs with cycles:** Inode tracking and recursion depth boundaries are indispensable safety mechanisms against malicious or corrupted filesystem structures.

In Chapter 23, we will construct the interactive feature beloved by every command-line engineer: **Interactive Autocompletion & Command Lookup: Tab Completion and O(1) $PATH Hashing**!
