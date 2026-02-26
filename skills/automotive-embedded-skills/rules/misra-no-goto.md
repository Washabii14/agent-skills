---
title: No goto Except for Error Cleanup in C
impact: LOW
impactDescription: maintains readability
tags: misra, goto, control-flow, error-handling, cleanup, readability
---

## No goto Except for Error Cleanup in C

In C, `goto` is acceptable only for forward jumps to a common cleanup label at the end of a function. All other uses of `goto` shall be avoided.

**Acceptable (error cleanup pattern):**

```c
Std_ReturnType ProcessMessage(const uint8_t *data, uint16_t len)
{
    Std_ReturnType ret = E_NOT_OK;
    MsgBuffer_t *buf = NULL;

    buf = MsgPool_Alloc();
    if (buf == NULL) { goto cleanup; }

    if (DecodeMessage(buf, data, len) != E_OK) { goto cleanup; }
    if (ValidateMessage(buf) != E_OK) { goto cleanup; }

    ret = RouteMessage(buf);

cleanup:
    if (buf != NULL) { MsgPool_Free(buf); }
    return ret;
}
```

This pattern consolidates cleanup logic at a single point, preventing resource leaks when multiple operations can fail. In C++, prefer RAII instead.

Reference: MISRA C:2012 Rule 15.1 — The goto statement should not be used.
