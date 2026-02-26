---
title: Clean Module Interfaces
impact: MEDIUM
impactDescription: Information hiding prevents unintended coupling and reduces the impact of internal changes
tags: arch, module, interface, information-hiding, encapsulation, header
---

## Clean Module Interfaces

Each module exposes only its public API through a header. Internal state and helper functions are static. This enforces information hiding and prevents other modules from depending on implementation details.

**Incorrect (internal details exposed in header):**

```c
/* sensor_module.h — too much exposed */
extern float g_sensorBuffer[32];
extern uint8_t g_sensorIndex;
void SensorModule_InternalFilter(float *data, uint8_t len);
void SensorModule_Init(void);
float SensorModule_GetValue(void);
```

**Correct (clean public interface, internals are static):**

```c
/* sensor_module.h — public API only */
void SensorModule_Init(void);
float SensorModule_GetValue(void);
Std_ReturnType SensorModule_GetStatus(SensorStatus_t *status);
```

```c
/* sensor_module.c — internals hidden with static */
static float g_sensorBuffer[32];
static uint8_t g_sensorIndex = 0U;

static void InternalFilter(float *data, uint8_t len)
{
    /* Only accessible within this translation unit */
}

void SensorModule_Init(void)
{
    g_sensorIndex = 0U;
    (void)memset(g_sensorBuffer, 0, sizeof(g_sensorBuffer));
}

float SensorModule_GetValue(void)
{
    InternalFilter(g_sensorBuffer, g_sensorIndex);
    return g_sensorBuffer[g_sensorIndex];
}
```

Use the module name as a prefix for all public functions (`SensorModule_*`). Keep headers minimal — only types, constants, and function prototypes that external users need.

Reference: MISRA C:2012 Dir 4.8 — Implementation of objects should be hidden, AUTOSAR C++14 Rule A3-3-1
