---
title: Avoid Implicit Type Conversions
impact: HIGH
impactDescription: prevents data loss and unexpected behavior
tags: misra, type-conversion, narrowing, signed-unsigned, casting, safety
---

## Avoid Implicit Type Conversions

Implicit conversions between signed/unsigned, narrowing conversions, and integer promotion can cause subtle bugs. Always use explicit casts with range checking.

**Incorrect (implicit narrowing):**

```c
uint16_t adc_value = ReadAdc();  /* Returns uint32_t */
int8_t temperature = (adc_value - 500) / 10;  /* Signed/unsigned mix, narrowing */
```

**Correct (explicit casting with range check):**

```c
uint32_t raw_adc = ReadAdc();
int32_t temp_calc = ((int32_t)raw_adc - 500) / 10;

if ((temp_calc >= INT8_MIN) && (temp_calc <= INT8_MAX))
{
    temperature = (int8_t)temp_calc;
}
else
{
    temperature = TEMP_DEFAULT_VALUE;
    ReportError(ERR_TEMP_RANGE);
}
```

Reference: MISRA C:2012 Rules 10.1–10.8 (Essential type model)
