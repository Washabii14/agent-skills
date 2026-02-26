---
title: MISRA Expressions — Side Effects, Precedence, Booleans, Sizeof
impact: HIGH
impactDescription: eliminates undefined evaluation order, implicit precedence bugs, and boolean misuse
tags: misra, expressions, side-effects, precedence, boolean, sizeof, evaluation-order, safety
---

## MISRA Expressions — Side Effects, Precedence, Booleans, Sizeof

Expressions with hidden side effects, ambiguous precedence, or misused booleans are a persistent source of intermittent bugs that are extremely difficult to reproduce in the field. These rules enforce unambiguous, deterministic expression evaluation.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 12.1 | Advisory | Precedence of operators should be made explicit with parentheses |
| 12.2 | Required | Right-hand operand of shift shall not exceed bit width |
| 12.3 | Advisory | Comma operator should not be used |
| 12.4 | Advisory | Evaluation of constant expressions should not lead to unsigned wrap-around |
| 12.5 | Mandatory | sizeof shall not have operands with side effects |
| 13.1 | Required | Initializer lists shall not contain persistent side effects |
| 13.2 | Required | Value of expression and its persistent side effects shall be deterministic |
| 13.3 | Advisory | Full expression containing increment/decrement should have no other side effects |
| 13.4 | Advisory | Result of an assignment shall not be used |
| 13.5 | Required | Right-hand operand of logical && or \|\| shall not have persistent side effects |
| 13.6 | Mandatory | sizeof operand shall not contain expression with potential side effects |
| 14.1 | Required | Loop counter shall have essentially floating type only for (not for) |
| 14.2 | Required | for loop shall be well-formed (single counter, standard pattern) |
| 14.3 | Required | Controlling expression shall not be invariant |
| 14.4 | Required | Controlling expression of if/while shall have essentially boolean type |

MISRA C++:2023 overlap: Rules 5.0.1 (evaluation order), 5.1.1–5.1.2 (sizeof), 8.14.1 (boolean conversion), 9.2.1 (for loop form).

---

### Top Violations and Patterns

**Rule 12.1 — Ambiguous Operator Precedence (most common)**

Incorrect (relying on precedence knowledge):

```c
uint8_t flags = status & MASK_READY | MASK_ENABLED;  /* & before | — intended? */
uint16_t result = a + b << 2;                         /* + before << — surprise */
```

Correct (explicit parentheses):

```c
uint8_t flags = (status & MASK_READY) | MASK_ENABLED;
uint16_t result = (uint16_t)((uint16_t)a + (uint16_t)((uint16_t)b << 2U));
```

---

**Rule 13.2 — Non-Deterministic Evaluation Order**

Incorrect (two side effects with undefined sequencing):

```c
buffer[idx++] = GetNextByte(idx);  /* idx read and modified — order undefined */
```

Correct (separate statements):

```c
uint8_t byte = GetNextByte(idx);
buffer[idx] = byte;
idx++;
```

---

**Rule 13.5 — Side Effects in Short-Circuit Operands**

Incorrect (function call in right-hand side may not execute):

```c
if (IsValid(frame) && (ProcessFrame(frame) == E_OK)) {
    /* ProcessFrame has side effects — may be skipped by short-circuit */
    ConfirmProcessed();
}
```

Correct (separate the side effect):

```c
if (IsValid(frame)) {
    Std_ReturnType proc_result = ProcessFrame(frame);
    if (proc_result == E_OK) {
        ConfirmProcessed();
    }
}
```

---

**Rule 14.4 — Boolean Type in Controlling Expression**

Incorrect (integer used as boolean):

```c
uint8_t error_count = GetErrorCount();
if (error_count) {       /* integer used as boolean */
    HandleErrors();
}

while (remaining--) {    /* decrement + implicit boolean — two violations */
    ProcessNext();
}
```

Correct (explicit boolean comparison):

```c
uint8_t error_count = GetErrorCount();
if (error_count > 0U) {
    HandleErrors();
}

while (remaining > 0U) {
    ProcessNext();
    remaining--;
}
```

---

**Rule 12.2 — Shift Exceeds Bit Width**

Incorrect (undefined behavior on over-shift):

```c
uint8_t val = 1U;
uint32_t mask = val << 24U;  /* shift of 8-bit value by 24 — UB */
```

Correct (widen before shifting):

```c
uint32_t mask = (uint32_t)1U << 24U;
```

---

**Rule 13.4 — Assignment Used as Value**

Incorrect (assignment inside condition):

```c
if ((result = PerformAction()) != E_OK) {
    HandleError(result);
}
```

Correct (separate assignment from test):

```c
result = PerformAction();
if (result != E_OK) {
    HandleError(result);
}
```

---

### Common Deviations

**Deviation: Rule 12.1 — Established Bitfield Idioms**

```
Deviation ID:    DEV-EXPR-001
Rule:            MISRA C:2012 Rule 12.1 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           Bitfield manipulation macros in HAL headers
Justification:   Established hardware register manipulation idioms like
                 (REG & MASK) and (REG | FLAG) have unambiguous precedence
                 universally understood by embedded engineers. Excessive
                 parenthesization of simple bit ops reduces readability.
Conditions:      - Only for single & or | with a mask constant
                 - Mixed operators (& with | with shift) must still use parentheses
                 - Only in HAL/MCAL register access macros
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 14.3 — Compile-Time Conditional Guards**

```
Deviation ID:    DEV-EXPR-002
Rule:            MISRA C:2012 Rule 14.3 (Required)
Category:        Required → Permitted with conditions
Scope:           Configuration-dependent code
Justification:   Some controlling expressions are invariant for a specific
                 build configuration but vary across variants (e.g.,
                 if (VARIANT_HAS_LIDAR)). The invariant expression is
                 intentional for dead-code elimination by the compiler
                 while keeping a single source file for all variants.
Conditions:      - Invariant expression must reference a documented configuration macro
                 - Configuration macro must be defined in a project-level config header
                 - Dead branch coverage is excluded from coverage targets for that variant
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Rules 12.1–12.5, 13.1–13.6, 14.1–14.4; MISRA C++:2023 Rules 5.0.x, 8.14.x, 9.2.x
