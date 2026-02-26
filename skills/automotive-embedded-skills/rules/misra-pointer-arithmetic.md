---
title: Restrict Pointer Arithmetic
impact: MEDIUM
impactDescription: prevents out-of-bounds access
tags: misra, pointer, arithmetic, bounds-checking, array, safety
---

## Restrict Pointer Arithmetic

Pointer arithmetic should only be used for array indexing. Never compare pointers from different objects. Always validate indices against buffer size.

**Incorrect (arbitrary pointer arithmetic):**

```c
void ProcessData(uint8_t *data, uint16_t offset)
{
    uint8_t *target = data + offset;  /* Could point anywhere */
    *target = 0xFFU;
}
```

**Correct (array indexing with bounds check):**

```c
void ProcessData(uint8_t data[], uint16_t dataSize, uint16_t offset)
{
    if (offset < dataSize)
    {
        data[offset] = 0xFFU;
    }
}
```

Reference: MISRA C:2012 Rule 18.1–18.4
