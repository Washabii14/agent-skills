---
title: Priority Inheritance Protocols
impact: HIGH
impactDescription: Priority inversion can cause high-priority safety tasks to miss deadlines
tags: rtos, priority-inversion, priority-inheritance, mutex, scheduling
---

## Priority Inheritance Protocols

Use RTOS features like priority inheritance mutexes for shared resources between tasks of different priorities. Without priority inheritance, a low-priority task holding a mutex can block a high-priority task indefinitely if a medium-priority task preempts the low-priority one (unbounded priority inversion).

**Incorrect (standard mutex allows unbounded priority inversion):**

```c
OS_MutexCreate(&sharedMutex, OS_MUTEX_NORMAL);

void Task_HighPriority(void)
{
    OS_LockMutex(sharedMutex);  /* Blocks if low-priority task holds it */
    AccessSharedResource();
    OS_UnlockMutex(sharedMutex);
}
```

**Correct (priority inheritance mutex bounds the inversion):**

```c
OS_MutexCreate(&sharedMutex, OS_MUTEX_PRIORITY_INHERIT);

void Task_HighPriority(void)
{
    OS_LockMutex(sharedMutex);
    AccessSharedResource();
    OS_UnlockMutex(sharedMutex);
}
```

With priority inheritance, when the high-priority task blocks on the mutex, the RTOS temporarily raises the low-priority holder's priority to prevent medium-priority tasks from preempting it. The priority ceiling protocol is an alternative that avoids deadlock entirely by raising priority on acquisition.

Reference: AUTOSAR OS Specification — Priority ceiling protocol, ISO 26262 Part 6
