---
title: FIBEX Network Descriptions
impact: MEDIUM
impactDescription: Maintains automotive Ethernet network topology descriptions
tags: integration, fibex, ethernet, network, topology, service-endpoint
---

## FIBEX Network Descriptions

Maintain FIBEX files for automotive Ethernet network topology and service endpoint descriptions. FIBEX (Field Bus Exchange) describes the network architecture including ECU endpoints, IP addresses, SOME/IP services, and PDU routing for Ethernet-based communication.

**Incorrect (network topology hardcoded in source):**

```c
#define GATEWAY_IP    "192.168.1.1"
#define ADAS_ECU_IP   "192.168.1.10"
#define DIAG_PORT     13400U
```

**Correct (FIBEX-aligned network configuration):**

```c
typedef struct
{
    const char  *ecuName;
    uint32_t     ipAddr;
    uint16_t     vlanId;
    uint16_t     someIpPort;
    const char  *fibexRef;
} EthNetworkEndpoint_t;

static const EthNetworkEndpoint_t g_networkConfig[] =
{
    { "GatewayECU", 0xC0A80101U, 10U, 30490U, "FIBEX_EP_Gateway" },
    { "ADAS_Front", 0xC0A8010AU, 20U, 30491U, "FIBEX_EP_ADAS_F"  },
    { "BodyCtrl",   0xC0A80114U, 30U, 30492U, "FIBEX_EP_BCM"     },
};
```

FIBEX references should be used in source as traceability links. When the FIBEX model changes (new ECU, changed IP), source configuration must be regenerated to stay in sync.

Reference: ASAM FIBEX — Field Bus Exchange Format
