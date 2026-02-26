---
title: MISRA Type System — Conversions, Casting, and Essential Type Model
impact: HIGH
impactDescription: prevents silent data corruption from implicit conversions and unsafe casts
tags: misra, type-system, casting, narrowing, signed-unsigned, essential-type, pointer-cast, safety
---

## MISRA Type System — Conversions, Casting, and Essential Type Model

The MISRA essential type model (Rules 10.x) and cast rules (Rules 11.x) form the backbone of type-safe embedded C/C++. Implicit conversions, narrowing, and unchecked casts are the single largest source of field defects in automotive software.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 10.1 | Required | Operands shall not be of an inappropriate essential type |
| 10.2 | Required | Expressions of essentially character type shall not be used in addition/subtraction |
| 10.3 | Required | Value of expression shall not be assigned to a narrower essential type |
| 10.4 | Required | Both operands of an operator shall have the same essential type category |
| 10.5 | Advisory | Value shall not be cast to an inappropriate essential type |
| 10.6 | Required | Value of composite expression shall not be assigned to wider essential type |
| 10.7 | Required | Composite expression with wider type shall not be implicit operand |
| 10.8 | Required | Value of composite expression shall not be cast to a different essential type |
| 11.1 | Required | Conversions shall not be performed between pointer to function and any other type |
| 11.2 | Required | Conversions shall not be performed between pointer to incomplete type and any other type |
| 11.3 | Required | Cast shall not convert between pointer to object and pointer to different object type |
| 11.4 | Advisory | Conversion between pointer to object and integer type shall not occur |
| 11.5 | Advisory | Conversion from pointer to void to pointer to object shall require explicit cast |
| 11.6 | Required | Cast shall not convert between pointer to void and arithmetic type |
| 11.7 | Required | Cast shall not convert between pointer to object and non-integer arithmetic type |
| 11.8 | Required | Cast shall not remove const/volatile qualification |
| 11.9 | Required | Macro NULL shall be the only permitted form of integer null pointer constant |

MISRA C++:2023 overlap: Rules 7.0.1–7.0.6 (standard conversions), 8.2.1–8.2.6 (explicit type conversions), 8.18.1 (reinterpret_cast restrictions).

---

### Top Violations and Patterns

**Rule 10.3 — Narrowing Assignment (most common violation)**

Incorrect (implicit narrowing):

```c
uint32_t raw_adc = ReadAdc();
uint16_t filtered = raw_adc * coeff;  /* narrowing: uint32_t → uint16_t */
int8_t temp = (raw_adc - 500) / 10;   /* signed/unsigned mix + narrowing */
```

Correct (explicit cast with range guard):

```c
uint32_t raw_adc = ReadAdc();
uint32_t calc = raw_adc * coeff;

if (calc <= UINT16_MAX) {
    uint16_t filtered = (uint16_t)calc;
} else {
    filtered = UINT16_MAX;
    Dem_ReportError(DEM_ADC_RANGE);
}
```

C++ equivalent:

```cpp
auto raw_adc = ReadAdc();
auto calc = raw_adc * coeff;
auto filtered = static_cast<uint16_t>(std::min(calc, static_cast<uint32_t>(UINT16_MAX)));
```

---

**Rule 10.4 — Mixed Essential Types in Operators**

Incorrect (signed/unsigned comparison):

```c
int16_t offset = -50;
uint16_t threshold = 100U;
if (offset < threshold) {  /* signed < unsigned: offset promotes to large unsigned */
    TriggerAlarm();
}
```

Correct (widen to common signed type):

```c
int16_t offset = -50;
uint16_t threshold = 100U;
if (offset < (int32_t)threshold) {
    TriggerAlarm();
}
```

---

**Rule 11.3 — Object Pointer Cast (hardware register access)**

Incorrect (arbitrary pointer reinterpretation):

```c
uint8_t buffer[64];
uint32_t *word_ptr = (uint32_t *)buffer;  /* alignment not guaranteed */
```

Correct (use memcpy for type punning):

```c
uint8_t buffer[64];
uint32_t word_val;
(void)memcpy(&word_val, &buffer[0], sizeof(word_val));
```

C++ equivalent:

```cpp
uint8_t buffer[64];
uint32_t word_val{};
std::memcpy(&word_val, &buffer[0], sizeof(word_val));
// C++20: std::bit_cast for trivially-copyable types
```

---

**Rule 11.4 — Pointer-to-Integer Conversion**

Incorrect (unguarded cast):

```c
void *ptr = GetBuffer();
uint32_t addr = (uint32_t)ptr;
```

Correct (use uintptr_t):

```c
#include <stdint.h>
void *ptr = GetBuffer();
uintptr_t addr = (uintptr_t)ptr;  /* guaranteed round-trip */
```

---

**Rule 11.8 — Removing const/volatile Qualification**

Incorrect:

```c
void WriteCalData(const CalData_t *cal) {
    CalData_t *mutable_cal = (CalData_t *)cal;  /* strips const */
    mutable_cal->checksum = Crc_Calculate(cal);
}
```

Correct (separate mutable copy):

```c
void WriteCalData(const CalData_t *cal, CalData_t *dest) {
    (void)memcpy(dest, cal, sizeof(CalData_t));
    dest->checksum = Crc_Calculate(dest);
}
```

---

### Common Deviations

**Deviation: Rule 11.3 — Hardware Register Access**

```
Deviation ID:    DEV-TYPE-001
Rule:            MISRA C:2012 Rule 11.3
Category:        Required → Permitted with conditions
Scope:           HAL/MCAL driver layer only
Justification:   Hardware peripheral registers are memory-mapped at fixed addresses
                 defined by the silicon vendor. Cast from integer constant to volatile
                 pointer is the only mechanism to access registers. All register
                 definitions use vendor-provided header files with verified alignment.
Conditions:      - Cast target must be volatile-qualified pointer
                 - Address must be a compile-time constant from vendor headers
                 - Cast must appear only in HAL/MCAL modules, never in application code
Approved by:     Safety Manager, [date]
```

Example of permitted usage:

```c
/* Vendor-supplied register definition — deviation DEV-TYPE-001 applies */
#define GPIOA_BASE  (0x40020000UL)
#define GPIOA       ((volatile GPIO_TypeDef *)GPIOA_BASE)  /* Rule 11.3 deviation */
```

**Deviation: Rule 11.4 — Pointer-to-Integer for DMA Addresses**

```
Deviation ID:    DEV-TYPE-002
Rule:            MISRA C:2012 Rule 11.4
Category:        Advisory → Permitted with conditions
Scope:           DMA controller configuration only
Justification:   DMA peripherals require physical addresses as integer values.
                 uintptr_t ensures no truncation.
Conditions:      - Must use uintptr_t, never plain uint32_t
                 - Only in DMA setup functions
```

Reference: MISRA C:2012 Rules 10.1–10.8, 11.1–11.9; MISRA C++:2023 Rules 7.0.x, 8.2.x
