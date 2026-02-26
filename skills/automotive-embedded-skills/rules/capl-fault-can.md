---
title: CAN/CAN FD Fault Injection Patterns
impact: HIGH
impactDescription: Missing fault injection coverage leaves ECU error handling untested, risking safety-critical failures in the field
tags: capl, fault-injection, can, can-fd, error-frame, bus-off, signal, timing
---

## CAN/CAN FD Fault Injection Patterns

Systematic fault injection on CAN/CAN FD validates ECU robustness against bus errors, signal corruption, and timing violations. Tests must cover bus-level, signal-level, and timing-level faults, controlled via environment variables or test automation.

### Bus-Level: Error Frame Generation

Inject error frames to test ECU error counters and recovery.

**Incorrect (no way to trigger errors on demand):**

```capl
/* No error injection — simulation only sends valid frames */
on timer cyclicTimer
{
    output(msg_EngineStatus);
    setTimer(cyclicTimer, 10);
}
```

**Correct (generate error frames via channel disturbance):**

```capl
variables
{
    int faultActive = 0;
}

on envVar FaultInject_ErrorFrame
{
    faultActive = (int)getValue(this);

    if (faultActive)
    {
        canSetChannelError(1, 1);
        write("CAN error frame injection ACTIVE on channel 1");
    }
    else
    {
        canSetChannelError(1, 0);
        write("CAN error frame injection INACTIVE");
    }
}
```

Use `canSetChannelError` to inject disturbances. Control activation through an environment variable so test automation can toggle faults.

### Bus-Level: Bus-Off Injection

Force a node into bus-off state to validate recovery strategies.

**Incorrect (no bus-off simulation capability):**

```capl
/* ECU simulation never goes offline — misses bus-off recovery testing */
```

**Correct (controlled bus-off via canOffline/canOnline):**

```capl
variables
{
    msTimer busOffRecoveryTimer;
    int busOffDurationMs = 1000;
}

on envVar FaultInject_BusOff
{
    if ((int)getValue(this) == 1)
    {
        canOffline(1);
        write("Channel 1 forced OFFLINE (bus-off injected)");
        setTimer(busOffRecoveryTimer, busOffDurationMs);
    }
    else
    {
        canOnline(1);
        write("Channel 1 forced ONLINE");
    }
}

on timer busOffRecoveryTimer
{
    canOnline(1);
    write("Channel 1 auto-recovered after %d ms", busOffDurationMs);
}
```

Use `canOffline`/`canOnline` to simulate bus-off. Include a timed recovery to test automatic recovery behavior in the DUT.

### Bus-Level: CAN FD Bit Rate Switch Error

Inject faults during BRS phase to test CAN FD error handling.

**Incorrect (only testing classical CAN frames):**

```capl
on timer cyclicTimer
{
    msg_StatusFD.dlc = 8;
    output(msg_StatusFD);
    setTimer(cyclicTimer, 10);
}
```

**Correct (inject BRS mismatch):**

```capl
variables
{
    int brsErrorActive = 0;
}

on envVar FaultInject_BrsError
{
    brsErrorActive = (int)getValue(this);
}

on timer cyclicTimer
{
    if (brsErrorActive)
    {
        msg_StatusFD.brs = 0;
        msg_StatusFD.dlc = 64;
        write("BRS fault: FD frame without bit rate switch");
    }
    else
    {
        msg_StatusFD.brs = 1;
        msg_StatusFD.dlc = 64;
    }
    output(msg_StatusFD);
    setTimer(cyclicTimer, 10);
}
```

Toggle `.brs` to simulate a transmitter failing to switch to the fast data rate. This tests whether the DUT correctly detects format violations.

### Signal-Level: Stuck-At Value Injection

Force a signal to a constant value regardless of the actual source.

**Incorrect (modifying the source model — changes the simulation, not the test):**

```capl
/* Wrong: changing the model itself corrupts the baseline simulation */
void CalcEngineSpeed()
{
    engineSpeed = 0;  /* Hardcoded to test stuck-at, but breaks normal operation */
}
```

**Correct (intercept and override at output):**

```capl
variables
{
    int stuckAtActive = 0;
    double stuckAtValue = 0.0;
}

on envVar FaultInject_StuckSignal
{
    stuckAtActive = (int)getValue(this);
}

on envVar FaultInject_StuckValue
{
    stuckAtValue = getValue(this);
}

on timer cyclicTimer
{
    if (stuckAtActive)
    {
        msg_EngineStatus.EngineSpeed.phys = stuckAtValue;
    }
    else
    {
        msg_EngineStatus.EngineSpeed.phys = simulatedRpm;
    }
    output(msg_EngineStatus);
    setTimer(cyclicTimer, 10);
}
```

Override the signal value at the output stage, keeping the simulation model intact. The stuck value and activation are independently controllable.

### Signal-Level: Out-of-Range Values

Inject values exceeding the signal's defined physical range.

**Incorrect (clamping to valid range — defeats the purpose):**

