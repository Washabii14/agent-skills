---
title: ICMP Handling for Network Diagnostics
impact: MEDIUM
impactDescription: network reachability and diagnostics
tags: comm, icmp, ethernet, ping, reachability, network-diagnostics
---

## ICMP Handling for Network Diagnostics

Use ICMP Echo (ping) for ECU reachability monitoring and Destination Unreachable for connection diagnostics. Track consecutive failures to detect persistent network issues.

**Incorrect (no reachability monitoring):**

```c
void CheckPeer(uint32_t peerIp)
{
    /* No monitoring — assume peer is always reachable */
}
```

**Correct (ICMP-based reachability monitoring):**

```c
typedef struct
{
    uint32_t targetIp;
    uint16_t sequenceNum;
    uint32_t lastRoundtripUs;
    uint32_t timeoutMs;
    boolean  isReachable;
    uint8_t  failCount;
    uint8_t  maxFailCount;
} IcmpMonitor_t;

Std_ReturnType Icmp_SendEchoRequest(IcmpMonitor_t *mon)
{
    IcmpEchoPacket_t pkt;
    pkt.type       = ICMP_ECHO_REQUEST;
    pkt.code       = 0U;
    pkt.identifier = ICMP_OWN_ID;
    pkt.sequence   = mon->sequenceNum++;
    pkt.checksum   = 0U;
    pkt.checksum   = Icmp_CalculateChecksum(&pkt, sizeof(pkt));

    return Eth_Transmit(mon->targetIp, IPPROTO_ICMP,
                        (const uint8_t *)&pkt, sizeof(pkt));
}

void Icmp_HandleEchoReply(IcmpMonitor_t *mon, uint32_t roundtripUs)
{
    mon->lastRoundtripUs = roundtripUs;
    mon->isReachable = TRUE;
    mon->failCount = 0U;
}

void Icmp_HandleTimeout(IcmpMonitor_t *mon)
{
    mon->failCount++;
    if (mon->failCount >= mon->maxFailCount)
    {
        mon->isReachable = FALSE;
        ReportDtc(DTC_ETH_PEER_UNREACHABLE);
    }
}
```

Automotive use cases:
- Gateway ECU monitoring connectivity to all Ethernet peers
- Pre-communication reachability check before DoIP sessions
- Network startup sequencing — wait for peer ECUs before sending data

Reference: AUTOSAR SWS TCP/IP Stack — ICMP handling
