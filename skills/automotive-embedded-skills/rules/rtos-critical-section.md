---
title: Minimize Critical Section Duration
impact: HIGH
impactDescription: Large critical sections increase blocking time and can cause deadline misses in higher-priority tasks
tags: rtos, critical-section, concurrency, blocking, real-time
---

## Minimize Critical Section Duration

Critical sections disable interrupts or block preemption. Keeping them as short as possible reduces worst-case blocking time for all higher-priority tasks. Only protect the actual shared data access — never include I/O, computation, or function calls inside critical sections.

**Incorrect (large critical section with I/O and computation):**

```c
void UpdateSharedData(void)
{
    OS_EnterCritical();
    ReadAllSensors();             /* Expensive I/O in critical section */
    CalculateOutputs();           /* CPU-intensive computation */
    g_sharedOutput = output;      /* Actual shared data access */
    OS_ExitCritical();
}
```

**Correct (minimal critical section protecting only shared access):**

```c
void UpdateSharedData(void)
{
    SensorData_t localSensors;
    float localOutput;

    ReadAllSensors(&localSensors);
    localOutput = CalculateOutputs(&localSensors);

    OS_EnterCritical();
    g_sharedOutput = localOutput;  /* Only protect the shared access */
    OS_ExitCritical();
}
```

Perform all computation and I/O into local variables first, then enter the critical section only to copy the result to shared memory. This pattern minimizes the window where interrupts are disabled.

Reference: AUTOSAR OS Specification — Resource management and interrupt handling
