---
title: Cyclic Rest Bus Simulation
impact: MEDIUM
impactDescription: Missing or incorrectly timed cyclic messages cause NM timeouts, DTC storage, and unreliable integration testing
tags: capl, canoe, rbs, simulation, cyclic, rest-bus
---

## Cyclic Rest Bus Simulation

Rest Bus Simulation (RBS) generates the background traffic that absent ECUs would normally produce. Cyclic messages must match database-defined timing, include correct alive counters and checksums, and start in a staggered sequence to avoid bus load spikes at simulation startup.

### Generating Cyclic Messages from Database Definitions

**Incorrect (hardcoded message IDs and manual signal packing):**

```capl
variables
{
    message 0x123 msg_Engine;
}

on timer engineTimer
{
    msg_Engine.byte(0) = 0x50;
    msg_Engine.byte(1) = 0x00;
    output(msg_Engine);
    setTimer(engineTimer, 10);
}
```

**Correct (database-bound message with symbolic signal access):**

```capl
variables
{
    message EngineStatus msg_Engine;
}

on start
{
    msg_Engine.EngineRPM = 800;
    msg_Engine.EngineTemp = 90;
    msg_Engine.EngineState = 1;
    setTimerCyclic(engineTimer, 10);
}

on timer engineTimer
{
    output(msg_Engine);
}
```

Declare messages using database symbolic names, not raw IDs. Access signals by name so the DBC layout handles byte ordering and scaling automatically.

### Configuring Cycle Times

**Incorrect (one-shot timer re-armed manually with drift risk):**

```capl
on timer sendTimer
{
    output(msg_Status);
    setTimer(sendTimer, 100);
    /* Timer drift accumulates — each cycle is 100ms + handler execution time */
}
```

**Correct (cyclic timer for stable periodicity):**

```capl
variables
{
    message EngineStatus msg_Engine;
    message TransmissionGear msg_Trans;
}

on start
{
    setTimerCyclic(engineCyclicTimer, 10);
    setTimerCyclic(transCyclicTimer, 20);
}

on timer engineCyclicTimer
{
    output(msg_Engine);
}

on timer transCyclicTimer
{
    output(msg_Trans);
}
```

Use `setTimerCyclic` for periodic messages. Unlike manually re-armed timers, cyclic timers maintain consistent intervals independent of handler execution time.

### Counter and CRC Signal Generation

**Incorrect (static counter and no checksum — receiver detects stale data):**

```capl
on timer engineTimer
{
    msg_Engine.EngineRPM = 800;
    output(msg_Engine);
    /* AliveCounter never increments, CRC never computed */
}
```

**Correct (rolling alive counter with CRC computation):**

```capl
variables
{
    message EngineStatus msg_Engine;
    byte aliveCounter_Engine = 0;
}

byte ComputeCRC(message * msg, int length)
{
    byte crc = 0xFF;
    int i;

    for (i = 1; i < length; i++)
    {
        crc ^= msg.byte(i);
    }
    return crc;
}

on timer engineCyclicTimer
{
    msg_Engine.AliveCounter = aliveCounter_Engine;
    aliveCounter_Engine = (aliveCounter_Engine + 1) % 16;

    msg_Engine.CRC = ComputeCRC(msg_Engine, msg_Engine.dlc);

    output(msg_Engine);
}
```

Increment the alive counter before each transmission and wrap at the defined modulus (commonly 4-bit = mod 16). Compute the CRC over the payload after setting all other signals, then place it in the CRC field. The receiver's E2E protection will reject messages with stale counters or incorrect checksums.

### Enabling and Disabling Individual Messages at Runtime

**Incorrect (global flag stops all messages):**

```capl
variables
{
    int simulationActive = 1;
}

on timer engineTimer
{
    if (simulationActive)
        output(msg_Engine);
}

on timer brakeTimer
{
    if (simulationActive)
        output(msg_Brake);
}
```

**Correct (per-message enable flags for selective control):**

```capl
variables
{
    message EngineStatus msg_Engine;
    message BrakeStatus msg_Brake;
    message SteeringAngle msg_Steering;
    int enableEngine = 1;
    int enableBrake = 1;
    int enableSteering = 1;
}

on timer engineCyclicTimer
{
    if (enableEngine)
        output(msg_Engine);
}

on timer brakeCyclicTimer
{
    if (enableBrake)
        output(msg_Brake);
}

on timer steeringCyclicTimer
{
    if (enableSteering)
        output(msg_Steering);
}

on sysvar sysvar::RBS::EnableEngine
{
    enableEngine = @this;
}

on sysvar sysvar::RBS::EnableBrake
{
    enableBrake = @this;
}
```

Use per-message enable flags controlled by system variables. This allows testers to selectively suppress individual messages from the CANoe panel without stopping the entire simulation — essential for testing missing-message DTCs.

### Startup Sequencing (Staggered Starts)

**Incorrect (all messages start at t=0 — bus burst at startup):**

```capl
on start
{
    setTimerCyclic(timer10ms_A, 10);
    setTimerCyclic(timer10ms_B, 10);
    setTimerCyclic(timer10ms_C, 10);
    setTimerCyclic(timer20ms_A, 20);
    setTimerCyclic(timer20ms_B, 20);
    /* All five messages fire at t=0, t=10, t=20... causing periodic load spikes */
}
```

**Correct (staggered initial offsets distribute bus load):**

```capl
variables
{
    message EngineStatus msg_Engine;
    message TransGear msg_Trans;
    message WheelSpeed_FL msg_WhlFL;
    message BrakeStatus msg_Brake;
    message SteeringAngle msg_Steer;
}

on start
{
    msg_Engine.EngineRPM = 800;
    msg_Trans.CurrentGear = 0;

    setTimer(startEngine, 0);
    setTimer(startTrans, 2);
    setTimer(startWhlFL, 4);
    setTimer(startBrake, 6);
    setTimer(startSteer, 8);
}

on timer startEngine   { setTimerCyclic(timerEngine, 10); }
on timer startTrans    { setTimerCyclic(timerTrans, 10); }
on timer startWhlFL    { setTimerCyclic(timerWhlFL, 10); }
on timer startBrake    { setTimerCyclic(timerBrake, 20); }
on timer startSteer    { setTimerCyclic(timerSteer, 20); }

on timer timerEngine { output(msg_Engine); }
on timer timerTrans  { output(msg_Trans); }
on timer timerWhlFL  { output(msg_WhlFL); }
on timer timerBrake  { output(msg_Brake); }
on timer timerSteer  { output(msg_Steer); }
```

Stagger cyclic timer starts by a few milliseconds using one-shot setup timers. This distributes the initial bus load evenly and prevents periodic collision of same-rate messages. Offset values should be smaller than the shortest cycle time.

Reference: Vector CANoe CAPL Programming Guide — Rest Bus Simulation, Timer Functions
