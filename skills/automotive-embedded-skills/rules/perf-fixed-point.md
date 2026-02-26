---
title: Fixed-Point Arithmetic
impact: HIGH
impactDescription: Avoids FPU dependency on MCUs without hardware floating-point
tags: perf, fixed-point, arithmetic, fpu, optimization
---

## Fixed-Point Arithmetic

Use fixed-point instead of floating-point on MCUs without hardware FPU. Software floating-point emulation is 10-100x slower and non-deterministic in timing.

**Incorrect (floating-point on FPU-less MCU):**

```c
float FilterSensor(float raw, float coeff)
{
    return raw * coeff + (1.0f - coeff) * g_prevValue;
}
```

**Correct (Q16.16 fixed-point):**

```c
typedef int32_t fixed16_16_t;  /* Q16.16 format */

#define FIXED_ONE     ((fixed16_16_t)0x00010000)
#define FLOAT_TO_FIX(f)  ((fixed16_16_t)((f) * 65536.0f))
#define FIX_TO_FLOAT(x)  ((float)(x) / 65536.0f)

static inline fixed16_16_t FixMul(fixed16_16_t a, fixed16_16_t b)
{
    return (fixed16_16_t)(((int64_t)a * b) >> 16);
}
```

Choose Q-format based on required range and precision. Q16.16 provides ±32767 range with ~0.000015 resolution. For signals with larger range (e.g., RPM), use Q8.24 or Q24.8.
