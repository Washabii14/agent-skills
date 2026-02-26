---
title: Use Static Allocation for Deterministic Memory
impact: CRITICAL
impactDescription: predictable at compile time
tags: memory, static, allocation, determinism, linker, compile-time
---

## Use Static Allocation for Deterministic Memory

All memory usage should be determinable at compile time. Static allocation ensures the linker can verify total memory requirements fit within the target MCU.

**Incorrect (runtime-determined buffer):**

```c
void InitComStack(uint8_t numChannels)
{
    ComChannel_t *channels = malloc(numChannels * sizeof(ComChannel_t));
    /* ... */
}
```

**Correct (compile-time-determined buffer):**

```c
#define COM_MAX_CHANNELS (8U)

static ComChannel_t g_comChannels[COM_MAX_CHANNELS];
static uint8_t g_numActiveChannels = 0U;

Std_ReturnType InitComStack(uint8_t numChannels)
{
    if (numChannels > COM_MAX_CHANNELS)
    {
        return E_NOT_OK;
    }
    g_numActiveChannels = numChannels;
    (void)memset(g_comChannels, 0, sizeof(g_comChannels));
    return E_OK;
}
```

Reference: MISRA C:2012 Rule 21.3
