---
title: Never Block in ISRs
impact: CRITICAL
impactDescription: preserves system responsiveness
tags: realtime, isr, interrupt, non-blocking, deferred-processing
---

## Never Block in ISRs

ISRs must execute quickly and never wait for resources. Defer complex processing to task context. Blocking operations (flash writes, mutexes, long computations) in an ISR delay all lower-priority interrupts and can cause deadline overruns across the entire system.

**Incorrect (blocking in ISR):**

```c
void CAN_RxISR(void)
{
    CanFrame_t frame = CAN_ReadMailbox();
    ProcessFrame(&frame);    /* Could take >100us */
    WriteToFlash(&frame);    /* Blocking flash write in ISR! */
}
```

**Correct (minimal ISR, deferred processing):**

```c
void CAN_RxISR(void)
{
    CanFrame_t frame = CAN_ReadMailbox();
    RingBuffer_Put(&g_canRxBuffer, &frame);
    OS_SetEvent(TASK_CAN_PROCESS, EVT_CAN_RX);
}
```

ISR responsibilities should be limited to:
1. Read hardware register / mailbox
2. Store data in a lock-free buffer (ring buffer, FIFO)
3. Signal a task event
4. Clear interrupt flag and return

All processing, validation, and I/O happen in task context where preemption is managed by the RTOS scheduler.

Reference: AUTOSAR OS specification — Category 1/2 ISR constraints
