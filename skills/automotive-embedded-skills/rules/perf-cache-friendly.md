---
title: Cache-Friendly Data Organization
impact: MEDIUM
impactDescription: Improves performance on high-end automotive MCUs with cache
tags: perf, cache, data-layout, memory, optimization
---

## Cache-Friendly Data Organization

Organize frequently accessed data contiguously for cache line efficiency. On high-end automotive MCUs (e.g., Renesas RH850, Infineon AURIX TC3xx) with data caches, layout matters for throughput.

**Incorrect (scattered access pattern):**

```c
typedef struct
{
    float   sensorValue;
    char    description[64];
    uint8_t flags;
    float   filteredValue;
    char    unit[16];
    float   minLimit;
    float   maxLimit;
} SensorData_t;
```

**Correct (hot/cold data separation):**

```c
typedef struct
{
    float   sensorValue;
    float   filteredValue;
    float   minLimit;
    float   maxLimit;
    uint8_t flags;
} SensorHotData_t;

typedef struct
{
    char description[64];
    char unit[16];
} SensorColdData_t;
```

Group fields accessed together (hot data) separately from rarely accessed fields (cold data). This minimizes cache line waste in tight cyclic tasks.
