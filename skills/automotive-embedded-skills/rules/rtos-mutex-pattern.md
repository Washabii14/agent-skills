---
title: Correct Mutex Usage
impact: HIGH
impactDescription: Incorrect mutex usage causes deadlocks that can lock up safety-critical systems
tags: rtos, mutex, deadlock, concurrency, synchronization
---

## Correct Mutex Usage

Always acquire mutexes in the same global order. Never hold multiple mutexes simultaneously if possible. Deadlocks in automotive ECUs are catastrophic — there is no user to restart the system.

**Incorrect (inconsistent lock ordering causes deadlock):**

```c
void Task_A(void)
{
    OS_LockMutex(mutexX);
    OS_LockMutex(mutexY);
    /* ... */
    OS_UnlockMutex(mutexY);
    OS_UnlockMutex(mutexX);
}

void Task_B(void)
{
    OS_LockMutex(mutexY);  /* Opposite order from Task_A */
    OS_LockMutex(mutexX);  /* Deadlock if Task_A holds X, Task_B holds Y */
    /* ... */
    OS_UnlockMutex(mutexX);
    OS_UnlockMutex(mutexY);
}
```

**Correct (consistent global lock ordering):**

```c
/* Global lock ordering: mutexX < mutexY — always acquire X before Y */

void Task_A(void)
{
    OS_LockMutex(mutexX);
    OS_LockMutex(mutexY);
    /* ... */
    OS_UnlockMutex(mutexY);
    OS_UnlockMutex(mutexX);
}

void Task_B(void)
{
    OS_LockMutex(mutexX);  /* Same order as Task_A */
    OS_LockMutex(mutexY);
    /* ... */
    OS_UnlockMutex(mutexY);
    OS_UnlockMutex(mutexX);
}
```

Use timeouts on mutex acquisition to detect and recover from potential deadlocks. Prefer designs that require only a single mutex per critical resource.

Reference: AUTOSAR OS Specification — Resource management
