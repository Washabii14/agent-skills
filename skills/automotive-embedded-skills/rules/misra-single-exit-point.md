---
title: Single Exit Point for Critical Functions
impact: MEDIUM
impactDescription: improves traceability and code review
tags: misra, control-flow, single-exit, safety, verification, traceability
---

## Single Exit Point for Critical Functions

For safety-critical functions, a single exit point makes control flow analysis tractable for formal verification tools.

**Incorrect (multiple return points):**

```c
Std_ReturnType ValidateFrame(const CanFrame_t *frame)
{
    if (frame == NULL) return E_NOT_OK;
    if (frame->dlc > 8U) return E_NOT_OK;
    if (frame->id > 0x7FFU) return E_NOT_OK;
    return E_OK;
}
```

**Correct (single exit with result variable):**

```c
Std_ReturnType ValidateFrame(const CanFrame_t *frame)
{
    Std_ReturnType result = E_NOT_OK;

    if (frame != NULL)
    {
        if ((frame->dlc <= 8U) && (frame->id <= 0x7FFU))
        {
            result = E_OK;
        }
    }

    return result;
}
```

Reference: MISRA C:2012 Rule 15.5 (Advisory) — A function should have a single point of exit at the end.
