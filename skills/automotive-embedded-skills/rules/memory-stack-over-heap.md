---
title: Prefer Stack Allocation Over Heap
impact: CRITICAL
impactDescription: deterministic, no fragmentation
tags: memory, stack, heap, allocation, determinism, embedded
---

## Prefer Stack Allocation Over Heap

In embedded systems, stack allocation is deterministic in both time and space. Heap allocation introduces non-deterministic latency and fragmentation risk. In automotive ECUs with no MMU, heap fragmentation can lead to silent memory corruption.

**Incorrect (heap allocation in periodic function):**

```c
void ProcessSensorData(void)
{
    float *buffer = (float *)malloc(SENSOR_COUNT * sizeof(float));
    if (buffer == NULL)
    {
        /* Error: no memory — but damage may already be done */
        return;
    }
    ReadSensors(buffer, SENSOR_COUNT);
    ProcessBuffer(buffer, SENSOR_COUNT);
    free(buffer);
}
```

**Correct (stack allocation with known bounds):**

```c
void ProcessSensorData(void)
{
    float buffer[SENSOR_COUNT]; /* Stack-allocated, deterministic */
    ReadSensors(buffer, SENSOR_COUNT);
    ProcessBuffer(buffer, SENSOR_COUNT);
}
```

For larger buffers that cannot fit on stack, use statically allocated module-level buffers.

Reference: MISRA C:2012 Rule 21.3 — The memory allocation and deallocation functions of `<stdlib.h>` shall not be used.
