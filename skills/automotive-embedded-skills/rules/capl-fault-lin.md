---
title: LIN Fault Injection Patterns
impact: MEDIUM
impactDescription: Untested LIN fault scenarios leave slave node error handling and master recovery logic unvalidated
tags: capl, fault-injection, lin, checksum, no-response, timing
---

## LIN Fault Injection Patterns

LIN fault injection validates master and slave node robustness against protocol violations. Faults cover checksum errors, missing responses, header corruption, schedule disruption, and timing violations.

### Checksum Error Injection

Modify the response checksum to test the master's checksum verification.

**Incorrect (sending completely wrong data — not a targeted checksum fault):**

```capl
on linReceiveHeader SensorResponse
{
    int i;
    for (i = 0; i < 8; i++)
        linMsg_SensorResponse.byte(i) = 0xFF;
    output(linMsg_SensorResponse);
}
```

**Correct (send valid data with corrupted checksum):**

```capl
variables
{
    int checksumFaultActive = 0;
}

on envVar FaultInject_LinChecksum
{
    checksumFaultActive = (int)getValue(this);
}

on linReceiveHeader SensorResponse
{
    linMsg_SensorResponse.Temperature.phys = simulatedTemp;
    linMsg_SensorResponse.Pressure.phys = simulatedPressure;

    if (checksumFaultActive)
    {
        linSetChecksumError(linMsg_SensorResponse, 1);
        write("LIN checksum error injected on SensorResponse");
    }
    else
    {
        linSetChecksumError(linMsg_SensorResponse, 0);
    }

    output(linMsg_SensorResponse);
}
```

Use `linSetChecksumError` to corrupt only the checksum while keeping the data payload valid. This isolates the checksum verification test from data content checks.

### No-Response Simulation

Simulate a slave that fails to respond to a header.

**Incorrect (sending an empty frame — still counts as a response):**

```capl
on linReceiveHeader SensorResponse
{
    /* Wrong: outputting a zeroed frame is still a valid response */
    int i;
    for (i = 0; i < 8; i++)
        linMsg_SensorResponse.byte(i) = 0x00;
    output(linMsg_SensorResponse);
}
```

**Correct (suppress response entirely):**

```capl
variables
{
    int noResponseActive = 0;
}

on envVar FaultInject_LinNoResponse
{
    noResponseActive = (int)getValue(this);
}

on linReceiveHeader SensorResponse
{
    if (noResponseActive)
    {
        write("LIN no-response fault: suppressing SensorResponse");
        return;
    }

    linMsg_SensorResponse.Temperature.phys = simulatedTemp;
    output(linMsg_SensorResponse);
}
```

Return without calling `output` so no response data appears on the bus. The master will detect a response timeout and set the appropriate error flag.

### Response Error: Wrong Data Length

Respond with fewer bytes than expected to trigger a framing error.

**Incorrect (padding with zeros to full length):**

```capl
on linReceiveHeader SensorResponse
{
    /* Wrong: this produces a valid 8-byte response */
    linMsg_SensorResponse.dlc = 8;
    output(linMsg_SensorResponse);
}
```

**Correct (send truncated response):**

```capl
variables
{
    int wrongLengthActive = 0;
}

on envVar FaultInject_LinWrongLength
{
    wrongLengthActive = (int)getValue(this);
}

on linReceiveHeader SensorResponse
{
    if (wrongLengthActive)
    {
        linMsg_SensorResponse.dlc = 4;
        write("LIN wrong DLC injected: 4 instead of 8");
    }
    else
    {
        linMsg_SensorResponse.dlc = 8;
    }

    linMsg_SensorResponse.Temperature.phys = simulatedTemp;
    output(linMsg_SensorResponse);
}
```

Set a shorter DLC than the master expects. This tests whether the master detects incomplete responses and handles the framing error.

### Response Error: Wrong NAD

Respond with an incorrect Node Address for diagnostic frames.

**Incorrect (always using the correct NAD):**

