---
title: Watchdog Monitoring Patterns
impact: HIGH
impactDescription: detects task deadlocks and runaway code
tags: safety, iso-26262, watchdog, alive-monitoring, supervision
---

## Watchdog Monitoring Patterns

Implement alive monitoring and logical supervision to detect software malfunctions such as task deadlocks, infinite loops, and incorrect execution sequences. Monitor both timing (deadline) and logical flow (sequence counter).

**Incorrect (no supervision of task execution):**

```c
void Task_10ms(void)
{
    ProcessData();
}
```

**Correct (watchdog supervision with timing and sequence checking):**

```c
typedef struct
{
    uint32_t expectedPeriodMs;
    uint32_t toleranceMs;
    uint32_t lastAliveTimestamp;
    uint16_t sequenceCounter;
    uint16_t expectedSequence;
    boolean  isActive;
} WdgSupervisor_t;

Std_ReturnType Wdg_CheckAlive(WdgSupervisor_t *sup, uint32_t currentTimeMs,
                               uint16_t seqCounter)
{
    uint32_t elapsed;

    if (!sup->isActive) { return E_NOT_OK; }

    elapsed = currentTimeMs - sup->lastAliveTimestamp;

    if (elapsed > (sup->expectedPeriodMs + sup->toleranceMs))
    {
        return E_NOT_OK;  /* Deadline missed */
    }

    if (seqCounter != sup->expectedSequence)
    {
        return E_NOT_OK;  /* Wrong execution order */
    }

    sup->lastAliveTimestamp = currentTimeMs;
    sup->expectedSequence = (seqCounter + 1U) % UINT16_MAX;
    return E_OK;
}
```

Combine alive supervision (timing) with logical supervision (sequence) for comprehensive fault detection.

Reference: ISO 26262 Part 6 — Program sequence monitoring (Alive supervision, Deadline monitoring, Logical supervision)
