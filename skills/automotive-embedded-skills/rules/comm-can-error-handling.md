---
title: CAN Bus-Off Recovery and Error Handling
impact: HIGH
impactDescription: ensures communication resilience
tags: comm, can, bus-off, error-handling, recovery, dtc
---

## CAN Bus-Off Recovery and Error Handling

Implement proper bus-off recovery with backoff strategy. A CAN controller enters bus-off state after accumulating too many transmit errors (TEC > 255). Without recovery logic, the ECU permanently loses CAN communication.

**Incorrect (no error handling):**

```c
void CAN_ErrorCallback(Can_ErrorType error)
{
    /* Nothing — bus-off silently kills communication */
}
```

**Correct (structured error recovery with backoff):**

```c
void CAN_ErrorCallback(Can_ErrorType error)
{
    switch (error)
    {
        case CAN_ERROR_BUSOFF:
            g_canBusOffCount++;
            if (g_canBusOffCount < MAX_BUSOFF_RECOVERY)
            {
                Timer_Start(TIMER_BUSOFF_RECOVERY, BUSOFF_DELAY_MS);
            }
            else
            {
                ReportDtc(DTC_CAN_BUSOFF_PERMANENT);
                EnterDegradedMode();
            }
            break;

        case CAN_ERROR_PASSIVE:
            ReportDtc(DTC_CAN_ERROR_PASSIVE);
            break;

        default:
            break;
    }
}
```

Key recovery considerations:
- Use increasing backoff delay between recovery attempts
- Limit total recovery attempts before declaring permanent failure
- Report DTCs for all error states (bus-off, error-passive, error-warning)
- Enter degraded mode if recovery fails (disable non-critical CAN-dependent features)

Reference: ISO 11898-1 — CAN error management; AUTOSAR CanSM — CAN State Manager
