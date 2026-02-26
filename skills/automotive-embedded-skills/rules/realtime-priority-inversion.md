---
title: Prevent Priority Inversion
impact: HIGH
impactDescription: prevents low-priority task blocking high-priority task
tags: realtime, priority-inversion, mutex, priority-inheritance, priority-ceiling
---

## Prevent Priority Inversion

Use priority inheritance or priority ceiling protocol when high-priority tasks share resources with lower-priority tasks. Without these mechanisms, a medium-priority task can preempt a low-priority task holding a resource needed by a high-priority task, causing unbounded blocking.

**Incorrect (standard mutex without priority inheritance):**

```c
OS_MutexCreate(&sharedMutex, MUTEX_NO_PRIORITY_INHERIT);

void HighPriorityTask(void)
{
    OS_MutexLock(&sharedMutex);  /* May block indefinitely */
    AccessSharedResource();
    OS_MutexUnlock(&sharedMutex);
}
```

**Correct (priority inheritance mutex):**

```c
OS_MutexCreate(&sharedMutex, MUTEX_PRIORITY_INHERIT);

void HighPriorityTask(void)
{
    OS_MutexLock(&sharedMutex);
    AccessSharedResource();
    OS_MutexUnlock(&sharedMutex);
}
```

Alternatively, use the priority ceiling protocol (immediate priority ceiling) where the mutex holder's priority is immediately raised to the highest priority of any task that may lock the mutex. This prevents chained blocking.

Where possible, avoid shared resources entirely by using message queues for inter-task communication.

Reference: AUTOSAR OS — Resource management (Priority ceiling protocol); OSEK/VDX OS Specification
