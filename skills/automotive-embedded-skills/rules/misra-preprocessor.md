---
title: MISRA Preprocessor — Macro Safety, Include Guards, Conditional Compilation
impact: MEDIUM
impactDescription: prevents macro expansion bugs, header conflicts, and fragile conditional builds
tags: misra, preprocessor, macros, include-guard, pragma, conditional-compilation, safety
---

## MISRA Preprocessor — Macro Safety, Include Guards, Conditional Compilation

The C preprocessor operates outside the type system and can silently introduce bugs. These rules constrain macro usage to safe patterns, enforce include guard discipline, and limit conditional compilation complexity.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 20.1  | Advisory | #include directives should only be preceded by preprocessor directives or comments |
| 20.2  | Required | ', " or \ characters and /* or // in #include header name shall not be used |
| 20.3  | Required | #include shall be followed by either <filename> or "filename" |
| 20.4  | Required | Macro shall not be defined with the same name as a keyword |
| 20.5  | Advisory | #undef should not be used |
| 20.6  | Required | Tokens that look like preprocessing directives shall not occur in macro arguments |
| 20.7  | Required | Macro parameter used as expression shall be parenthesized |
| 20.8  | Required | Controlling expression of #if or #elif shall evaluate to 0 or 1 |
| 20.9  | Required | All identifiers used in controlling expression of #if/#elif shall be defined before use |
| 20.10 | Advisory | # and ## preprocessor operators should not be used |
| 20.11 | Required | Macro parameter immediately following # shall not be followed by ## |
| 20.12 | Required | Macro parameter used as argument to # or ## shall only be used that way |
| 20.13 | Required | Line whose first token is # shall be a valid preprocessor directive |
| 20.14 | Required | All #else, #elif and #endif shall reside in the same file as matching #if/#ifdef |
| Dir 4.4  | Advisory | Sections of code should not be "commented out" |
| Dir 4.9  | Advisory | Function-like macro should be replaced by inline function where possible |
| Dir 4.10 | Required | Precautions shall be taken to prevent header file contents from being included twice |
| Dir 4.11 | Required | Validity of function arguments shall be checked |
| Dir 4.12 | Required | Dynamic memory allocation shall not be used |
| Dir 4.13 | Advisory | Functions with variable number of arguments shall not be used |

MISRA C++:2023 overlap: Rules 19.0.1 (include guards / #pragma once), 19.1.1–19.1.3 (macro restrictions), 19.3.1 (include form).

---

### Top Violations and Patterns

**Rule 20.7 — Unparenthesized Macro Parameters (most common)**

Incorrect (parameter not protected):

```c
#define SQUARE(x)   x * x
#define MAX(a, b)   a > b ? a : b

uint16_t val = SQUARE(2 + 3);  /* expands to: 2 + 3 * 2 + 3 = 11, not 25 */
uint16_t m = MAX(x, y) + 1;   /* expands to: x > y ? x : y + 1 — wrong */
```

Correct (fully parenthesized):

```c
#define SQUARE(x)    ((x) * (x))
#define MAX(a, b)    (((a) > (b)) ? (a) : (b))
```

Better (inline function — Dir 4.9):

```c
static inline uint16_t Square(uint16_t x) {
    return x * x;
}
static inline uint16_t Max_u16(uint16_t a, uint16_t b) {
    return (a > b) ? a : b;
}
```

C++ equivalent:

```cpp
constexpr uint16_t Square(uint16_t x) { return x * x; }

template<typename T>
constexpr T Max(T a, T b) { return (a > b) ? a : b; }
```

---

**Dir 4.10 — Include Guard Discipline**

Incorrect (no include guard):

```c
/* sensor_cfg.h */
typedef struct { uint16_t offset; uint16_t gain; } SensorCfg_t;
/* including this header twice causes redefinition error */
```

Correct (traditional include guard):

```c
/* sensor_cfg.h */
#ifndef SENSOR_CFG_H
#define SENSOR_CFG_H

typedef struct { uint16_t offset; uint16_t gain; } SensorCfg_t;

#endif /* SENSOR_CFG_H */
```

Correct (pragma once — where compiler supports it):

```c
/* sensor_cfg.h */
#pragma once

typedef struct { uint16_t offset; uint16_t gain; } SensorCfg_t;
```

Convention: Use `MODULE_FILENAME_H` pattern. AUTOSAR headers use `MODULENAME_H` (e.g., `COM_H`, `DEM_H`).

---

**Dir 4.9 — Prefer Inline Functions Over Function-Like Macros**

Incorrect (macro with side-effect risk):

```c
#define ABS(x)  (((x) >= 0) ? (x) : -(x))
int16_t v = ABS(ReadSensor());  /* ReadSensor() called twice */
```

Correct (inline function):

```c
static inline int16_t Abs_s16(int16_t x) {
    return (x >= 0) ? x : (int16_t)(-x);
}
int16_t v = Abs_s16(ReadSensor());  /* called once */
```

---

**Rule 20.9 — Undefined Macro in #if Expression**

Incorrect (testing undefined macro — defaults to 0 silently):

```c
#if FEATURE_LIDAR_ENABLED   /* if not defined, silently evaluates to 0 */
    InitLidar();
#endif
```

Correct (use #ifdef or ensure definition):

```c
#ifdef FEATURE_LIDAR_ENABLED
    #if (FEATURE_LIDAR_ENABLED == 1)
        InitLidar();
    #endif
#endif
```

Or ensure all feature macros are defined in a project config header:

```c
/* project_cfg.h — all features explicitly defined */
#define FEATURE_LIDAR_ENABLED   0
#define FEATURE_RADAR_ENABLED   1
```

---

**Dir 4.4 — No Commented-Out Code**

Incorrect:

```c
void ProcessMsg(const Msg_t *msg) {
    // ValidateCrc(msg);  /* old validation — removed in v2.3 */
    DecodePayload(msg);
}
```

Correct (remove dead code, use version control):

```c
void ProcessMsg(const Msg_t *msg) {
    DecodePayload(msg);
}
```

---

### Common Deviations

**Deviation: Dir 4.9 — Macros for Compile-Time Constant Expressions**

```
Deviation ID:    DEV-PP-001
Rule:            MISRA C:2012 Dir 4.9 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           Project-wide
Justification:   Some macros produce compile-time constant expressions that
                 inline functions cannot (e.g., array sizes, bit masks,
                 struct initializers). These are used in contexts requiring
                 constant expressions (static array dimensions, case labels).
Conditions:      - Macro must produce a compile-time constant expression
                 - All parameters must be parenthesized per Rule 20.7
                 - If the macro can be replaced by constexpr (C++), it must be
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 20.10 — Stringification for Diagnostic Macros**

```
Deviation ID:    DEV-PP-002
Rule:            MISRA C:2012 Rule 20.10 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           Diagnostic assertion macros only
Justification:   The # (stringification) operator is required to produce
                 human-readable assertion failure messages that include
                 the failing expression text. This is critical for
                 debugging during development.
Conditions:      - Only in assertion/diagnostic macros (e.g., DEV_ASSERT)
                 - Assertion macros must be compile-time removable for production builds
                 - No use of ## (token pasting) in combination with #
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Rules 20.1–20.14, Directives 4.4, 4.9–4.13; MISRA C++:2023 Rules 19.x
