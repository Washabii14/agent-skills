---
title: Task Stack Sizing
impact: HIGH
impactDescription: Undersized stacks cause silent corruption; oversized stacks waste scarce RAM
tags: rtos, stack, memory, sizing, overflow
---

## Task Stack Sizing

Calculate task stack size based on call depth analysis plus safety margin (typically 20-30%). Stack overflow in embedded systems without MMU protection silently corrupts adjacent memory, causing unpredictable failures.

**Incorrect (arbitrary stack sizes):**

```c
OS_CreateTask(TaskControl, 256U, PRIORITY_HIGH);   /* Too small? */
OS_CreateTask(TaskLogging, 4096U, PRIORITY_LOW);    /* Wasteful? */
```

**Correct (analyzed stack size with documented margin):**

```c
#define TASK_CONTROL_STACK_SIZE  (1024U + 256U)  /* Analyzed 1024 + 25% margin */

OS_CreateTask(TaskControl, TASK_CONTROL_STACK_SIZE, PRIORITY_HIGH);
```

Use static analysis tools (e.g., stack usage reports from the compiler with `-fstack-usage`) to determine actual maximum call depth. Add a safety margin of 20-30% for ISR nesting and compiler-generated temporaries. Fill stacks with a known pattern at startup and periodically check watermarks at runtime to detect near-overflow conditions.

```c
void CheckStackWatermark(OS_TaskId_t taskId)
{
    uint32_t usedBytes = OS_GetStackUsage(taskId);
    uint32_t totalBytes = OS_GetStackSize(taskId);

    if (usedBytes > (totalBytes * 80U / 100U))
    {
        ReportError(ERR_STACK_HIGH_WATERMARK);
    }
}
```

Reference: AUTOSAR OS Specification — Task stack configuration, ISO 26262 Part 6
