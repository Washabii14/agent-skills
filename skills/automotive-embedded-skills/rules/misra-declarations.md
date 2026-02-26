---
title: MISRA Declarations — Scope, Linkage, Storage Class, Unused Variables
impact: MEDIUM
impactDescription: ensures correct visibility, prevents naming collisions, eliminates dead declarations
tags: misra, declarations, scope, linkage, extern, static, storage-class, unused, safety
---

## MISRA Declarations — Scope, Linkage, Storage Class, Unused Variables

Correct declarations enforce encapsulation, prevent unintended symbol exposure, and eliminate dead code. These rules ensure every declaration has exactly the scope and linkage it needs.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 8.1  | Required | Types shall be explicitly stated (no implicit int) |
| 8.2  | Required | Function types shall be in prototype form with named parameters |
| 8.3  | Required | Declaration and definition of function shall use the same names and types |
| 8.4  | Required | Compatible declaration shall be visible when function/object is defined |
| 8.5  | Required | External object/function shall be declared once in one header |
| 8.6  | Required | Identifier with external linkage shall have exactly one definition |
| 8.7  | Advisory | Functions and objects not accessed outside the TU should be file-scoped (static) |
| 8.8  | Required | static storage class specifier shall be used for internal linkage |
| 8.9  | Advisory | Object with block scope that is not modified should be declared at the narrowest scope |
| 8.10 | Required | Inline function shall be declared with static storage class |
| 8.11 | Advisory | extern array declarations should have an explicit size |
| 8.12 | Required | Within an enum, the value of an implicitly specified constant shall be unique |
| 8.13 | Advisory | Pointer parameter should be declared const if the pointed-to object is not modified |
| 8.14 | Required | The restrict type qualifier shall not be used |

MISRA C++:2023 overlap: Rules 6.0.1 (trivial declarations), 6.0.4 (identifier scope), 6.2.1–6.2.4 (linkage), 6.4.1–6.4.2 (name hiding).

---

### Top Violations and Patterns

**Rule 8.7 — File-Scope Static for Internal Symbols (most common violation)**

Incorrect (unnecessary external linkage):

```c
/* motor_ctrl.c */
uint16_t motor_speed_rpm;            /* globally visible — any TU can modify */
void MotorCtrl_UpdatePwm(void) { }   /* globally visible but only used here */
```

Correct (static for file-private symbols):

```c
/* motor_ctrl.c */
static uint16_t motor_speed_rpm;            /* file-scoped only */
static void MotorCtrl_UpdatePwm(void) { }   /* file-scoped only */
```

C++ equivalent (anonymous namespace):

```cpp
// motor_ctrl.cpp
namespace {
    uint16_t motor_speed_rpm{0U};
    void UpdatePwm() { /* ... */ }
}
```

---

**Rule 8.2 — Named Parameters in Prototypes**

Incorrect (unnamed parameters):

```c
Std_ReturnType Spi_Transfer(uint8_t, const uint8_t *, uint16_t);
```

Correct (named parameters for self-documentation):

```c
Std_ReturnType Spi_Transfer(uint8_t channel, const uint8_t *txData, uint16_t length);
```

---

**Rule 8.4 — Visible Declaration Before Definition**

Incorrect (function defined without prior declaration visible):

```c
/* sensor.c — no #include "sensor.h" */
uint16_t Sensor_ReadRaw(uint8_t channel) {  /* no prototype visible */
    return AdcHw_Read(channel);
}
```

Correct (include header declaring the function):

```c
/* sensor.h */
uint16_t Sensor_ReadRaw(uint8_t channel);

/* sensor.c */
#include "sensor.h"  /* declaration now visible */

uint16_t Sensor_ReadRaw(uint8_t channel) {
    return AdcHw_Read(channel);
}
```

---

**Rule 8.13 — Const-Qualify Unmodified Pointer Targets**

Incorrect (missing const on read-only data):

```c
uint16_t CalcChecksum(uint8_t *data, uint16_t len) {
    uint16_t sum = 0U;
    for (uint16_t i = 0U; i < len; i++) {
        sum += (uint16_t)data[i];
    }
    return sum;
}
```

Correct (const-qualified pointer):

```c
uint16_t CalcChecksum(const uint8_t *data, uint16_t len) {
    uint16_t sum = 0U;
    for (uint16_t i = 0U; i < len; i++) {
        sum += (uint16_t)data[i];
    }
    return sum;
}
```

---

**Rule 8.12 — Unique Enum Constants**

Incorrect (duplicate implicit values):

```c
typedef enum {
    ERR_NONE = 0,
    ERR_TIMEOUT,     /* implicitly 1 */
    ERR_CRC,         /* implicitly 2 */
    ERR_FIRST = 1    /* 1 again — collides with ERR_TIMEOUT */
} ErrorCode_t;
```

Correct (all values explicit and unique):

```c
typedef enum {
    ERR_NONE    = 0U,
    ERR_TIMEOUT = 1U,
    ERR_CRC     = 2U,
    ERR_FIRST   = ERR_TIMEOUT  /* alias uses named constant, intent is clear */
} ErrorCode_t;
```

---

### Common Deviations

**Deviation: Rule 8.7 — Extern Declarations in Shared Headers**

```
Deviation ID:    DEV-DECL-001
Rule:            MISRA C:2012 Rule 8.7 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           AUTOSAR BSW inter-module interfaces
Justification:   AUTOSAR Basic Software specifies that certain global objects
                 (e.g., task control blocks, configuration structures) shall
                 have external linkage to be accessible across BSW modules
                 as defined by the AUTOSAR SWS. Making them static would
                 violate the AUTOSAR architecture specification.
Conditions:      - Only for symbols explicitly required by AUTOSAR SWS
                 - Symbol must be declared in exactly one header (Rule 8.5)
                 - Header must be the module's public interface header
                 - Each extern declaration must reference a single definition (Rule 8.6)
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 8.14 — Restrict Qualifier in Performance-Critical DSP**

```
Deviation ID:    DEV-DECL-002
Rule:            MISRA C:2012 Rule 8.14 (Required)
Category:        Required → Permitted with conditions
Scope:           Signal processing / DSP library functions only
Justification:   The restrict qualifier enables critical vectorization
                 optimizations in signal filtering loops. Without it,
                 the compiler must assume aliasing, resulting in 2-3x
                 slower execution that violates timing requirements.
Conditions:      - Only in clearly documented DSP/math library functions
                 - No overlapping pointer arguments (verified by unit test)
                 - Function documentation must specify the aliasing constraint
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Rules 8.1–8.14; MISRA C++:2023 Rules 6.0.x, 6.2.x, 6.4.x
