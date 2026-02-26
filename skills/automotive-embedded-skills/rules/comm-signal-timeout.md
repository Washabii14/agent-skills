---
title: Signal Timeout Monitoring
impact: HIGH
impactDescription: detects communication loss
tags: comm, signal-timeout, monitoring, default-value, dtc, freshness
---

## Signal Timeout Monitoring

Monitor received signal freshness and substitute default values on timeout. This applies to all bus types (CAN, LIN, Ethernet). A missing signal could indicate sender ECU failure, bus fault, or wiring issue.

**Incorrect (using last received value indefinitely):**

```c
float g_vehicleSpeed = 0.0f;

void OnSpeedReceived(float speed)
{
    g_vehicleSpeed = speed;
}

float GetVehicleSpeed(void)
{
    return g_vehicleSpeed;  /* May be stale */
}
```

**Correct (timeout-monitored signal with default substitution):**

```c
typedef struct
{
    uint32_t lastRxTimestamp;
    uint16_t timeoutMs;
    boolean  isTimedOut;
    float    defaultValue;
} SignalMonitor_t;

float GetMonitoredSignal(SignalMonitor_t *mon, float rxValue,
                          uint32_t currentTime)
{
    if (mon->isTimedOut)
    {
        return mon->defaultValue;
    }

    uint32_t elapsed = currentTime - mon->lastRxTimestamp;
    if (elapsed > mon->timeoutMs)
    {
        mon->isTimedOut = TRUE;
        ReportDtc(DTC_SIGNAL_TIMEOUT);
        return mon->defaultValue;
    }

    mon->lastRxTimestamp = currentTime;
    return rxValue;
}
```

Timeout values are typically 3x the expected cycle time (e.g., 30ms timeout for a 10ms cyclic signal). On timeout, set the signal to a safe default and raise a DTC.

Reference: AUTOSAR COM — Signal timeout monitoring (ComTimeout, ComRxDataTimeoutAction)
