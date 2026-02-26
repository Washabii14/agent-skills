---
title: VLAN and QoS Priority Mapping
impact: MEDIUM
impactDescription: traffic isolation and priority per IEEE 802.1Q
tags: comm, vlan, qos, ethernet, ieee-802.1q, traffic-priority, pcp
---

## VLAN and QoS Priority Mapping

Use VLAN tagging to isolate traffic domains (safety, diagnostics, infotainment) and QoS priority mapping to ensure time-critical frames are prioritized by Ethernet switches.

**Incorrect (all traffic on same VLAN with no priority):**

```c
void Eth_SendFrame(const uint8_t *data, uint16_t len)
{
    Eth_Transmit(data, len);  /* No VLAN tag, no QoS */
}
```

**Correct (VLAN-tagged with QoS priority):**

```c
typedef struct
{
    uint16_t vlanId;
    uint8_t  priority;     /* PCP: 0 (best effort) to 7 (highest) */
    const char *domain;
} VlanConfig_t;

static const VlanConfig_t g_vlanConfig[] =
{
    { .vlanId = 10U, .priority = 7U, .domain = "Safety-Critical" },
    { .vlanId = 20U, .priority = 5U, .domain = "ADAS-Sensors"    },
    { .vlanId = 30U, .priority = 4U, .domain = "Diagnostics"     },
    { .vlanId = 40U, .priority = 2U, .domain = "OTA-Updates"     },
    { .vlanId = 50U, .priority = 0U, .domain = "Infotainment"    },
};

Std_ReturnType Eth_SetVlanTag(EthFrame_t *frame, uint16_t vlanId,
                               uint8_t priority)
{
    if (priority > 7U) { return E_NOT_OK; }

    frame->vlanTag.tpid = ETH_TPID_8021Q;  /* 0x8100 */
    frame->vlanTag.tci  = (uint16_t)((uint16_t)(priority << 13U) |
                                     (vlanId & 0x0FFFU));
    return E_OK;
}
```

Key automotive Ethernet QoS considerations:
- Map safety-critical traffic (brake, steering) to highest PCP priority (6-7)
- Map diagnostics to mid-priority (3-4)
- Map infotainment/OTA to best-effort priority (0-1)
- Ensure switch hardware supports strict priority queuing for safety traffic

Reference: IEEE 802.1Q; AUTOSAR SWS EthSwt
