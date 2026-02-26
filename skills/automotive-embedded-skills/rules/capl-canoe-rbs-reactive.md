---
title: Reactive Rest Bus Simulation
impact: HIGH
impactDescription: Missing reactive responses cause NM timeouts, failed diagnostic sequences, and incomplete ECU state simulation
tags: capl, canoe, rbs, simulation, reactive, interaction-layer, nm
---

## Reactive Rest Bus Simulation

Reactive RBS goes beyond cyclic message generation by responding dynamically to received traffic. This includes Network Management (NM) keep-alive, UDS diagnostic responses, and state-dependent behavior. Combining the Interaction Layer (IL) with CAPL logic provides the most realistic simulation.

### NM Keep-Alive Response

**Incorrect (no NM response — DUT detects network loss):**

```capl
/* Simulation sends cyclic messages but never responds to NM requests.
   DUT transitions to Bus-Sleep because no NM echo is received. */
on start
{
    setTimerCyclic(cyclicTimer, 100);
}
```

**Correct (NM message echo with proper timing):**

```capl
variables
{
    message NM_Gateway msg_NmResponse;
    int nmActive = 0;
}

on message NM_DUT
{
    if (this.dir != rx)
        return;

    nmActive = 1;
    cancelTimer(nmTimeoutTimer);
    setTimer(nmTimeoutTimer, 3000);

    msg_NmResponse.NM_NodeID = 0x10;
    msg_NmResponse.NM_RepeatRequest = this.NM_RepeatRequest;
    setTimer(nmResponseDelay, 10);
}

on timer nmResponseDelay
{
    output(msg_NmResponse);
}

on timer nmTimeoutTimer
{
    nmActive = 0;
    write("NM timeout — DUT stopped NM communication");
}
```

Respond to NM messages within the protocol-defined window. Use a short delay timer to avoid responding in the same bus slot. Track NM activity with a timeout to detect when the DUT enters sleep mode.

### Request-Response Patterns (UDS Simulation)

**Incorrect (hardcoded single-service response):**

```capl
on message DiagRequest
{
    msg_DiagResp.byte(0) = 0x62;
    msg_DiagResp.byte(1) = 0xF1;
    msg_DiagResp.byte(2) = 0x90;
    output(msg_DiagResp);
    /* Only handles one DID, ignores service ID, no negative response */
}
```

**Correct (service-aware UDS responder with NRC support):**

```capl
variables
{
    message DiagResponse msg_DiagResp;
    int currentSession = 1;
}

on message DiagRequest
{
    if (this.dir != rx)
        return;

    switch (this.byte(0))
    {
        case 0x10:
            HandleSessionControl(this);
            break;
        case 0x22:
            HandleReadDID(this);
            break;
        case 0x2E:
            HandleWriteDID(this);
            break;
        default:
            SendNegativeResponse(this.byte(0), 0x11);
            break;
    }
}

void HandleSessionControl(message * req)
{
    byte subFunc;

    subFunc = req.byte(1);

    if (subFunc == 0x01 || subFunc == 0x03)
    {
        currentSession = subFunc;
        msg_DiagResp.byte(0) = 0x50;
        msg_DiagResp.byte(1) = subFunc;
        msg_DiagResp.dlc = 6;
        output(msg_DiagResp);
    }
    else
    {
        SendNegativeResponse(0x10, 0x12);
    }
}

void HandleReadDID(message * req)
{
    word did;

    did = (req.byte(1) << 8) | req.byte(2);

    if (did == 0xF190)
    {
        msg_DiagResp.byte(0) = 0x62;
        msg_DiagResp.byte(1) = 0xF1;
        msg_DiagResp.byte(2) = 0x90;
        msg_DiagResp.byte(3) = 0x41;
        msg_DiagResp.dlc = 4;
        output(msg_DiagResp);
    }
    else
    {
        SendNegativeResponse(0x22, 0x31);
    }
}

void SendNegativeResponse(byte serviceId, byte nrc)
{
    msg_DiagResp.byte(0) = 0x7F;
    msg_DiagResp.byte(1) = serviceId;
    msg_DiagResp.byte(2) = nrc;
    msg_DiagResp.dlc = 3;
    output(msg_DiagResp);
}
```

Dispatch on UDS service ID and return proper positive or negative responses. Track session state so service availability changes with the active diagnostic session. Always handle unknown services with NRC `0x11` (serviceNotSupported).

### State-Dependent Responses

**Incorrect (stateless — always responds the same way):**

```capl
on message EngineStartCmd
{
    msg_EngineStatus.Running = 1;
    output(msg_EngineStatus);
    /* Always reports running, regardless of preconditions */
}
```

