---
title: Gateway Simulation and Routing Patterns
impact: HIGH
impactDescription: Incorrect gateway routing causes signal loss, wrong byte order, or broken cross-protocol communication in virtual networks
tags: capl, canoe, gateway, routing, signal-mapping, cross-protocol
---

## Gateway Simulation and Routing Patterns

Gateway ECU simulation requires correct signal mapping, byte order handling, and cross-protocol translation. Errors in routing logic silently corrupt data or drop messages, making integration test results unreliable.

### Signal-Based Routing

Route individual signals between messages, applying scaling and offset conversions when source and destination databases differ.

**Incorrect (raw byte copy ignores scaling differences):**

```capl
on message EngineStatus_CAN1
{
    if (this.dir != rx)
        return;

    /* Wrong: raw byte copy doesn't account for different scaling/offset */
    msg_EngineRPM_CAN2.byte(0) = this.byte(0);
    msg_EngineRPM_CAN2.byte(1) = this.byte(1);
    output(msg_EngineRPM_CAN2);
}
```

**Correct (extract physical value, apply destination scaling):**

```capl
on message EngineStatus_CAN1
{
    if (this.dir != rx)
        return;

    double physRpm;
    physRpm = this.EngineSpeed.phys;

    msg_EngineRPM_CAN2.EngineSpeedRouted.phys = physRpm;
    output(msg_EngineRPM_CAN2);
}
```

Use `.phys` to read/write in physical units so the database scaling/offset is applied automatically. Never copy raw bytes between messages that have different signal definitions.

### PDU-Based Routing (Whole Message Forwarding)

Forward an entire message from one CAN channel to another.

**Incorrect (output to same channel, missing channel assignment):**

```capl
on message EngineStatus
{
    if (this.dir != rx)
        return;

    /* Wrong: no channel assignment — sends on default channel */
    output(this);
}
```

**Correct (assign target channel before forwarding):**

```capl
on message CAN1::EngineStatus
{
    if (this.dir != rx)
        return;

    message EngineStatus fwdMsg;
    fwdMsg = this;
    fwdMsg.msgChannel = 2;
    output(fwdMsg);
}
```

Create a local copy, set `.msgChannel` to the destination, then output. Using qualified names (`CAN1::`) prevents handler recursion.

### Cross-Protocol Routing: CAN → Ethernet (SOME/IP)

Pack CAN signal values into a SOME/IP service payload for Ethernet transmission.

**Incorrect (sending raw CAN frame bytes as Ethernet payload):**

```capl
on message VehicleSpeed_CAN
{
    if (this.dir != rx)
        return;

    /* Wrong: dumping raw CAN bytes into IP payload with no structure */
    long pktHandle;
    pktHandle = EthGetPacket(64);
    EthSetTokenData(pktHandle, 0, 8, this.byte(0));
    EthOutputPacket(pktHandle);
}
```

**Correct (pack signals into structured SOME/IP payload):**

```capl
on message VehicleSpeed_CAN
{
    if (this.dir != rx)
        return;

    long pktHandle;
    byte payload[12];
    word speedRaw;

    speedRaw = (word)(this.VehicleSpeed.phys / 0.01);

    payload[0] = (speedRaw >> 8) & 0xFF;
    payload[1] = speedRaw & 0xFF;

    pktHandle = SomeIpCreateMessage(0x1234, 0x0001, 0x01);
    SomeIpSetPayload(pktHandle, payload, elCount(payload));
    SomeIpSend(pktHandle);
}
```

Extract the physical value from CAN, encode it according to the SOME/IP service interface specification (byte order, width), and send via the SOME/IP API.

### Cross-Protocol Routing: Ethernet → CAN

Unpack SOME/IP payload data into CAN message signals.

**Incorrect (no byte order awareness when unpacking):**

```capl
on SomeIpMessage 0x1234.*
{
    byte payload[64];
    SomeIpGetPayload(this, payload, 64);

    /* Wrong: assumes host byte order matches CAN signal layout */
    msg_TargetSpeed_CAN.TargetSpeed.raw = payload[0];
    output(msg_TargetSpeed_CAN);
}
```

**Correct (explicit byte order unpacking):**

```capl
on SomeIpMessage 0x1234.*
{
    byte payload[64];
    int len;
    word speedRaw;

    len = SomeIpGetPayload(this, payload, elCount(payload));
    if (len < 2)
        return;

    speedRaw = (payload[0] << 8) | payload[1];
    msg_TargetSpeed_CAN.TargetSpeed.phys = speedRaw * 0.01;
    output(msg_TargetSpeed_CAN);
}
```

### Cross-Protocol Routing: CAN → LIN

Route a CAN signal to a LIN response.

**Incorrect (writing LIN response at arbitrary time):**

```capl
on message BrakeLight_CAN
{
    if (this.dir != rx)
        return;

    /* Wrong: output at CAN message rate, not synced to LIN schedule */
    linMsg_BrakeLight.BrakeLightCmd = this.BrakeLightReq.phys;
    output(linMsg_BrakeLight);
}
```

