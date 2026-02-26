---
title: Use volatile Correctly
impact: CRITICAL
impactDescription: prevents compiler optimization of hardware access
tags: memory, volatile, hardware, registers, isr, compiler-optimization
---

## Use volatile Correctly

`volatile` tells the compiler the value may change outside the program flow. Required for: hardware registers, shared variables modified by ISRs, and memory-mapped I/O.

**Incorrect (compiler may optimize away the read):**

```c
uint32_t WaitForFlag(void)
{
    uint32_t *statusReg = (uint32_t *)0x40020010U;
    while ((*statusReg & 0x01U) == 0U)
    {
        /* Compiler may read statusReg once and loop forever */
    }
    return *statusReg;
}
```

**Correct (volatile forces re-read):**

```c
uint32_t WaitForFlag(void)
{
    volatile uint32_t *statusReg = (volatile uint32_t *)0x40020010U;
    while ((*statusReg & 0x01U) == 0U)
    {
        /* Each iteration re-reads the hardware register */
    }
    return *statusReg;
}
```

`volatile` does NOT provide atomicity or memory ordering. For shared data between ISR and task, also use critical sections or atomic operations.

Reference: MISRA C:2012 Rule 8.13, ARM Cortex-M Programming Manual
