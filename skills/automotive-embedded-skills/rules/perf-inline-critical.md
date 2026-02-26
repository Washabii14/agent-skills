---
title: Inline Critical Functions
impact: LOW
impactDescription: Eliminates function call overhead for small hot functions
tags: perf, inline, call-overhead, optimization, embedded
---

## Inline Critical Functions

Use `static inline` for small, frequently called functions to eliminate call overhead (push/pop, branch). Particularly effective for register access helpers and flag checks called from ISRs or tight loops.

**Incorrect (function call overhead for trivial operation):**

```c
uint8_t ReadStatusFlag(uint8_t mask)
{
    return (g_statusReg & mask);
}
```

**Correct (inlined for zero call overhead):**

```c
static inline uint8_t ReadStatusFlag(uint8_t mask)
{
    return (g_statusReg & mask);
}
```

Only inline small functions (1-3 lines). Excessive inlining increases code size and can degrade instruction cache performance, which is counterproductive on Flash-constrained MCUs.
