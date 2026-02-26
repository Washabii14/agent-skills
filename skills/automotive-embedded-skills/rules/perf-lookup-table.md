---
title: Lookup Tables Over Runtime Computation
impact: HIGH
impactDescription: Trades Flash for CPU cycles via pre-computed values
tags: perf, lookup-table, flash, computation, optimization
---

## Lookup Tables Over Runtime Computation

Pre-compute values and store in Flash ROM to avoid runtime computation. This trades Flash space for deterministic, fast access — critical when FPU is absent or cycle budgets are tight.

**Correct (pre-computed sine table):**

```c
/* Pre-computed sine table (0-90 degrees, 1-degree resolution) */
static const int16_t g_sinTable[91] =
{
    0, 175, 349, 523, /* ... values scaled by 10000 */
};

int16_t FastSin(uint16_t angleDeg)
{
    angleDeg = angleDeg % 360U;
    if (angleDeg <= 90U)  { return g_sinTable[angleDeg]; }
    if (angleDeg <= 180U) { return g_sinTable[180U - angleDeg]; }
    if (angleDeg <= 270U) { return -g_sinTable[angleDeg - 180U]; }
    return -g_sinTable[360U - angleDeg];
}
```

Use lookup tables for trigonometric functions, CRC tables, linearization curves, and sensor characteristic maps.
