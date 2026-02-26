---
title: Error Frame Handling in Simulation
impact: MEDIUM
impactDescription: Unhandled error frames cause unrealistic simulation behavior and missed failure scenarios
tags: capl, error-frame, simulation, bus-off, can, robustness
---

## Error Frame Handling in Simulation

Handle error frames and bus-off conditions in simulation nodes. Realistic simulation requires responding to error conditions the same way the real ECU would.

**Incorrect (no error handling in simulation):**

```capl
/* Simulation only sends messages, ignores all errors */
on timer cyclicTimer
{
    output(msg_EngineStatus);
    setTimer(cyclicTimer, 10);
}
```

**Correct (error frame detection and bus-off recovery):**

```capl
variables
{
    int busOffCount = 0;
    int isBusOff = 0;
}

on errorFrame
{
    write("Error frame detected on %s", this.msgChannel);
}

on busOff
{
    busOffCount++;
    isBusOff = 1;
    write("Bus-Off detected! Count: %d", busOffCount);

    if (busOffCount < 3)
    {
        setTimer(recoveryTimer, 500);
    }
    else
    {
        write("Permanent bus-off — entering degraded mode");
    }
}

on timer recoveryTimer
{
    resetCan();
    isBusOff = 0;
}

on timer cyclicTimer
{
    if (!isBusOff)
    {
        output(msg_EngineStatus);
    }
    setTimer(cyclicTimer, 10);
}
```

Implement `on errorFrame` and `on busOff` handlers to track CAN error states. Suppress message output during bus-off and implement recovery with backoff to match real ECU behavior.

Reference: Vector CANoe/CANalyzer CAPL Programming Guide — Error Handling
