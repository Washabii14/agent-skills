---
title: Gateway Message Routing
impact: MEDIUM
impactDescription: correct multi-bus message routing
tags: comm, gateway, routing, can-to-ethernet, bus-translation, rate-limiting
---

## Gateway Message Routing

Implement proper message routing in gateway ECUs with signal mapping, timing considerations, and bus-type translation (CAN-to-CAN, CAN-to-Ethernet, Ethernet-to-CAN). Use a routing table for configurability and rate-limiting to prevent bus overload.

**Incorrect (hardcoded routing without rate limiting):**

```c
void Gateway_Forward(uint8_t srcBus, uint32_t msgId,
                     const uint8_t *data, uint8_t len)
{
    if (srcBus == BUS_CAN0)
    {
        CAN1_Transmit(msgId, data, len);  /* No rate limit, no ID translation */
    }
}
```

**Correct (table-driven routing with rate limiting):**

```c
typedef struct
{
    uint32_t srcMsgId;
    uint8_t  srcBus;
    uint32_t dstMsgId;
    uint8_t  dstBus;
    uint8_t  routingMode;  /* DIRECT, SIGNAL_MAPPED, GATEWAY_TRANSLATED */
    uint16_t minIntervalMs;
} GatewayRoute_t;

static const GatewayRoute_t g_routeTable[] =
{
    { 0x100U, BUS_CAN0, 0x200U, BUS_CAN1, ROUTE_DIRECT,        10U },
    { 0x300U, BUS_CAN0, 0x400U, BUS_ETH0, ROUTE_SIGNAL_MAPPED, 20U },
};

void Gateway_RouteMessage(uint8_t srcBus, uint32_t msgId,
                           const uint8_t *data, uint8_t len)
{
    for (uint16_t i = 0U; i < ROUTE_TABLE_SIZE; i++)
    {
        if ((g_routeTable[i].srcBus == srcBus) &&
            (g_routeTable[i].srcMsgId == msgId))
        {
            if (Gateway_IsRateLimited(&g_routeTable[i]))
            {
                continue;
            }
            Gateway_TransmitRouted(&g_routeTable[i], data, len);
        }
    }
}
```

Key gateway routing considerations:
- Support 1:N routing (one source message to multiple destinations)
- Implement rate limiting per route to prevent bus overload
- Handle bus-type translation (e.g., CAN DLC 8 to Ethernet payload with padding)
- Log routing errors as DTCs for diagnostics

Reference: AUTOSAR COM — Gateway signal routing; AUTOSAR PduR — PDU Router
