---
title: Always Validate Buffer Boundaries
impact: CRITICAL
impactDescription: prevents memory corruption, security vulnerabilities
tags: memory, buffer, bounds-checking, safety, security, overflow
---

## Always Validate Buffer Boundaries

Every buffer access must be validated against the buffer size. In automotive ECUs without MMU protection, out-of-bounds writes silently corrupt adjacent memory.

**Incorrect (no bounds checking):**

```c
void CopyDtcData(uint8_t *dest, const uint8_t *src, uint16_t len)
{
    (void)memcpy(dest, src, len); /* len could exceed dest capacity */
}
```

**Correct (bounds-checked copy):**

```c
Std_ReturnType CopyDtcData(uint8_t *dest, uint16_t destSize,
                            const uint8_t *src, uint16_t srcLen)
{
    if ((dest == NULL) || (src == NULL))
    {
        return E_NOT_OK;
    }
    if (srcLen > destSize)
    {
        return E_NOT_OK;
    }
    (void)memcpy(dest, src, srcLen);
    return E_OK;
}
```

Reference: MISRA C:2012 Rule 21.17, ISO 26262 Part 6 Table 4
