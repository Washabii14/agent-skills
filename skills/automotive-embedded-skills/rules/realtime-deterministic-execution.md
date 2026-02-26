---
title: Deterministic Execution in Cyclic Tasks
impact: CRITICAL
impactDescription: ensures deadline compliance
tags: realtime, timing, deterministic, cyclic-task, bounded-execution
---

## Deterministic Execution in Cyclic Tasks

Cyclic task execution time must be bounded and deterministic. Avoid data-dependent loops, dynamic allocation, and any unbounded operations inside cyclic tasks. Always limit the number of iterations per cycle.

**Incorrect (unbounded loop in cyclic task):**

```c
void Task_10ms(void)
{
    while (MessageQueue_HasPending())  /* Unbounded — could overrun */
    {
        ProcessNextMessage();
    }
}
```

**Correct (bounded processing per cycle):**

```c
#define MAX_MESSAGES_PER_CYCLE (5U)

void Task_10ms(void)
{
    uint8_t count = 0U;
    while ((MessageQueue_HasPending()) && (count < MAX_MESSAGES_PER_CYCLE))
    {
        ProcessNextMessage();
        count++;
    }
}
```

Remaining messages are processed in subsequent cycles. The upper bound ensures the task completes within its allocated time budget regardless of input volume.

Reference: ISO 26262 Part 6 — Timing constraints; AUTOSAR OS timing protection
