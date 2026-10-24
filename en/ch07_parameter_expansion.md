# Chapter 7: The Art of Parameter Expansion: Slicing, Defaults, and Lengths
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch07_parameter_expansion.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"Data dominates. If you've chosen the right data structures and organized things well, the algorithms will almost always be self-evident."**  
> — **Rob Pike** (Bell Labs)  
> 
> **"A crown jewel is not forged in comfort; it is crystallized through thousands of rigorous tests until it reflects pure truth."**  
> — **Wirot Chookeaw**

---

## 🧭 1. Parameter Expansion Hierarchy in POSIX

Among the most expressive capabilities of a POSIX shell is **Parameter Expansion**.  
Beyond basic identifier lookups like `$NAME`, a conforming shell supports:
* Fallback defaults on unset or null variables: `${VAR:-default}`
* In-place assignment and environment persistence: `${VAR:=default}`
* Scalar string lengths and array cardinality: `${#VAR}` and `${#arr[@]}`
* Computed substring slicing: `${VAR:offset:length}`

The central systems engineering challenge is: **How can this comprehensive grammar be implemented in clean C without bloating the binary beyond 86 KB or introducing heap memory leaks?**

The entire implementation is unified within `zbash_expand_vars` in [`src/parser/lexer.c`](src/parser/lexer.c#L183-L380).

---

## 🛠️ 2. Default Fallbacks: `${VAR:-default}` and `${VAR:=default}`

Consider the parsing logic in [`src/parser/lexer.c`](src/parser/lexer.c#L301-L326):

```c
/* File: src/parser/lexer.c (lines 301 - 326) */
                    } else if (op_char == ':') {
                        const char *env_v = getenv(vname_id);
                        if (rest[1] == '-') {
                            /* ${VAR:-default} */
                            const char *def_val = rest + 2;
                            if (env_v && *env_v) {
                                val = env_v;
                            } else {
                                // If the fallback default contains embedded parameters, expand recursively!
                                if (strchr(def_val, '$')) {
                                    dynamic_val = zbash_expand_vars(state, def_val);
                                    val = dynamic_val;
                                } else {
                                    val = def_val;
                                }
                            }
                        } else if (rest[1] == '=') {
                            /* ${VAR:=default} */
                            const char *def_val = rest + 2;
                            if (!env_v || *env_v == '\0') {
                                char *exp_def = strchr(def_val, '$') ? zbash_expand_vars(state, def_val) : strdup(def_val);
                                setenv(vname_id, exp_def, 1); // Commit side-effect to OS environment immediately!
                                dynamic_val = exp_def;
                                val = exp_def;
                            } else {
                                val = env_v;
                            }
                        }
```

### Critical Architectural Nuances:
1. **Recursive Parameter Resolution:**  
   When a user writes `${VAR:-$HOME/backup}`, the nested parameter `$HOME` must evaluate properly rather than remain an unexpanded literal string. `zbash` detects inner references with `strchr(def_val, '$')` and invokes `zbash_expand_vars` recursively.
2. **Persistent Side Effects of `${VAR:=}`:**  
   Unlike `:-`, the `:=` operator modifies the environment permanently via `setenv(vname_id, exp_def, 1)`. Subsequent processes in the pipeline immediately observe the newly assigned value.

---

## ✂️ 3. Computed Substring Slicing: `${VAR:offset:length}`

Substring slicing presents unique complexities because **both offset and length can be compound arithmetic expressions or variable references**, such as `${VAR:1+1:2*2}` or `${VAR:$off:$len}`.  
Consider [`src/parser/lexer.c`](src/parser/lexer.c#L341-L366):

```c
/* File: src/parser/lexer.c (lines 341 - 366) */
                        } else {
                            /* Substring slicing: ${VAR:offset} or ${VAR:offset:length} */
                            if (!env_v) env_v = "";
                            int vlen = (int)strlen(env_v);
                            char *sep2 = strchr(rest + 1, ':');
                            int offset = 0, slen = -1;
                            if (sep2) {
                                *sep2 = '\0';
                                // Compute offset and length via zbash's internal arithmetic engine!
                                offset = (int)zbash_eval_arithmetic(state, rest + 1);
                                slen = (int)zbash_eval_arithmetic(state, sep2 + 1);
                            } else {
                                offset = (int)zbash_eval_arithmetic(state, rest + 1);
                            }

                            // Strict boundary safety guards against buffer over-reads
                            if (offset < 0) offset = vlen + offset;
                            if (offset < 0) offset = 0;
                            if (offset > vlen) offset = vlen;
                            if (slen < 0 || offset + slen > vlen) slen = vlen - offset;
                            if (slen < 0) slen = 0;

                            dynamic_val = malloc(slen + 1);
                            if (dynamic_val) {
                                strncpy(dynamic_val, env_v + offset, slen);
                                dynamic_val[slen] = '\0';
                                val = dynamic_val;
                            } else {
                                val = "";
                            }
                        }
```

### Engineering Innovations:
* **Arithmetic Engine Integration:**  
  Rather than invoking basic `atoi()`, offsets and lengths pass directly into `zbash_eval_arithmetic(state, ...)`. This seamlessly evaluates arbitrary mathematical expressions (e.g., `${VAR:start+1:len*2}`).
* **Negative Offset Calculation:**  
  In POSIX shells, negative offsets (e.g., `${VAR: -2}`) represent counting backwards from the string's end. The normalization step `offset = vlen + offset` accurately aligns with this specification.

---

## 📏 4. Length and Element Cardinality: `${#VAR}` and `${#arr[@]}`

Consider the length calculation routines in [`src/parser/lexer.c`](src/parser/lexer.c#L198-L220):

```c
/* File: src/parser/lexer.c (lines 198 - 220) */
                /* Evaluate leading '#' length operator */
                else if (var_name[0] == '#') {
                    // Case 1: Total array cardinality ${#arr[@]} or ${#arr[*]}
                    if (strstr(var_name, "[@]") != NULL || strstr(var_name, "[*]") != NULL) {
                        char *b_start = strchr(var_name, '[');
                        *b_start = '\0';
                        zbash_array_t *arr = zbash_find_array(state, var_name + 1);
                        snprintf(num_buf, sizeof(num_buf), "%d", arr ? arr->count : 0);
                        val = num_buf;
                    } 
                    // Case 2: Individual array element string length ${#arr[idx]}
                    else if (strchr(var_name, '[') != NULL) {
                        char *b_start = strchr(var_name, '[');
                        char *b_end = strchr(b_start, ']');
                        if (b_end) {
                            *b_start = '\0'; *b_end = '\0';
                            int idx = atoi(b_start + 1);
                            const char *elem = zbash_get_array_elem(state, var_name + 1, idx);
                            snprintf(num_buf, sizeof(num_buf), "%zu", elem ? strlen(elem) : 0);
                            val = num_buf;
                        }
                    } 
                    // Case 3: Standard scalar variable string length ${#VAR}
                    else {
                        const char *env_v = getenv(var_name + 1);
                        snprintf(num_buf, sizeof(num_buf), "%zu", env_v ? strlen(env_v) : 0);
                        val = num_buf;
                    }
                }
```

* **Zero-Allocation Stack Buffer:** Number formatting uses a local `num_buf[64]` on the execution stack, eliminating heap allocation overhead during high-frequency length inspections.

---

## 🏁 Summary

1. **Parameter Expansion Completeness:** Proper handling of `:-`, `:=`, and `${#...}` forms the basis of reliable shell automation.
2. **Unified Arithmetic Evaluation:** Reusing the arithmetic evaluator for slicing boundaries enables expressive substring manipulation.
3. **Defensive Memory Safety:** Bounded string clamping prevents segmentation faults and out-of-bounds reads across all edge cases.

In **[Chapter 8: Embedded Math: Building the Arithmetic Evaluation Engine](ch08_arithmetic_engine.md)**, we examine how `$(( a + b * c ))` is parsed and calculated using a recursive descent engine with full operator precedence and 64-bit integer precision.
