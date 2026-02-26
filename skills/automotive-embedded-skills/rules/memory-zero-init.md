---
title: Always Initialize Variables
impact: HIGH
impactDescription: prevents undefined behavior in safety-critical code
tags: memory, initialization, undefined-behavior, safety, defaults
---

## Always Initialize Variables

Uninitialized variables produce undefined behavior. In safety-critical automotive code, every variable must have a defined initial value.

**Incorrect (uninitialized local):**

```c
Std_ReturnType ReadSensor(uint16_t *value)
{
    Std_ReturnType ret;  /* Uninitialized — UB if no branch sets it */
    uint16_t rawVal;

    if (HAL_ADC_Read(&rawVal) == HAL_OK)
    {
        *value = rawVal;
        ret = E_OK;
    }
    /* If HAL_ADC_Read fails, ret is garbage */
    return ret;
}
```

**Correct (initialized with safe defaults):**

```c
Std_ReturnType ReadSensor(uint16_t *value)
{
    Std_ReturnType ret = E_NOT_OK;
    uint16_t rawVal = 0U;

    if (value == NULL)
    {
        return E_NOT_OK;
    }

    if (HAL_ADC_Read(&rawVal) == HAL_OK)
    {
        *value = rawVal;
        ret = E_OK;
    }
    else
    {
        *value = 0U;
    }
    return ret;
}
```

Reference: MISRA C:2012 Rule 9.1 — The value of an object with automatic storage duration shall not be read before it has been set.
