---
title: Deadline Monitoring for Critical Tasks
impact: HIGH
impactDescription: ASIL compliance for timing supervision
tags: realtime, deadline, monitoring, timing-supervision, asil
---

## Deadline Monitoring for Critical Tasks

Monitor that critical tasks complete within their allocated time budget. Measure elapsed time at task entry and exit, and report an error if the deadline is exceeded. This is a required safety mechanism for ASIL B and above.

**Incorrect (no timing supervision):**

```c
void Task_5ms(void)
{
    ExecuteControlLoop();
}
```

**Correct (deadline-monitored task):**

```c
void Task_5ms(void)
{
    uint32_t startTime = Timer_GetCurrentUs();

    /* Task body */
    ExecuteControlLoop();

    uint32_t elapsed = Timer_GetCurrentUs() - startTime;
    if (elapsed > TASK_5MS_DEADLINE_US)
    {
        ReportError(ERR_DEADLINE_OVERRUN);
    }
}
```

For comprehensive monitoring, combine:
- **Deadline monitoring**: task must complete within its period
- **Alive supervision**: task must run at its expected rate
- **Logical supervision**: task must execute checkpoints in correct order

AUTOSAR Watchdog Manager (WdgM) provides standardized APIs for all three.

Reference: ISO 26262 Part 6 — Program sequence monitoring; AUTOSAR SWS Watchdog Manager