```capl
on linReceiveHeader DiagResponse
{
    linMsg_DiagResponse.byte(0) = 0x01;  /* Correct NAD */
    output(linMsg_DiagResponse);
}
```

**Correct (inject wrong NAD on demand):**

```capl
variables
{
    int wrongNadActive = 0;
    byte faultNad = 0x7F;
    byte correctNad = 0x01;
}

on envVar FaultInject_LinWrongNAD
{
    wrongNadActive = (int)getValue(this);
}

on linReceiveHeader DiagResponse
{
    if (wrongNadActive)
    {
        linMsg_DiagResponse.byte(0) = faultNad;
        write("LIN wrong NAD injected: 0x%02X", faultNad);
    }
    else
    {
        linMsg_DiagResponse.byte(0) = correctNad;
    }

    output(linMsg_DiagResponse);
}
```

### Header Error Injection: PID Error

Send a response to the wrong protected identifier to test PID validation.

**Incorrect (modifying the frame ID in the database — permanent change):**

```capl
/* Wrong: changing the DBC/LDF definition is not a runtime fault injection */
```

**Correct (inject PID parity error):**

```capl
variables
{
    int pidErrorActive = 0;
}

on envVar FaultInject_LinPidError
{
    pidErrorActive = (int)getValue(this);
}

on linSendHeader *
{
    if (pidErrorActive)
    {
        linSetParityError(1);
        write("LIN PID parity error injected");
    }
    else
    {
        linSetParityError(0);
    }
}
```

Use `linSetParityError` to corrupt the PID parity bits in the header. Slaves should detect the parity mismatch and discard the header.

### Schedule Table Disruption

Skip slots or alter the schedule to test slave robustness against unexpected timing.

**Incorrect (disabling the entire schedule — too coarse):**

```capl
on envVar FaultInject_LinSchedule
{
    /* Wrong: stopping the whole schedule kills all communication */
    linStopScheduler();
}
```

**Correct (switch to a fault schedule that skips specific slots):**

```capl
variables
{
    int scheduleDisrupted = 0;
}

on envVar FaultInject_LinSchedule
{
    scheduleDisrupted = (int)getValue(this);

    if (scheduleDisrupted)
    {
        linChangeScheduleTable(1);
        write("LIN schedule switched to fault table (slots skipped)");
    }
    else
    {
        linChangeScheduleTable(0);
        write("LIN schedule restored to normal");
    }
}
```

Define a secondary schedule table in the LDF that omits certain frame slots or reorders them. Use `linChangeScheduleTable` to switch at runtime. This is more realistic than stopping the scheduler entirely.

### Timing Violations: Response Too Early / Too Late

Inject timing faults in the slave response window.

**Incorrect (no timing control on response):**

```capl
on linReceiveHeader SensorResponse
{
    output(linMsg_SensorResponse);  /* Timing depends on CAPL engine — not controlled */
}
```

**Correct (delay response beyond the allowed window):**

```capl
variables
{
    msTimer linResponseDelayTimer;
    int responseDelayActive = 0;
    int responseDelayMs = 10;
}

on envVar FaultInject_LinResponseDelay
{
    responseDelayActive = (int)getValue(this);
    responseDelayMs = (int)getValue(FaultInject_LinResponseDelayMs);
}

on linReceiveHeader SensorResponse
{
    if (responseDelayActive)
    {
        linMsg_SensorResponse.Temperature.phys = simulatedTemp;
        setTimer(linResponseDelayTimer, responseDelayMs);
        write("LIN late response: %d ms delay injected", responseDelayMs);
        return;
    }

    linMsg_SensorResponse.Temperature.phys = simulatedTemp;
    output(linMsg_SensorResponse);
}

on timer linResponseDelayTimer
{
    output(linMsg_SensorResponse);
}
```

Delay the `output` call using a timer so the response arrives outside the response slot window. The master should detect a response timeout or late response error. Adjust `responseDelayMs` to test boundary conditions around the allowed window.

Reference: LIN Specification 2.2A — Error Detection and Recovery, Vector CANoe LIN Simulation Guide
