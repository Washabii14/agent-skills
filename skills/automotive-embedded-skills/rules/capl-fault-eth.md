---
title: Ethernet Fault Injection Patterns
impact: MEDIUM
impactDescription: Untested Ethernet faults leave ECU network stack error handling and failover logic unvalidated
tags: capl, fault-injection, ethernet, packet-loss, latency, link-down
---

## Ethernet Fault Injection Patterns

Ethernet fault injection in vehicle networks tests ECU resilience against link failures, packet loss, latency spikes, reordering, payload corruption, and VLAN misconfigurations. Control all faults via environment variables for test automation integration.

### Link Down Simulation

Disable an Ethernet port to test link-loss detection and failover.

**Incorrect (stopping all Ethernet communication by removing the node — too destructive):**

```capl
/* Wrong: removing the node from the simulation config is not a runtime fault */
```

**Correct (toggle link state at runtime):**

```capl
variables
{
    int linkDownActive = 0;
}

on envVar FaultInject_EthLinkDown
{
    linkDownActive = (int)getValue(this);

    if (linkDownActive)
    {
        ethSetLinkState(1, 0);
        write("Ethernet port 1 link DOWN");
    }
    else
    {
        ethSetLinkState(1, 1);
        write("Ethernet port 1 link UP");
    }
}
```

Use `ethSetLinkState` to bring a port down without removing the node. The DUT's TCP/IP stack should detect the link loss and trigger reconnection or failover logic.

### Packet Loss Injection

Selectively drop received packets at a configurable rate.

**Incorrect (dropping all packets — not a realistic fault):**

```capl
on ethernetPacket *
{
    /* Wrong: 100% drop rate destroys all communication */
    return;
}
```

**Correct (probabilistic packet drop):**

```capl
variables
{
    int packetLossActive = 0;
    int lossRatePercent = 10;
    long dropCount = 0;
    long totalCount = 0;
}

on envVar FaultInject_EthPacketLoss
{
    packetLossActive = (int)getValue(this);
    lossRatePercent = (int)getValue(FaultInject_EthLossRate);
    dropCount = 0;
    totalCount = 0;
}

on ethernetPacket *
{
    if (!packetLossActive)
        return;

    totalCount++;

    if (random(100) < lossRatePercent)
    {
        dropCount++;
        write("ETH packet dropped (%d/%d)", dropCount, totalCount);
        /* Consume the packet by not forwarding */
        return;
    }
}
```

Use `random()` to implement a probabilistic drop rate. Log drop statistics so test reports can verify the actual loss rate. In a gateway simulation, the `return` prevents forwarding; for a pass-through node, omit the `output` call.

### Latency Injection

Delay packet forwarding by a configurable time to test timeout handling.

**Incorrect (blocking in the handler — CAPL is single-threaded, this blocks everything):**

```capl
on ethernetPacket *
{
    /* Wrong: CAPL has no sleep(); this approach doesn't work */
    /* delay(50); */
    output(this);
}
```

**Correct (queue packet, forward after timer):**

```capl
variables
{
    int latencyActive = 0;
    int latencyMs = 50;
    long pendingPktHandle = 0;
    msTimer latencyTimer;
}

on envVar FaultInject_EthLatency
{
    latencyActive = (int)getValue(this);
    latencyMs = (int)getValue(FaultInject_EthLatencyMs);
}

on ethernetPacket *
{
    if (!latencyActive)
        return;

    pendingPktHandle = EthCopyPacket(this);
    setTimer(latencyTimer, latencyMs);
}

on timer latencyTimer
{
    if (pendingPktHandle != 0)
    {
        EthOutputPacket(pendingPktHandle);
        EthReleasePacket(pendingPktHandle);
        pendingPktHandle = 0;
    }
}
```

Copy the packet handle, hold it, and forward after the delay timer expires. Release the handle after output to avoid memory leaks. For multiple concurrent packets, use an array or ring buffer of handles.

### Packet Reordering

Swap the delivery order of consecutive packets.

**Incorrect (random delay on every packet — unpredictable, not controlled reordering):**

```capl
on ethernetPacket *
{
    /* Wrong: random delay doesn't guarantee reordering */
    setTimer(reorderTimer, random(10));
}
```

**Correct (hold-and-swap using a two-slot buffer):**

