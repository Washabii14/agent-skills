---
title: AUTOSAR OS (OSEK) — Tasks, ISRs, Resources, Alarms, and Schedule Tables
impact: HIGH
impactDescription: OS misconfiguration causes priority inversion, stack overflow, missed deadlines, or ISR latency violations in safety-critical systems
tags: autosar, classic, os, osek, task, isr, resource, alarm, schedule-table, counter, trusted-function
---

## AUTOSAR OS (OSEK) — Tasks, ISRs, Resources, Alarms, and Schedule Tables

The AUTOSAR OS (R22-11) is based on OSEK/VDX OS with AUTOSAR extensions for memory protection, trusted functions, and timing protection. It is the scheduling backbone for all BSW modules and application SWCs. Misconfiguration directly impacts real-time behavior and safety.

### Task Configuration

**Incorrect (all tasks at same priority, no preemption strategy):**

```c
/* OIL / ARXML: All tasks at priority 1 — no deterministic scheduling */
/* TASK Task_10ms  { PRIORITY = 1; SCHEDULE = FULL; AUTOSTART = TRUE; }  */
/* TASK Task_50ms  { PRIORITY = 1; SCHEDULE = FULL; AUTOSTART = TRUE; }  */
/* TASK Task_100ms { PRIORITY = 1; SCHEDULE = FULL; AUTOSTART = TRUE; }  */
```

**Correct (rate-monotonic priority assignment):**

```c
/* Higher frequency = higher priority (Rate Monotonic Scheduling) */

/* TASK Task_1ms   { PRIORITY = 10; SCHEDULE = FULL; ACTIVATION = 1; AUTOSTART = FALSE; } */
/* TASK Task_10ms  { PRIORITY = 5;  SCHEDULE = FULL; ACTIVATION = 1; AUTOSTART = FALSE; } */
/* TASK Task_100ms { PRIORITY = 2;  SCHEDULE = FULL; ACTIVATION = 1; AUTOSTART = FALSE; } */
/* TASK Task_Idle  { PRIORITY = 1;  SCHEDULE = FULL; ACTIVATION = 1; AUTOSTART = TRUE;  } */

TASK(Task_10ms)
{
    /* BSW MainFunctions + application runnables at 10ms rate */
    Com_MainFunctionTx();
    Com_MainFunctionRx();
    BswM_MainFunction();
    App_EngineControl_10ms();

    TerminateTask();
}
```

### ISR Categories

**Incorrect (calling OS services from Category 1 ISR):**

```c
/* Cat1 ISR — runs outside OS context, no OS API allowed */
ISR(Timer_ISR_Cat1)
{
    SetEvent(Task_10ms, EVT_TIMER);  /* ILLEGAL in Cat1 — undefined behavior */
    g_timerCount++;
}
```

**Correct (Cat1 for minimal latency, Cat2 for OS interaction):**

```c
/* Category 1: Minimal latency, no OS API, not managed by OS */
ISR1(ADC_ConvComplete_ISR)
{
    volatile uint16 *result = (volatile uint16 *)ADC_RESULT_REG;
    g_adcBuffer[g_adcIndex++] = *result;
    /* Only direct hardware access — no OS calls */
}

/* Category 2: Managed by OS, can use limited OS API */
ISR2(CAN_RxInterrupt_ISR)
{
    CanFrame_t frame;
    Can_ReadHwBuffer(&frame);
    CanIf_RxIndication(frame.id, &frame.data);

    /* Allowed: SetEvent, ActivateTask, GetResource, ReleaseResource */
    /* Forbidden: WaitEvent, TerminateTask, Schedule */
}
```

### Resource Management (Priority Ceiling Protocol)

**Incorrect (nested resources in wrong order — deadlock risk):**

```c
TASK(Task_A)
{
    GetResource(RES_UART);
    GetResource(RES_SPI);     /* Must always acquire in consistent order */
    /* ... */
    ReleaseResource(RES_UART); /* WRONG order — must be LIFO (SPI first) */
    ReleaseResource(RES_SPI);
    TerminateTask();
}
```

**Correct (LIFO resource acquisition/release):**

```c
TASK(Task_A)
{
    GetResource(RES_SPI);
    GetResource(RES_UART);    /* Inner resource acquired second */
    /* ... critical section using both SPI and UART ... */
    ReleaseResource(RES_UART); /* LIFO: release inner first */
    ReleaseResource(RES_SPI);  /* Then release outer */
    TerminateTask();
}

/* OS uses Immediate Priority Ceiling Protocol:
 * Task priority raised to ceiling of resource while held
 * Prevents priority inversion without blocking */
```

### Alarms and Schedule Tables

**Incorrect (using software delays for periodic activation):**

```c
TASK(Task_Periodic)
{
    while (1)
    {
        App_Cyclic();
        /* Software delay — wastes CPU, no preemption, jitter accumulates */
        for (volatile uint32 i = 0; i < DELAY_COUNT; i++) {}
    }
}
```

**Correct (alarm-driven periodic activation):**

```c
/* ALARM Alarm_10ms {
 *   COUNTER = SystemCounter;
 *   ACTION = ACTIVATETASK { TASK = Task_10ms; };
 *   AUTOSTART = TRUE {
 *     ALARMTIME = 10;    -- Initial offset (ticks)
 *     CYCLETIME = 10;    -- Period (ticks)
 *   };
 * } */

/* Schedule Table: for synchronized, phased activation */
/* SCHEDULETABLE ST_5ms {
 *   COUNTER = SystemCounter;
 *   DURATION = 20;     -- Table length in ticks
 *   REPEATING = TRUE;
 *   EXPIRY_POINT EP_0  { OFFSET = 0;  ACTION = ACTIVATETASK { TASK = Task_ADC; }; };
 *   EXPIRY_POINT EP_5  { OFFSET = 5;  ACTION = ACTIVATETASK { TASK = Task_Control; }; };
 *   EXPIRY_POINT EP_10 { OFFSET = 10; ACTION = ACTIVATETASK { TASK = Task_COM; }; };
 *   EXPIRY_POINT EP_15 { OFFSET = 15; ACTION = ACTIVATETASK { TASK = Task_Diag; }; };
 * } */

/* Schedule tables enable deterministic phasing — tasks don't pile up */
```

### Trusted Functions (Memory Protection)

```c
/* In OS-Application with memory protection, untrusted apps cannot
 * access hardware directly. Use trusted functions: */

/* Trusted function declaration (runs in privileged mode) */
StatusType TRUSTED_Hw_WriteRegister(
    TrustedFunctionIndexType FunctionIndex,
    TrustedFunctionParameterRefType Params)
{
    const HwWriteParams_t *p = (const HwWriteParams_t *)Params;
    *(volatile uint32 *)p->address = p->value;
    return E_OK;
}

/* Untrusted app calls through OS: */
TASK(Task_App_Untrusted)
{
    HwWriteParams_t params = { .address = GPIO_PORT_A, .value = 0x01u };
    CallTrustedFunction(TRUSTED_IDX_HW_WRITE, &params);
    TerminateTask();
}
```

Reference: AUTOSAR Classic R22-11 SWS_OS (SWS OS), OSEK/VDX OS Specification 2.2.3
