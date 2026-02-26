---
title: MISRA Standard Library — Banned Functions, Restricted Headers, String Safety
impact: HIGH
impactDescription: eliminates heap fragmentation, prevents unbounded string operations, and bans unsafe libc usage
tags: misra, stdlib, malloc, free, banned-functions, string-safety, headers, safety
---

## MISRA Standard Library — Banned Functions, Restricted Headers, String Safety

Dynamic memory, unbounded string operations, and unrestricted standard library usage are the most dangerous categories in safety-critical embedded systems. These rules establish a strict allowlist of permitted standard library features.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 21.1  | Required | #define and #undef shall not be used on reserved identifiers or macros |
| 21.2  | Required | Reserved identifiers/macros shall not be declared |
| 21.3  | Required | Memory allocation/deallocation functions (malloc, calloc, realloc, free) shall not be used |
| 21.4  | Required | Standard header <setjmp.h> shall not be used |
| 21.5  | Required | Standard header <signal.h> shall not be used |
| 21.6  | Required | Standard I/O library <stdio.h> shall not be used in production code |
| 21.7  | Required | atof, atoi, atol, atoll shall not be used |
| 21.8  | Required | abort, exit, _Exit, quick_exit shall not be used |
| 21.9  | Required | <tgmath.h> shall not be used |
| 21.10 | Required | Standard header <time.h> shall not be used |
| 21.11 | Required | Standard header <tgmath.h> shall not be used |
| 21.12 | Advisory | <fenv.h> shall not be used |
| 21.13 | Mandatory | ctype.h functions shall be called with unsigned char values |
| 21.14 | Required | memcmp shall not be used to compare null-terminated strings |
| 21.15 | Required | memcpy/memmove/memcmp pointer arguments shall point to compatible types |
| 21.16 | Required | memcmp pointer arguments shall point to compatible types |
| 21.17 | Mandatory | String-handling functions shall not access beyond array bounds |
| 21.18 | Mandatory | size_t argument to string/memory function shall be valid |
| 21.19 | Mandatory | Pointers returned by localization functions shall not be modified |
| 21.20 | Mandatory | Returned pointer from standard library function shall not be used after next call |
| 21.21 | Required | System() of <stdlib.h> shall not be used |
| 22.1  | Required | All resources obtained dynamically shall be explicitly released |
| 22.2  | Mandatory | File object shall not be referenced after close |
| 22.3  | Required | Same file shall not be open for read and write simultaneously |
| 22.4  | Mandatory | Writing to read-only file shall not occur |
| 22.5  | Mandatory | FILE pointer shall not be dereferenced |
| 22.6  | Mandatory | FILE value shall not be copied |
| 22.7  | Required | EOF macro shall be compared with int return values |
| 22.8  | Required | errno shall be set to zero before calling errno-setting function |
| 22.9  | Required | errno shall be tested after calling errno-setting function |
| 22.10 | Required | errno shall only be tested when the function may set it |

MISRA C++:2023 overlap: Rules 21.6.1 (no dynamic memory in safety code), 21.10.1 (string safety), 0.2.1 (banned headers).

---

### Top Violations and Patterns

**Rule 21.3 — No Dynamic Memory Allocation (most critical)**

Incorrect (heap allocation in embedded):

```c
void ProcessCanMessages(uint16_t count) {
    CanMsg_t *msgs = (CanMsg_t *)malloc(count * sizeof(CanMsg_t));
    if (msgs == NULL) {
        return;  /* heap exhausted — system is now degraded */
    }
    /* ... process ... */
    free(msgs);  /* fragmentation risk */
}
```

Correct (static pool allocation):

```c
#define CAN_MSG_POOL_SIZE  64U
static CanMsg_t can_msg_pool[CAN_MSG_POOL_SIZE];
static uint16_t can_msg_pool_idx = 0U;

CanMsg_t *CanMsgPool_Alloc(void) {
    CanMsg_t *msg = NULL;
    if (can_msg_pool_idx < CAN_MSG_POOL_SIZE) {
        msg = &can_msg_pool[can_msg_pool_idx];
        can_msg_pool_idx++;
    }
    return msg;
}
```