**Correct (state machine governs response behavior):**

```capl
variables
{
    message EngineStatus msg_EngineStatus;
    int ecuState = 0;
}

const int STATE_OFF       = 0;
const int STATE_ACC       = 1;
const int STATE_CRANKING   = 2;
const int STATE_RUNNING    = 3;

on message IgnitionStatus
{
    if (this.dir != rx)
        return;

    switch (this.IgnitionPos)
    {
        case 0:
            ecuState = STATE_OFF;
            break;
        case 1:
            ecuState = STATE_ACC;
            break;
        case 2:
            TransitionToCranking();
            break;
    }
}

void TransitionToCranking()
{
    if (ecuState == STATE_ACC)
    {
        ecuState = STATE_CRANKING;
        setTimer(crankTimer, 1500);
    }
}

on timer crankTimer
{
    ecuState = STATE_RUNNING;
}

on timer statusCyclicTimer
{
    msg_EngineStatus.EngineState = ecuState;

    switch (ecuState)
    {
        case STATE_OFF:
            msg_EngineStatus.EngineRPM = 0;
            break;
        case STATE_ACC:
            msg_EngineStatus.EngineRPM = 0;
            break;
        case STATE_CRANKING:
            msg_EngineStatus.EngineRPM = 300;
            break;
        case STATE_RUNNING:
            msg_EngineStatus.EngineRPM = 800;
            break;
    }

    output(msg_EngineStatus);
}
```

Use explicit state variables and named constants to govern simulation behavior. Transition between states based on received messages and timers. The cyclic output always reflects the current state, producing realistic signal progressions.

### Interaction Layer (IL) Integration

**Incorrect (IL enabled but signals overwritten by direct byte access):**

```capl
on start
{
    ILEnable();
    /* IL now manages the message, but... */
}

on timer updateTimer
{
    msg_Engine.byte(2) = 0x55;
    output(msg_Engine);
    /* Direct byte write and manual output conflict with IL scheduling */
}
```

**Correct (IL signal control via API):**

```capl
on start
{
    ILEnable();
}

on sysvar sysvar::Simulation::EngineRPM
{
    ILSetSignal(EngineStatus::EngineRPM, @this);
}

on sysvar sysvar::Simulation::EngineTemp
{
    ILSetSignal(EngineStatus::EngineTemp, @this);
}

on key 's'
{
    ILDisable();
    write("IL stopped — all IL-managed messages suspended");
}

on key 'r'
{
    ILEnable();
    write("IL resumed — messages restored");
}
```

When the Interaction Layer is active, always use `ILSetSignal` to update signal values. Never write to message bytes directly or call `output()` on IL-managed messages — the IL handles transmission timing and signal packing internally. Use `ILEnable` / `ILDisable` to start and stop IL message generation cleanly.

### Combining IL and Pure CAPL Message Generation

**Incorrect (IL manages everything — no way to inject custom reactive messages):**

```capl
on start
{
    ILEnable();
    /* All messages are IL-controlled. No mechanism for reactive 
       messages that don't exist in the database. */
}
```

**Correct (IL for cyclic, CAPL for reactive and custom messages):**

```capl
variables
{
    message DiagResponse msg_DiagResp;
    message NM_Gateway msg_NmResp;
}

on start
{
    ILEnable();
    /* IL handles: EngineStatus, TransGear, WheelSpeeds (cyclic) */
    /* CAPL handles: DiagResponse, NM_Gateway (reactive) */
}

on message NM_DUT
{
    if (this.dir != rx)
        return;

    msg_NmResp.NM_NodeID = 0x10;
    setTimer(nmReplyTimer, 5);
}

on timer nmReplyTimer
{
    output(msg_NmResp);
}

on message DiagRequest
{
    if (this.dir != rx)
        return;

    HandleDiagService(this);
}

on sysvar sysvar::Simulation::TargetRPM
{
    ILSetSignal(EngineStatus::EngineRPM, @this);
}

on key 'p'
{
    ILDisable();
    write("IL paused — cyclic messages stopped, reactive handlers still active");
}

on key 'o'
{
    ILEnable();
    write("IL resumed — cyclic messages restarted");
}
```

Use the IL for standard cyclic database messages and pure CAPL for reactive or non-database messages. This separation keeps cyclic timing accurate (managed by the IL engine) while allowing full control over request-response logic. IL pause/resume does not affect CAPL-managed reactive handlers.

Reference: Vector CANoe CAPL Programming Guide — Interaction Layer, Network Management Simulation, UDS Services
