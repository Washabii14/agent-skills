---
title: Bitwise Operations for Flags and Registers
impact: MEDIUM
impactDescription: Efficient register manipulation and flag management
tags: perf, bitwise, flags, registers, hardware
---

## Bitwise Operations for Flags and Registers

Use bitwise operations for hardware register access and status flags. This is more efficient and idiomatic for embedded targets than arithmetic or boolean arrays.

**Correct (bitwise flag operations):**

```c
#define STATUS_FLAG_READY    ((uint8_t)0x01U)
#define STATUS_FLAG_ERROR    ((uint8_t)0x02U)
#define STATUS_FLAG_BUSY     ((uint8_t)0x04U)

static uint8_t g_statusFlags = 0U;

static inline void SetFlag(uint8_t flag)   { g_statusFlags |= flag; }
static inline void ClearFlag(uint8_t flag) { g_statusFlags &= (uint8_t)~flag; }
static inline boolean IsFlagSet(uint8_t flag)
{
    return ((g_statusFlags & flag) != 0U);
}
```

Always use explicit casts and unsigned types when performing bitwise operations to avoid MISRA violations and undefined behavior from signed shifts.