C++ equivalent:

```cpp
#include <array>

template<typename T, std::size_t N>
class StaticPool {
    std::array<T, N> pool_{};
    std::size_t next_{0U};
public:
    T* allocate() {
        if (next_ >= N) { return nullptr; }
        return &pool_[next_++];
    }
    void reset() { next_ = 0U; }
};

StaticPool<CanMsg, 64U> can_pool;
```

---

**Rule 21.7 — No atoi/atol (no error detection)**

Incorrect:

```c
int32_t ParseParam(const char *str) {
    return atoi(str);  /* no overflow detection, UB on invalid input */
}
```

Correct (strtol with validation):

```c
Std_ReturnType ParseParam(const char *str, int32_t *result) {
    char *end = NULL;
    errno = 0;
    long val = strtol(str, &end, 10);

    if ((end == str) || (*end != '\0') || (errno != 0) ||
        (val < INT32_MIN) || (val > INT32_MAX)) {
        return E_NOT_OK;
    }

    *result = (int32_t)val;
    return E_OK;
}
```

---

**Rule 21.8 — No abort/exit**

Incorrect:

```c
void FatalError(uint16_t code) {
    printf("Fatal: %u\n", code);
    exit(1);  /* uncontrolled shutdown — no safe state entry */
}
```

Correct (controlled shutdown with safe state):

```c
void FatalError(uint16_t code) {
    Dem_ReportError(code);
    EnterSafeState();
    DisableAllInterrupts();
    for (;;) {
        Wdg_Trigger();  /* keep watchdog alive in safe state */
    }
}
```

---

**Rule 21.14 — No memcmp for String Comparison**

Incorrect:

```c
if (memcmp(version_str, "V2.0", 4U) == 0) { /* fragile: length mismatch risk */ }
```

Correct (strncmp with explicit limit):

```c
if (strncmp(version_str, "V2.0", sizeof("V2.0") - 1U) == 0) { /* bounded */ }
```

---

**Rule 21.17/21.18 — Bounded String Operations**

Incorrect (unbounded copy):

```c
char dest[16];
strcpy(dest, src);  /* buffer overflow if src > 15 chars */
```

Correct (bounded with explicit size):

```c
char dest[16];
(void)strncpy(dest, src, sizeof(dest) - 1U);
dest[sizeof(dest) - 1U] = '\0';
```

---

### Common Deviations

**Deviation: Rule 21.6 — stdio.h for Debug/Calibration Builds**

```
Deviation ID:    DEV-STDLIB-001
Rule:            MISRA C:2012 Rule 21.6 (Required)
Category:        Required → Permitted in non-production builds only
Scope:           Debug and calibration builds (compile switch)
Justification:   printf-based diagnostic output is needed during development
                 and calibration phases. All stdio usage is removed in
                 production builds via compile switch.
Conditions:      - Guarded by #ifdef DEBUG_BUILD or equivalent
                 - Production build system must define NDEBUG and NOT define DEBUG_BUILD
                 - CI/CD verifies that production binary has zero stdio references
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 21.3 — Pool Allocator with Fixed Lifetime**

```
Deviation ID:    DEV-STDLIB-002
Rule:            MISRA C:2012 Rule 21.3 (Required)
Category:        Required → NOT deviated (no malloc/free permitted)
Note:            This rule has no acceptable deviation in production embedded
                 code. All memory must be statically allocated. If a component
                 requires dynamic-like behavior, use a static pool allocator
                 with compile-time-known maximum capacity and WCET-bounded
                 allocation time.
```

Reference: MISRA C:2012 Rules 21.1–21.21, 22.1–22.10; MISRA C++:2023 Rules 21.6.1, 21.10.1, 0.2.1