**Correct (update shadow value, send on LIN schedule slot):**

```capl
variables
{
    int brakeLightValue = 0;
}

on message BrakeLight_CAN
{
    if (this.dir != rx)
        return;

    brakeLightValue = (int)this.BrakeLightReq.phys;
}

on linReceiveHeader BrakeLight
{
    linMsg_BrakeLight.BrakeLightCmd = brakeLightValue;
    output(linMsg_BrakeLight);
}
```

Store the CAN value and only output the LIN response when the LIN master sends the header. This matches real LIN slave behavior.

### Byte Order Conversion

Convert between big-endian (Motorola) and little-endian (Intel) during routing.

**Incorrect (no byte swap when routing between different-endian buses):**

```capl
void RouteSpeed(message * src, message * dst)
{
    /* Wrong: source is Motorola byte order, destination is Intel */
    dst.byte(0) = src.byte(0);
    dst.byte(1) = src.byte(1);
}
```

**Correct (explicit byte swap):**

```capl
word SwapBytes16(word val)
{
    return ((val & 0xFF) << 8) | ((val >> 8) & 0xFF);
}

void RouteSpeedWithByteSwap(double physValue)
{
    msg_SpeedDst.SpeedSignal.phys = physValue;
    output(msg_SpeedDst);
}

on message SpeedSrc
{
    if (this.dir != rx)
        return;

    RouteSpeedWithByteSwap(this.SpeedSignal.phys);
}
```

Prefer routing via `.phys` values so the database handles byte order automatically. If you must copy raw bytes, swap explicitly.

### Signal Multiplexing/Demultiplexing

Split signals from one source message into multiple destination messages, or combine multiple sources into one.

**Incorrect (all signals forced into one destination, ignoring mux index):**

```capl
on message MultiplexedStatus
{
    if (this.dir != rx)
        return;

    /* Wrong: always overwrites all targets regardless of mux value */
    msg_Dst1.Temp.phys = this.Temperature.phys;
    msg_Dst2.Press.phys = this.Pressure.phys;
    output(msg_Dst1);
    output(msg_Dst2);
}
```

**Correct (route based on multiplexer value):**

```capl
on message MultiplexedStatus
{
    if (this.dir != rx)
        return;

    int muxIdx;
    muxIdx = (int)this.MuxSelector.phys;

    switch (muxIdx)
    {
        case 0:
            msg_Dst1.Temp.phys = this.Temperature.phys;
            output(msg_Dst1);
            break;
        case 1:
            msg_Dst2.Press.phys = this.Pressure.phys;
            output(msg_Dst2);
            break;
        case 2:
            msg_Dst3.Voltage.phys = this.BattVoltage.phys;
            output(msg_Dst3);
            break;
    }
}
```

Check the multiplexer selector to determine which signal group is valid in the current frame, then route only the relevant signals.

### Conditional Routing (NM / Mode Dependent)

Route messages only when network management is active or other preconditions hold.

**Incorrect (routes unconditionally):**

```capl
on message EngineData
{
    if (this.dir != rx)
        return;

    msg_FwdEngineData.msgChannel = 2;
    msg_FwdEngineData = this;
    output(msg_FwdEngineData);
}
```

**Correct (check NM state before routing):**

```capl
variables
{
    int nmActive = 0;
}

on envVar NmState
{
    nmActive = (int)getValue(this);
}

on message EngineData
{
    if (this.dir != rx)
        return;

    if (!nmActive)
        return;

    message EngineData fwdMsg;
    fwdMsg = this;
    fwdMsg.msgChannel = 2;
    output(fwdMsg);
}
```

Guard routing with NM state, ignition status, or diagnostic session checks. This prevents ghost messages on sleeping buses.

### Routing Delay and Timing

Real gateways introduce latency. Simulate this to validate timeout monitors in downstream ECUs.

**Incorrect (instantaneous forwarding — unrealistic, hides timing issues):**

```capl
on message SensorData
{
    if (this.dir != rx)
        return;

    output(msg_FwdSensorData);
}
```

**Correct (configurable routing delay):**

```capl
variables
{
    msTimer routingDelayTimer;
    int routingDelayMs = 5;
    message SensorData pendingMsg;
    int hasPending = 0;
}

on message SensorData
{
    if (this.dir != rx)
        return;

    pendingMsg = this;
    hasPending = 1;
    setTimer(routingDelayTimer, routingDelayMs);
}

on timer routingDelayTimer
{
    if (!hasPending)
        return;

    message FwdSensorData fwdMsg;
    fwdMsg.SensorValue.phys = pendingMsg.SensorValue.phys;
    fwdMsg.msgChannel = 2;
    output(fwdMsg);
    hasPending = 0;
}
```

Use a timer to simulate gateway processing delay. Make the delay configurable via environment variable for parametric testing.

Reference: Vector CANoe Application Notes — Gateway Simulation, AUTOSAR COM Routing Specification