```capl
on timer cyclicTimer
{
    double temp;
    temp = 300.0;
    if (temp > 200.0) temp = 200.0;  /* Wrong: clamping hides the fault */
    msg_TempStatus.CoolantTemp.phys = temp;
    output(msg_TempStatus);
    setTimer(cyclicTimer, 100);
}
```

**Correct (inject raw value beyond physical range):**

```capl
variables
{
    int oorActive = 0;
    double oorValue = 0.0;
}

on envVar FaultInject_OOR
{
    oorActive = (int)getValue(this);
}

on envVar FaultInject_OORValue
{
    oorValue = getValue(this);
}

on timer cyclicTimer
{
    if (oorActive)
    {
        msg_TempStatus.CoolantTemp.phys = oorValue;
    }
    else
    {
        msg_TempStatus.CoolantTemp.phys = simulatedTemp;
    }
    output(msg_TempStatus);
    setTimer(cyclicTimer, 100);
}
```

Set `.phys` to values outside the database-defined range (e.g., 300°C when max is 200°C) to verify the DUT's range-check logic.

### Signal-Level: Alive Counter Freeze

Stop incrementing the alive counter to trigger E2E protection timeout in the receiver.

**Incorrect (disabling the counter by removing it — changes message layout):**

```capl
/* Wrong: removing alive counter from the frame is not a realistic fault */
```

**Correct (freeze the counter at its current value):**

```capl
variables
{
    int aliveCounterFrozen = 0;
    byte frozenAliveValue = 0;
    byte normalAliveCounter = 0;
}

on envVar FaultInject_AliveFreeze
{
    aliveCounterFrozen = (int)getValue(this);
    if (aliveCounterFrozen)
    {
        frozenAliveValue = normalAliveCounter;
        write("Alive counter frozen at %d", frozenAliveValue);
    }
}

on timer cyclicTimer
{
    if (aliveCounterFrozen)
    {
        msg_EngineStatus.AliveCounter = frozenAliveValue;
    }
    else
    {
        normalAliveCounter = (normalAliveCounter + 1) % 16;
        msg_EngineStatus.AliveCounter = normalAliveCounter;
    }
    output(msg_EngineStatus);
    setTimer(cyclicTimer, 10);
}
```

Freeze the counter at a fixed value to test the DUT's E2E alive counter monitoring. The receiver should detect the stale counter and report a communication error.

### Signal-Level: CRC Corruption

Inject an incorrect CRC to trigger E2E protection failure.

**Incorrect (setting CRC to zero — too obvious, may match valid CRC by chance):**

```capl
msg_EngineStatus.CRC = 0x00;
```

**Correct (invert the correct CRC):**

```capl
variables
{
    int crcCorruptActive = 0;
}

on envVar FaultInject_CrcCorrupt
{
    crcCorruptActive = (int)getValue(this);
}

on preTransmit EngineStatus
{
    if (crcCorruptActive)
    {
        this.CRC = this.CRC ^ 0xFF;
    }
}
```

Use XOR to invert the CRC after it has been calculated. This guarantees an invalid CRC regardless of the payload. Using `on preTransmit` applies the corruption just before transmission.

### Timing-Level: Message Delay Injection

Add a configurable delay to cyclic message transmission.

**Incorrect (changing the cycle timer period — shifts all subsequent messages):**

```capl
on envVar FaultInject_Delay
{
    /* Wrong: permanently alters cycle time instead of injecting a one-shot delay */
    setTimer(cyclicTimer, 10 + (int)getValue(this));
}
```

**Correct (inject one-shot delay, then resume normal cycle):**

```capl
variables
{
    msTimer delayTimer;
    int delayActive = 0;
    int delayMs = 50;
    int normalCycleMs = 10;
}

on envVar FaultInject_Delay
{
    delayActive = (int)getValue(this);
    delayMs = (int)getValue(FaultInject_DelayMs);
}

on timer cyclicTimer
{
    if (delayActive)
    {
        setTimer(delayTimer, delayMs);
        return;
    }
    output(msg_EngineStatus);
    setTimer(cyclicTimer, normalCycleMs);
}

on timer delayTimer
{
    output(msg_EngineStatus);
    setTimer(cyclicTimer, normalCycleMs);
}
```

The delay timer inserts extra latency for the current cycle only, then the normal cycle resumes. This simulates sporadic delays without altering the base period.

### Timing-Level: Jitter Injection

Add random variation to message timing.

**Incorrect (using a fixed pseudo-random seed — produces repeatable, non-random pattern):**

```capl
on timer cyclicTimer
{
    output(msg_SensorData);
    setTimer(cyclicTimer, 10 + 5);  /* Wrong: constant offset, not jitter */
}
```

**Correct (random jitter within bounds):**

