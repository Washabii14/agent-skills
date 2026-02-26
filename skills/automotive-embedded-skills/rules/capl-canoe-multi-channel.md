---
title: Multi-Channel Bus Simulation
impact: HIGH
impactDescription: Incorrect channel routing causes messages to appear on wrong buses, corrupting simulation fidelity and masking integration defects
tags: capl, canoe, simulation, multi-channel, can, lin, ethernet
---

## Multi-Channel Bus Simulation

Automotive ECUs often bridge multiple networks. Simulating them requires explicit channel assignment for every outgoing message and correct handler binding for each bus type. Failing to specify channels causes messages to route to the default bus, producing unrealistic traffic and hiding real integration issues.

### CAN + CAN Dual Channel

**Incorrect (no channel assignment — both messages go to default channel):**

```capl
variables
{
    message PrivateDiag_Req msg_PrivateDiag;
    message PublicVehicleStatus msg_VehicleStatus;
}

on timer sendTimer
{
    output(msg_PrivateDiag);
    output(msg_VehicleStatus);
    setTimer(sendTimer, 20);
}
```

**Correct (explicit channel assignment per bus):**

```capl
variables
{
    message PrivateDiag_Req msg_PrivateDiag;
    message PublicVehicleStatus msg_VehicleStatus;
}

on start
{
    msg_PrivateDiag.MsgChannel = 1;
    msg_VehicleStatus.MsgChannel = 2;
    setTimer(sendTimer, 20);
}

on timer sendTimer
{
    output(msg_PrivateDiag);
    output(msg_VehicleStatus);
    setTimer(sendTimer, 20);
}
```

Set `MsgChannel` once at startup or use `output()` with a channel parameter. Never rely on default channel routing when simulating multi-bus nodes.

### CAN + LIN (Body Controller with LIN Sub-Bus)

**Incorrect (using `on message` for LIN frames):**

```capl
on message LIN_WindowPos
{
    /* LIN frames require on linMessage, not on message */
}
```

**Correct (LIN handler and CAN-to-LIN gateway logic):**

```capl
variables
{
    message VehicleBodyStatus msg_BodyStatus;
    linMessage LIN_WindowCmd msg_LinWindowCmd;
    int windowPosition = 0;
}

on message WindowControl
{
    if (this.dir != rx)
        return;

    msg_LinWindowCmd.WindowTarget = this.TargetPos;
    output(msg_LinWindowCmd);
}

on linMessage LIN_WindowPos
{
    if (this.dir != rx)
        return;

    windowPosition = this.CurrentPos;
    msg_BodyStatus.WindowPos = windowPosition;
    output(msg_BodyStatus);
}
```

Use `on linMessage` for LIN frame reception and `linMessage` type for LIN transmit variables. Gateway nodes translate between CAN and LIN using the correct handler and message types for each bus.

### CAN + Ethernet (Gateway Testing)

**Incorrect (no Ethernet handler — gateway simulation ignores IP traffic):**

```capl
on message CAN_GatewayStatus
{
    if (this.dir != rx)
        return;

    /* Forwards to Ethernet but never receives Ethernet responses */
}
```

**Correct (bidirectional CAN-Ethernet gateway simulation):**

```capl
variables
{
    message CAN_DiagResponse msg_CanDiagResp;
}

on ethernetPacket *
{
    if (this.dir != rx)
        return;

    if (EthGetProtocol(this) == 0x0800)
    {
        HandleIpPacket(this);
    }
}

void HandleIpPacket(ethernetPacket * pkt)
{
    byte payload[8];

    EthGetPayload(pkt, payload, elCount(payload));
    msg_CanDiagResp.DiagData = payload[0];
    msg_CanDiagResp.MsgChannel = 1;
    output(msg_CanDiagResp);
}

on message CAN_DiagRequest
{
    if (this.dir != rx)
        return;

    SendEthernetDiagRequest(this);
}
```

Use `on ethernetPacket` for Ethernet reception. Filter by EtherType to distinguish protocol layers. Assign CAN channel explicitly when forwarding from Ethernet to CAN.

### Full Vehicle Simulation (CAN1 + CAN2 + LIN + ETH Coordination)

**Incorrect (flat structure with no synchronization):**

```capl
on start
{
    setTimer(canTimer, 10);
    setTimer(linTimer, 10);
    /* All buses start simultaneously with identical timing — unrealistic bus burst */
}
```

**Correct (staggered startup with cross-bus synchronization):**

```capl
variables
{
    message PT_EngineStatus msg_PtEngine;
    message CH_BrakeStatus msg_ChBrake;
    linMessage LIN_SeatCmd msg_LinSeat;
    int vehicleState = 0;
    int busesReady = 0;
}

on start
{
    msg_PtEngine.MsgChannel = 1;
    msg_ChBrake.MsgChannel = 2;

    setTimer(can1StartTimer, 5);
    setTimer(can2StartTimer, 15);
    setTimer(linStartTimer, 30);
}

on timer can1StartTimer
{
    busesReady |= 0x01;
    setTimerCyclic(can1CyclicTimer, 10);
}

on timer can2StartTimer
{
    busesReady |= 0x02;
    setTimerCyclic(can2CyclicTimer, 20);
}

on timer linStartTimer
{
    busesReady |= 0x04;
    setTimerCyclic(linCyclicTimer, 50);
}

on timer can1CyclicTimer
{
    msg_PtEngine.EngineRPM = GetSimulatedRPM();
    output(msg_PtEngine);
}

on timer can2CyclicTimer
{
    msg_ChBrake.BrakePressure = GetSimulatedBrake();
    output(msg_ChBrake);
}

on timer linCyclicTimer
{
    msg_LinSeat.Position = GetSeatTarget();
    output(msg_LinSeat);
}

on ethernetPacket *
{
    if (this.dir != rx)
        return;

    if (busesReady == 0x07)
    {
        RouteEthernetToVehicle(this);
    }
}
```

Stagger bus startup timers to avoid simultaneous bus load spikes. Track bus readiness with a bitmask before enabling cross-bus routing. Assign `MsgChannel` explicitly per message to prevent channel misrouting in complex topologies.

### Channel Assignment Methods

**Incorrect (hardcoded channel in output without clarity):**

```capl
on timer sendTimer
{
    output(msg_Status);
    /* Which channel does this go to? Depends on hidden default. */
}
```

**Correct (explicit assignment with named constants):**

```capl
variables
{
    const int CH_POWERTRAIN = 1;
    const int CH_CHASSIS = 2;
    message EngineStatus msg_Engine;
    message BrakeStatus msg_Brake;
}

on start
{
    msg_Engine.MsgChannel = CH_POWERTRAIN;
    msg_Brake.MsgChannel = CH_CHASSIS;
}
```

Use named constants for channel numbers. This makes channel assignments self-documenting and simplifies reconfiguration when bus topology changes.

Reference: Vector CANoe CAPL Programming Guide — Multi-Channel Configuration, LIN/Ethernet Node Programming
