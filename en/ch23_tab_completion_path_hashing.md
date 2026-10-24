# Chapter 23: Interactive Autocompletion & Command Lookup: Tab Completion and O(1) $PATH Hashing
> **"Intelligent Command Lookup: Programmable Tab-Completion and O(1) PATH Hash Tables in C"**  
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **Source Repository:** `zbash/docs/book/en/ch23_tab_completion_path_hashing.md`  
> **License:** [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

---

> [!NOTE]
> ### 🏛️ Chapter Epigraph
> **"Speed is not just raw compute; it is remembering what you have already found."**  
> — **Wirot Chookeaw**  
> 
> **"An algorithm must be seen to be believed, and the best way to see what an algorithm is doing is to participate with it."**  
> — **Donald Knuth**  

---

## 🧭 1. The Pulse of User Experience: The Tab Key and $PATH

If you survey seasoned software engineers and systems operators with this question:
> *"Which key on your keyboard do you strike most frequently when working in a terminal?"*

The overwhelming answer is rarely an alphanumeric key; it is almost universally the **Tab key (`\t`)**!  
The ergonomic fluidity and appeal of the command-line interface hinges on typing two or three characters and striking Tab to let the shell automatically complete the intended command or path:
```bash
$ gi[Tab] ➔ $ git st[Tab] ➔ $ git status
```

And when the user presses Enter to execute `git status`:
* Does the shell traverse every discrete directory in `$PATH` each time?
* If the user's `$PATH` variable contains 15 directory entries (e.g., `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`...), must the shell issue up to 15 `access()` or `stat()` system calls to storage media for every single command execution?
* If it did, the shell would waste precious CPU cycles and storage I/O bandwidth on redundant directory scans!

In this chapter, we will dissect the architecture of a **Context-Aware Tab Autocompletion Engine** combined with an **$O(1)$ Command Hash Table ($PATH Hashing)** written in bare-metal C to deliver lightning-fast interactive responsiveness.

---

## 🏗️ 2. Context-Aware Autocompletion Architecture

When the user strikes the Tab key inside the line editor, the engine does not immediately query the filesystem. First, it performs **Contextual Analysis** on the cursor position and token boundaries:

```
                            [ Keystroke: Tab (\t) ]
                                        │
                  ┌─────────────────────┴─────────────────────┐
                  ▼                                           ▼
       [ First Word of Line (Command) ]          [ Subsequent Tokens (Arguments / Flags) ]
       Complete Builtins and $PATH binaries        Complete Files and Directories on Disk
                  │                                           │
                  └─────────────────────┬─────────────────────┘
                                        ▼
                          [ Prefixed with '$' Sigil? ]
                                        │
                                        ├──────► Yes: Complete Environment Variables ($HOME, $PATH...)
                                        └──────► No: Query Standard Filesystem / Builtin Registry
```

### Computing the Longest Common Prefix (LCP):
Suppose the user types `ca`, and the target directory contains both `cat.c` and `catalog.txt`:
* The engine does not guess or terminate prematurely; it expands the input buffer up to their longest shared character sequence: `cat`.
* If the user presses Tab twice in succession, the shell cleanly outputs the complete list of matching candidates below the active prompt and redraws the command line.

---

## 🔍 3. C Implementation of the Tab Completion Engine

Below is the completion candidate gathering architecture:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#include <dirent.h>

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

void zbash_complete_word(const char *prefix, bool is_command, zbash_completion_list_t *list) {
    size_t plen = strlen(prefix);

    /* 1. Command context: Search internal zbash builtin registry */
    if (is_command) {
        static const char *builtins[] = { "cd", "pwd", "echo", "exit", "export", "read", "set", "unset", NULL };
        for (int i = 0; builtins[i]; i++) {
            if (strncmp(builtins[i], prefix, plen) == 0) {
                add_completion(list, builtins[i]);
            }
        }
    }

    /* 2. File and directory traversal in current working directory */
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

Once candidates are assembled:
* If `list->count == 1`: Append the remaining suffix directly into the line editor buffer along with a trailing space.
* If `list->count > 1`: Compute the Longest Common Prefix (LCP) across all candidates, update the line buffer, and trigger the terminal bell character (`\a`) to alert the user.

---

## ⚡ 4. The Bottleneck of `$PATH` Lookups and the Genesis of $PATH Hashing

Consider a representative `$PATH` environment on a modern workstation:
```bash
PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:~/.cargo/bin:~/.local/bin
```
When a user issues the command `ls`:
1. Check `/opt/homebrew/bin/ls` (Not found) ➔ Kernel Stat System Call 1
2. Check `/usr/local/bin/ls` (Not found) ➔ Kernel Stat System Call 2
3. Check `/usr/bin/ls` (Not found) ➔ Kernel Stat System Call 3
4. Check `/bin/ls` (Found!) ➔ Kernel Stat System Call 4

If a shell script executes a loop 10,000 times invoking external utilities, the operating system would be forced to process **40,000 disk metadata queries** just to resolve the location of `ls`!

### The Systems Solution: $O(1)$ Hash Table
Standard Unix shells mitigate this overhead using **Command Hashing**:  
The first time an external command is resolved, the shell performs standard linear path traversal across `$PATH`. Once located, the binary's fully-qualified path is cached in an internal **Hash Table**.  
On subsequent invocations, disk traversal is completely bypassed; the shell retrieves the cached absolute path in $O(1)$ constant time and dispatches `execve()` immediately!

---

## 🗄️ 5. The $PATH Hash Table Data Structure in `zbash`

Here is the lightweight, zero-leak hash table design:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define ZBASH_HASH_BUCKETS 256

typedef struct path_hash_entry {
    char *cmd_name;                 /* e.g., "grep" */
    char *full_path;                /* e.g., "/usr/bin/grep" */
    int hits;                       /* Execution hit counter */
    struct path_hash_entry *next;   /* Chaining pointer for collisions */
} path_hash_entry_t;

static path_hash_entry_t *g_path_table[ZBASH_HASH_BUCKETS];

/* DJB2 Hash Algorithm: High dispersion, L1-cache friendly */
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
            entry->hits++; /* Increment hit count telemetry */
            return entry->full_path; /* Sub-microsecond O(1) resolution! */
        }
        entry = entry->next;
    }
    return NULL;
}
```

---

## 🛠️ 6. The Builtin `hash` Command and Cache Invalidation

When users execute the bare `hash` builtin, the shell prints telemetry regarding all cached external binaries and their execution hit counts:

```bash
$ hash
hits    command
   4    /usr/bin/grep
  12    /bin/ls
   1    /usr/bin/find
```

### Invalidating Cached Paths via `hash -r`:
If an administrator installs new software in `/usr/local/bin` or updates the `$PATH` environment variable, stale cached entries might point to older binaries. The `hash -r` builtin purges all hash buckets and frees allocated memory, forcing the shell to perform fresh filesystem searches on subsequent invocations.

---

## 🎯 7. Key Takeaways for Systems Developers

1. **UX is the soul of an interactive tool:** Tab autocompletion transforms a shell from a static prompt into an intuitive, responsive command cockpit.
2. **Never disregard the aggregate cost of kernel I/O:** Eliminating redundant `stat()` and `access()` calls via an in-memory hash table saves millions of CPU cycles in script execution.
3. **Caches must provide an invalidation mechanism:** Providing a user-facing flush mechanism (`hash -r`) ensures absolute correctness whenever filesystem state evolves.

In Chapter 24, we will reveal the engineering triumph that defines this project: **The Art of Binary Minimization: From C Source to an 86 KB Executable**!
