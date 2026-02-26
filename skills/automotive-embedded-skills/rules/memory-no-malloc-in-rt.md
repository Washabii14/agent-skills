---
title: Never Use malloc/free in Real-Time Critical Paths
impact: CRITICAL
impactDescription: non-deterministic latency, heap fragmentation
tags: memory, malloc, free, real-time, determinism, heap, latency
---

## Never Use malloc/free in Real-Time Critical Paths

`malloc()` has non-deterministic execution time. In a 10ms cyclic task, a single `malloc()` can cause deadline overrun due to heap fragmentation and searching.

**Incorrect (heap allocation in cyclic task):**

```c
void Cyclic10ms(void)
{
    uint8_t *buf = (uint8_t *)malloc(MSG_SIZE);
    if (buf == NULL)
    {
        return; /* Deadline already at risk */
    }
    ProcessMessage(buf);
    free(buf);
}
```

**Correct (static or stack allocation):**

```c
static uint8_t g_msgBuffer[MSG_SIZE];

void Cyclic10ms(void)
{
    ProcessMessage(g_msgBuffer);
}
```

Use stack allocation, static buffers, or memory pools instead. See the Memory Pool Pattern rule for cases requiring dynamic-like allocation.

Reference: MISRA C:2012 Rule 21.3 — The memory allocation and deallocation functions of `<stdlib.h>` shall not be used.