```capl
variables
{
    int jitterActive = 0;
    int jitterMaxMs = 5;
    int baseCycleMs = 10;
}

on envVar FaultInject_Jitter
{
    jitterActive = (int)getValue(this);
    jitterMaxMs = (int)getValue(FaultInject_JitterMax);
}

on timer cyclicTimer
{
    int nextInterval;

    output(msg_SensorData);

    if (jitterActive)
    {
        nextInterval = baseCycleMs + (random(2 * jitterMaxMs + 1) - jitterMaxMs);
        if (nextInterval < 1)
            nextInterval = 1;
    }
    else
    {
        nextInterval = baseCycleMs;
    }
    setTimer(cyclicTimer, nextInterval);
}
```

Use `random()` to add bounded jitter around the nominal cycle. Clamp to a minimum of 1 ms to avoid zero or negative intervals.

### Timing-Level: Burst Transmission

Send multiple instances of a message in rapid succession to test receiver overflow.

**Incorrect (loop with output but no timing control):**

```capl
void InjectBurst()
{
    int i;
    for (i = 0; i < 100; i++)
    {
        output(msg_EngineStatus);  /* Wrong: all sent in same time tick */
    }
}
```

**Correct (burst with inter-frame spacing):**

```capl
variables
{
    int burstRemaining = 0;
    int burstCount = 10;
    msTimer burstTimer;
}

on envVar FaultInject_Burst
{
    if ((int)getValue(this) == 1)
    {
        burstRemaining = burstCount;
        setTimer(burstTimer, 1);
        write("Burst injection: %d frames", burstCount);
    }
}

on timer burstTimer
{
    if (burstRemaining <= 0)
        return;

    output(msg_EngineStatus);
    burstRemaining--;

    if (burstRemaining > 0)
    {
        setTimer(burstTimer, 1);
    }
}
```

Use a 1 ms timer to space burst frames. This floods the bus realistically and tests DUT queue/FIFO overflow handling.

### Timing-Level: Missing Cyclic Message (Suppression)

Suppress a cyclic message to trigger DUT timeout detection.

**Incorrect (stopping the timer — cannot resume cleanly):**

```capl
on envVar FaultInject_Suppress
{
    if ((int)getValue(this) == 1)
    {
        cancelTimer(cyclicTimer);  /* Wrong: hard to restart at correct phase */
    }
}
```

**Correct (skip output but keep timer running):**

```capl
variables
{
    int suppressActive = 0;
}

on envVar FaultInject_Suppress
{
    suppressActive = (int)getValue(this);
    if (suppressActive)
        write("Message suppression ACTIVE");
    else
        write("Message suppression INACTIVE");
}

on timer cyclicTimer
{
    if (!suppressActive)
    {
        output(msg_EngineStatus);
    }
    setTimer(cyclicTimer, 10);
}
```

Keep the timer running and skip `output` when suppression is active. This preserves timing phase so the message resumes correctly on deactivation.

### Combining Multiple Faults

Apply several faults simultaneously for complex failure scenarios.

**Incorrect (separate scripts with no coordination):**

```capl
/* Wrong: multiple independent scripts may conflict or miss interactions */
```

**Correct (unified fault controller):**

```capl
variables
{
    int faultStuckAt = 0;
    int faultDelay = 0;
    int faultCrcCorrupt = 0;
    double stuckValue = 0.0;
    int delayMs = 50;
    msTimer delayTimer;
    byte normalAlive = 0;
}

on envVar FaultInject_Profile
{
    int profile;
    profile = (int)getValue(this);

    faultStuckAt = 0;
    faultDelay = 0;
    faultCrcCorrupt = 0;

    switch (profile)
    {
        case 1:
            faultStuckAt = 1;
            stuckValue = 0.0;
            write("Fault profile 1: stuck-at zero");
            break;
        case 2:
            faultDelay = 1;
            delayMs = 100;
            write("Fault profile 2: 100ms delay");
            break;
        case 3:
            faultStuckAt = 1;
            stuckValue = 0.0;
            faultDelay = 1;
            delayMs = 50;
            faultCrcCorrupt = 1;
            write("Fault profile 3: stuck + delay + CRC corrupt");
            break;
    }
}

on timer cyclicTimer
{
    if (faultStuckAt)
        msg_EngineStatus.EngineSpeed.phys = stuckValue;
    else
        msg_EngineStatus.EngineSpeed.phys = simulatedRpm;

    normalAlive = (normalAlive + 1) % 16;
    msg_EngineStatus.AliveCounter = normalAlive;

    if (faultDelay)
    {
        setTimer(delayTimer, delayMs);
        return;
    }

    output(msg_EngineStatus);
    setTimer(cyclicTimer, 10);
}

on preTransmit EngineStatus
{
    if (faultCrcCorrupt)
        this.CRC = this.CRC ^ 0xFF;
}

on timer delayTimer
{
    output(msg_EngineStatus);
    setTimer(cyclicTimer, 10);
}
```

Use fault profiles to activate combinations of faults atomically. A single environment variable selects the profile, avoiding race conditions from toggling multiple variables.

Reference: Vector CANoe Help — Fault Injection, ISO 11898 CAN Error Handling, AUTOSAR E2E Protection Specification
