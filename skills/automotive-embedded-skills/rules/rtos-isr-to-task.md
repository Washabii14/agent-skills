---
title: Defer ISR Processing to Task Context
impact: HIGH
impactDescription: Long ISR execution blocks all lower-priority interrupts and tasks, risking deadline misses
tags: rtos, isr, deferred-processing, interrupt, real-time
---

## Defer ISR Processing to Task Context

ISRs should only capture data and signal a task. All processing happens in task context. Keeping ISRs short ensures that higher-priority interrupts are not blocked and task scheduling remains predictable.

**Incorrect (heavy processing in ISR):**

```c
void CAN_RxISR(void)
{
    CanFrame_t frame = CAN_ReadMailbox();
    ProcessFrame(&frame);    /* Could take >100us */
    WriteToFlash(&frame);    /* Blocking flash write in ISR! */
}
```

**Correct (minimal ISR, deferred processing to task):**

```c
void CAN_RxISR(void)
{
    CanFrame_t frame = CAN_ReadMailbox();
    RingBuffer_Put(&g_canRxBuffer, &frame);
    OS_SetEvent(TASK_CAN_PROCESS, EVT_CAN_RX);
}

void Task_CanProcess(void)
{
    OS_WaitEvent(EVT_CAN_RX);
    OS_ClearEvent(EVT_CAN_RX);

    CanFrame_t frame;
    while (RingBuffer_Get(&g_canRxBuffer, &frame))
    {
        ProcessFrame(&frame);
    }
}
```

The ISR reads the hardware and buffers the data, then signals the processing task. The task context handles all validation, routing, and storage operations where blocking and longer execution times are acceptable.

Reference: AUTOSAR OS Specification — Category 2 ISRs, ISO 26262 Part 6
