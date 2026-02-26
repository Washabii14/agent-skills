---
title: Loop Optimization for Embedded Targets
impact: MEDIUM
impactDescription: Reduces CPU cycles in tight embedded loops
tags: perf, loop, optimization, cpu-cycles, embedded
---

## Loop Optimization for Embedded Targets

Cache loop-invariant computations outside the loop body. Recalculating values on each iteration wastes CPU cycles on resource-constrained MCUs.

**Incorrect (recalculated on each iteration):**

```c
for (uint16_t i = 0U; i < GetArraySize(arr); i++)
{
    arr[i] = arr[i] * GetScaleFactor();
}
```

**Correct (cached values):**

```c
const uint16_t size = GetArraySize(arr);
const float scale = GetScaleFactor();
for (uint16_t i = 0U; i < size; i++)
{
    arr[i] = arr[i] * scale;
}
```

Hoisting loop-invariant function calls avoids redundant work and makes WCET analysis more tractable.
