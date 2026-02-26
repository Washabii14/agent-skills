---
title: Cyclic Scheduling Patterns
impact: MEDIUM
impactDescription: ensures consistent timing and load distribution
tags: realtime, scheduling, cyclic-task, phase-offset, load-balancing
---

## Cyclic Scheduling Patterns

Distribute workload evenly across cycles using phase offsets. Not all sub-tasks need to run every cycle — use a phase counter to spread non-critical work across multiple invocations, keeping peak CPU load below the deadline budget.

**Incorrect (all work in every cycle):**

```c
void Task_10ms(void)
{
    ReadSensors();
    RunDiagnostics();
    UpdateNvm();
    ProcessBackgroundCalc();
}
```

**Correct (phase-distributed workload):**

```c
void Task_10ms(void)
{
    static uint8_t phase = 0U;

    ReadSensors();  /* Every 10ms */

    switch (phase)
    {
        case 0U: RunDiagnostics();     break;
        case 1U: UpdateNvm();          break;
        case 2U: ProcessBackgroundCalc(); break;
        default: /* No additional work */ break;
    }

    phase = (phase + 1U) % 3U;
}
```

This pattern effectively runs `RunDiagnostics()` every 30ms, `UpdateNvm()` every 30ms, etc., while keeping each 10ms cycle short. Only time-critical work (e.g., `ReadSensors()`) runs every cycle.

Reference: AUTOSAR OS — Alarm and schedule table patterns
