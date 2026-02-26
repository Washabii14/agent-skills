---
title: MISRA Functions — Prototypes, Parameters, Returns, Recursion Ban
impact: HIGH
impactDescription: ensures correct function interfaces, prevents stack overflow from recursion, enforces return value handling
tags: misra, functions, prototype, return-value, recursion, parameter, variadic, safety
---

## MISRA Functions — Prototypes, Parameters, Returns, Recursion Ban

Function interface errors (missing prototypes, ignored return values, unintended recursion) are consistently in the top 10 automotive field defects. These rules enforce strict function contracts.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 17.1 | Required | stdarg.h shall not be used (no variadic functions) |
| 17.2 | Required | Functions shall not call themselves, directly or indirectly (no recursion) |
| 17.3 | Mandatory | Function shall not be declared implicitly |
| 17.4 | Mandatory | All exit paths from function with non-void return shall have explicit return |
| 17.5 | Advisory | Function argument corresponding to array parameter should have appropriate size |
| 17.6 | Mandatory | Array parameter shall not be written to beyond its declared size |
| 17.7 | Required | Value returned by non-void function shall be used |
| 17.8 | Advisory | Function parameters should not be modified |
| 8.2  | Required | Function types shall be in prototype form with named parameters |
| 8.3  | Required | Declaration and definition of function shall use same names and types |
| 8.4  | Required | Compatible declaration visible when function/object defined |

MISRA C++:2023 overlap: Rules 6.8.1 (function declaration), 7.6.1 (return type), 0.3.1–0.3.2 (unused return values), 21.10.1 (no recursion for safety-critical).

---

### Top Violations and Patterns

**Rule 17.7 — Ignoring Return Value (most common)**

Incorrect (return value silently ignored):

```c
Eep_Write(addr, data, len);  /* returns Std_ReturnType — error lost */
memcpy(dest, src, len);       /* returns dest pointer — typically fine to ignore */
```

Correct (check or explicitly discard):

```c
Std_ReturnType result = Eep_Write(addr, data, len);
if (result != E_OK) {
    Dem_ReportError(DEM_EEP_WRITE_FAIL);
}

(void)memcpy(dest, src, len);  /* explicit void cast = intentional discard */
```

C++ equivalent:

```cpp
[[nodiscard]] Std_ReturnType Eep_Write(uint32_t addr, const uint8_t* data, uint16_t len);

auto result = Eep_Write(addr, data, len);  // compiler warns if ignored
if (result != Std_ReturnType::kOk) {
    Dem_ReportError(DemEventId::kEepWriteFail);
}
```

---

**Rule 17.2 — No Recursion**

Incorrect (recursive tree walk):

```c
uint32_t SumTree(const Node_t *node) {
    if (node == NULL) { return 0U; }
    return node->value + SumTree(node->left) + SumTree(node->right);  /* recursion */
}
```

Correct (iterative with explicit stack):

```c
#define MAX_DEPTH 32U

uint32_t SumTree(const Node_t *root) {
    uint32_t sum = 0U;
    const Node_t *stack[MAX_DEPTH];
    uint8_t sp = 0U;

    if (root != NULL) {
        stack[sp] = root;
        sp++;
    }

    while (sp > 0U) {
        sp--;
        const Node_t *node = stack[sp];
        sum += node->value;

        if ((node->right != NULL) && (sp < MAX_DEPTH)) {
            stack[sp] = node->right;
            sp++;
        }
        if ((node->left != NULL) && (sp < MAX_DEPTH)) {
            stack[sp] = node->left;
            sp++;
        }
    }
    return sum;
}
```

---

**Rule 17.1 — No Variadic Functions**

Incorrect (variadic logging):

```c
#include <stdarg.h>
void Log(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    vprintf(fmt, args);
    va_end(args);
}
```

Correct (typed parameters or macro with fixed args):

```c
void Log_U32(const char *msg, uint32_t value) {
    /* type-safe logging with known parameter types */
}

void Log_Status(const char *module, Std_ReturnType status) {
    /* domain-specific log with explicit types */
}
```

C++ equivalent (variadic template — type-safe):

```cpp
template<typename... Args>
void Log(std::string_view fmt, Args&&... args) {
    // Type-safe at compile time — no runtime variadic promotion issues
    // MISRA C++:2023 permits variadic templates (not C-style varargs)
}
```

---

**Rule 17.4 — All Paths Must Return**

Incorrect (missing return on one branch):

```c
uint8_t GetPriority(TaskId_t id) {
    if (id < MAX_TASKS) {
        return task_table[id].priority;
    }
    /* no return statement — undefined behavior */
}
```

Correct (explicit return on all paths):

```c
uint8_t GetPriority(TaskId_t id) {
    uint8_t prio = PRIO_LOWEST;
    if (id < MAX_TASKS) {
        prio = task_table[id].priority;
    } else {
        Dem_ReportError(DEM_INVALID_TASK_ID);
    }
    return prio;
}
```

---

**Rule 17.8 — Do Not Modify Parameters**

Incorrect (parameter used as loop variable):

```c
void SendBytes(const uint8_t *data, uint16_t len) {
    while (len > 0U) {
        Uart_TxByte(*data);
        data++;
        len--;  /* modifying parameter directly */
    }
}
```

Correct (local copy of parameter):

```c
void SendBytes(const uint8_t *data, uint16_t len) {
    uint16_t remaining = len;
    const uint8_t *ptr = data;

    while (remaining > 0U) {
        Uart_TxByte(*ptr);
        ptr++;
        remaining--;
    }
}
```

---

### Common Deviations

**Deviation: Rule 17.2 — Recursion in Code-Generation Tools**

```
Deviation ID:    DEV-FUNC-001
Rule:            MISRA C:2012 Rule 17.2 (Required)
Category:        Required → NOT deviated in production code
Note:            Recursion is NEVER permitted in production embedded code.
                 Host-side tooling (code generators, test harnesses running
                 on PC) may use recursion but that code is not deployed
                 to the target and is outside MISRA scope.
```

**Deviation: Rule 17.7 — Void-Cast for Known-Safe Returns**

```
Deviation ID:    DEV-FUNC-002
Rule:            MISRA C:2012 Rule 17.7 (Required)
Category:        Required → Mitigated
Scope:           Standard library functions with always-succeeding semantics
Justification:   Functions like memcpy, memset, and strcpy return their
                 destination pointer. Since the destination is already known
                 and failure is undefined (not signaled by return), the return
                 value carries no useful information.
Conditions:      - Must use explicit (void) cast to indicate intentional discard
                 - Only for standard library functions whose return value is
                   the destination pointer
                 - Functions returning error codes must ALWAYS be checked
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Rules 17.1–17.8, 8.2–8.4; MISRA C++:2023 Rules 6.8.1, 7.6.1, 0.3.x, 21.10.1