```capl
variables
{
    int reorderActive = 0;
    long pktSlot1 = 0;
    long pktSlot2 = 0;
    int slotFilled = 0;
    msTimer reorderFlushTimer;
}

on envVar FaultInject_EthReorder
{
    reorderActive = (int)getValue(this);
    slotFilled = 0;
}

on ethernetPacket *
{
    if (!reorderActive)
        return;

    if (slotFilled == 0)
    {
        pktSlot1 = EthCopyPacket(this);
        slotFilled = 1;
        setTimer(reorderFlushTimer, 5);
    }
    else
    {
        pktSlot2 = EthCopyPacket(this);

        EthOutputPacket(pktSlot2);
        EthReleasePacket(pktSlot2);

        EthOutputPacket(pktSlot1);
        EthReleasePacket(pktSlot1);

        slotFilled = 0;
        cancelTimer(reorderFlushTimer);
        pktSlot1 = 0;
        pktSlot2 = 0;
    }
}

on timer reorderFlushTimer
{
    if (slotFilled == 1 && pktSlot1 != 0)
    {
        EthOutputPacket(pktSlot1);
        EthReleasePacket(pktSlot1);
        pktSlot1 = 0;
        slotFilled = 0;
    }
}
```

Buffer the first packet and wait for the second. Output the second before the first to guarantee reordering. A flush timer prevents the first packet from being held indefinitely if no second packet arrives.

### Corrupt Payload

Flip bits in the packet payload to test data integrity checks.

**Incorrect (zeroing the entire payload — too obvious, doesn't test bit-level detection):**

```capl
on ethernetPacket *
{
    int i;
    byte data[1500];
    int len;

    len = EthGetTokenData(this, 0, 1500, data);
    for (i = 0; i < len; i++)
        data[i] = 0x00;  /* Wrong: complete zeroing is not a realistic bit error */
    EthSetTokenData(this, 0, len, data);
}
```

**Correct (flip specific bits at configurable offset):**

```capl
variables
{
    int corruptActive = 0;
    int corruptOffset = 20;
    byte corruptMask = 0x01;
}

on envVar FaultInject_EthCorrupt
{
    corruptActive = (int)getValue(this);
    corruptOffset = (int)getValue(FaultInject_EthCorruptOffset);
}

on ethernetPacket *
{
    byte payload[1500];
    int len;

    if (!corruptActive)
        return;

    len = EthGetTokenData(this, 0, elCount(payload), payload);

    if (corruptOffset < len)
    {
        payload[corruptOffset] = payload[corruptOffset] ^ corruptMask;
        EthSetTokenData(this, 0, len, payload);
        write("ETH payload corrupted at offset %d", corruptOffset);
    }
}
```

XOR a single byte at a configurable offset to inject a bit flip. This tests whether the application-layer CRC or integrity check detects the corruption. The offset and mask are configurable via environment variables.

### VLAN Stripping / Mistagging

Remove or modify the VLAN tag to test VLAN-aware switching and ECU VLAN filtering.

**Incorrect (hard-coding a VLAN ID without ability to revert):**

```capl
on ethernetPacket *
{
    /* Wrong: permanently modifies VLAN with no control */
    EthSetVlanId(this, 99);
}
```

**Correct (configurable VLAN fault modes):**

```capl
variables
{
    int vlanFaultMode = 0;
    int faultVlanId = 99;
}

on envVar FaultInject_EthVlanMode
{
    vlanFaultMode = (int)getValue(this);
    faultVlanId = (int)getValue(FaultInject_EthVlanId);
}

on ethernetPacket *
{
    if (vlanFaultMode == 0)
        return;

    switch (vlanFaultMode)
    {
        case 1:
            EthRemoveVlanTag(this);
            write("ETH VLAN tag stripped");
            break;
        case 2:
            EthSetVlanId(this, faultVlanId);
            write("ETH VLAN mistagged to %d", faultVlanId);
            break;
        case 3:
            EthSetVlanPriority(this, 0);
            write("ETH VLAN priority forced to 0");
            break;
    }
}
```

Support multiple VLAN fault modes: strip the tag entirely (mode 1), change the VLAN ID to a wrong value (mode 2), or downgrade the priority (mode 3). The DUT's VLAN filter should reject or misroute the affected frames.

Reference: Vector CANoe Ethernet Simulation Guide, IEEE 802.3 Error Handling, IEEE 802.1Q VLAN Tagging
