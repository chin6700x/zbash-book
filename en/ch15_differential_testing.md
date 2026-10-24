# Chapter 15: "Fast with Wrong Output Is Worthless": Differential Testing
> **Author:** Wirot Chookeaw (`chin6700x`)  
> **License:** Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0) © 2026  
> *Published for free personal and educational reading. Commercial reproduction, redistribution, or modification is strictly prohibited.*  
> **Source Reference:** `zbash/docs/book/en/ch15_differential_testing.md`

---

> [!NOTE]
> ### 🏛️ Guiding Maxims
> **"High execution speed with incorrect output is worthless; high speed coupled with bit-exact parity represents genuine progress."**  
> — **Wirot Chookeaw**  
> 
> **"Beware of bugs in the above code; I have only proved it correct, not tried it."**  
> — **Donald Knuth**

---

## 🧭 1. The Illusion of Speed

In modern software development, new tools often announce themselves with striking benchmark bar charts:  
> *"10x faster startup! 5x lower memory usage!"*

Yet when introduced into production pipelines, unverified replacements frequently introduce subtle failures:
* Build workflows in Xcode or Make fail midway because an exit code deviated from established convention.
* Configuration templates corrupt because parameter expansion stripped trailing whitespace unexpectedly.
* Environment variables fail to inherit properly, breaking runtime dynamic linkers.

This reality anchors the engineering philosophy of `zbash`:
> **"High execution speed with incorrect output is worthless; high speed coupled with bit-exact parity represents genuine progress."**

Raw throughput holds zero value if output deviates by even a single byte or breaks behaviors relied upon across forty years of computing history.

In this chapter, we explore **Differential Parity Testing**, the verification methodology that ensures `zbash` serves as a faithful drop-in replacement for GNU Bash.

---

## 🏗️ 2. Why Conventional Unit Testing Falls Short for Shells

In standard application development, unit tests focus on isolated assertions:

```c
assert(add(2, 3) == 5);
assert(strlen("hello") == 5);
```

For a Unix shell, this strategy is insufficient:
1. **Combinatorial State Space:** Shell syntax allows quotes, wildcards, pipelines, redirections, subshells, and variable substitutions to compose in millions of permutations. Manually authoring static expected assertions cannot cover this surface area.
2. **Undocumented Historical Behaviors:** Decades of real-world scripts rely on specific behaviors formed through historical bug fixes across 35 years of GNU Bash maintenance, many of which extend beyond formal POSIX specifications.

The effective path forward is straightforward: **Do not guess expected outputs—measure directly against the reference implementation!**

---

## 🔍 3. Differential Testing Architecture

Differential testing executes identical command inputs simultaneously against the reference baseline (**macOS `/bin/bash`**) and the system under test (**`zbash`**), validating outputs side by side:

```
                            [ Script Test Case ]
                                     │
                 ┌───────────────────┴───────────────────┐
                 ▼                                       ▼
        [ Golden Baseline ]                     [ Under Test (SUT) ]
         macOS /bin/bash                              zbash
                 │                                       │
         ┌───────┴───────┐                       ┌───────┴───────┐
         ▼               ▼                       ▼               ▼
     [ stdout ]      [ exit code ]           [ stdout ]      [ exit code ]
         │               │                       │               │
         └───────────────┼───────────────────────┘               │
                         ▼                                       ▼
                 [ diff / cmp -s ]                       [ rc1 == rc2 ]
                         │                                       │
                         └───────────────────┬───────────────────┘
                                             ▼
                                  [ PASS / FAIL VERDICT ]
```

### Verification Criteria:
A test case earns a **PASS** verdict if and only if:
1. **Standard Output (STDOUT):** Matches byte-for-byte with the baseline.
2. **Termination Exit Status (`$?`):** Matches identically (e.g., 0 vs 0, 127 vs 127, 2 vs 2).

Any discrepancy—even a single missing newline or mismatched status code—triggers an immediate **FAIL**.

---

## ⚙️ 4. The Test Harness in `test/1000-test/common.sh`

Consider the test harness driving the suite in [`test/1000-test/common.sh`](test/1000-test/common.sh):

```bash
# File: test/1000-test/common.sh
run_test() {
    local test_num="$1"
    local desc="$2"
    local cmd="$3"

    # 1. Execute against macOS /bin/bash (Golden Baseline)
    BASH_OUT=$(/bin/bash -c "$cmd" 2>&1)
    BASH_RC=$?

    # 2. Execute against zbash (Bare-Metal Engine)
    ZBASH_OUT=$("$ZBASH_BIN" -c "$cmd" 2>&1)
    ZBASH_RC=$?

    # 3. Compare Results
    if [ "$BASH_RC" -eq "$ZBASH_RC" ] && [ "$BASH_OUT" = "$ZBASH_OUT" ]; then
        PASS_COUNT=$((PASS_COUNT + 1))
        echo -e "  [PASS] Test #$test_num: $desc"
    else
        FAIL_COUNT=$((FAIL_COUNT + 1))
        echo -e "  [FAIL] Test #$test_num: $desc"
        echo -e "    COMMAND: $cmd"
        echo -e "    EXPECTED RC: $BASH_RC, GOT: $ZBASH_RC"
        echo -e "    EXPECTED OUT: $BASH_OUT"
        echo -e "    ACTUAL OUT:   $ZBASH_OUT"
    fi
}
```

### Key Advantages:
1. **Automatic Edge Case Verification:** Rather than manually computing the output of `${foo:-${bar:=baz}}`, the harness queries Bash directly, verifying that `zbash` delivers identical behavior.
2. **Instant Regression Detection:** Whenever C routines are refactored or ARM64 assembly kernels are optimized, running the differential suite takes mere seconds, catching regressions immediately.

---

## 🧪 5. From 100 to 1,000 Tests: Elevating Quality Standards

Initial prototypes were evaluated against a 100-case suite. However, true production reliability demands far deeper validation.  
The suite was expanded into **10 modular test suites spanning 1,000 distinct cases**:
* Variable scopes and environment variables
* Nested recursive arithmetic expressions
* 4-level deep loops and conditionals
* Complex pipelines and compound redirections
* Subshells and in-process command grouping
* Indexed arrays and sparse gap handling
* UTF-8 Unicode multibyte strings and stress boundaries

---

## 🏁 Summary

1. **Let Proven Standards Guide Validation:** When building a drop-in replacement, evaluate against the incumbent reference.
2. **Differential Testing Delivers Depth:** It eliminates the labor of authoring static assertions and provides far broader coverage than isolated unit tests.
3. **Correctness Is Non-Negotiable:** Tolerating small discrepancies during development invites systemic failure in production.

In **[Chapter 16: Anatomy of the 1,000-Test Suite: 10 Modules vs macOS Bash](ch16_anatomy_of_1000_tests.md)**, we break down each of the 10 test modules, examining their specific test vectors and edge cases.
