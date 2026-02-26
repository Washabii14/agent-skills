---
title: Automotive Ethernet Switch Configuration
impact: MEDIUM
impactDescription: Misconfigured switches cause network loops, unintended traffic leakage between security domains, or loss of diagnostic visibility
tags: ethernet, switch, vlan, mac-filtering, port-mirroring, stp, rstp, bandwidth, diagnostics
---

## Automotive Ethernet Switch Configuration

Automotive Ethernet switches require careful configuration of port-based VLANs for domain isolation, MAC address filtering for security, port mirroring for diagnostics, STP/RSTP for loop prevention, and per-port bandwidth limitation. These settings are typically configured at boot and must match the vehicle network architecture.

**Incorrect (flat network, no VLANs, no loop protection):**

```c
/* WRONG: All ports in one broadcast domain, no isolation */
void EthSwt_Init_Bad(void)
{
    for (uint8_t port = 0; port < NUM_SWITCH_PORTS; port++)
    {
        EthSwt_SetPortState(port, ETH_PORT_FORWARDING);
        /* No VLAN — all ECUs see all traffic */
        /* No STP — network loop causes broadcast storm */
        /* No MAC filter — spoofing possible */
    }
}
```

**Correct (VLAN isolation, MAC filtering, port mirroring):**

```c
/* eth_switch.h — switch configuration structures */
typedef struct {
    uint8_t  port;
    uint16_t pvid;          /* Port VLAN ID (untagged frames get this VID) */
    uint16_t allowedVlans;  /* Bitmask of allowed VLANs */
    bool     tagOnEgress;   /* Tag frames leaving this port */
} PortVlanConfig_t;

typedef struct {
    uint8_t port;
    uint8_t allowedMacs[MAX_MAC_PER_PORT][6];
    uint8_t macCount;
    bool    dropUnknownSrc;  /* Drop frames from unknown source MACs */
} PortMacFilter_t;
```

```c
/* eth_switch.c — VLAN configuration */
/*
 * VLAN layout:
 *   VLAN 10: Safety domain (ADAS, braking, steering)
 *   VLAN 20: Body domain (lighting, HVAC, doors)
 *   VLAN 30: Infotainment domain
 *   VLAN 40: Diagnostics (accessible from all domains)
 *   VLAN 99: Management (switch config, firmware update)
 */

#define VLAN_SAFETY       10U
#define VLAN_BODY          20U
#define VLAN_INFOTAINMENT  30U
#define VLAN_DIAG          40U
#define VLAN_MGMT          99U

const PortVlanConfig_t g_vlanConfig[] = {
    /* Port 0: ADAS ECU — safety domain only + diagnostics */
    { .port = 0, .pvid = VLAN_SAFETY,
      .allowedVlans = (1U << VLAN_SAFETY) | (1U << VLAN_DIAG),
      .tagOnEgress = false },

    /* Port 1: Brake ECU — safety domain only */
    { .port = 1, .pvid = VLAN_SAFETY,
      .allowedVlans = (1U << VLAN_SAFETY) | (1U << VLAN_DIAG),
      .tagOnEgress = false },

    /* Port 2: Body controller — body domain */
    { .port = 2, .pvid = VLAN_BODY,
      .allowedVlans = (1U << VLAN_BODY) | (1U << VLAN_DIAG),
      .tagOnEgress = false },

    /* Port 3: Head unit — infotainment, isolated from safety */
    { .port = 3, .pvid = VLAN_INFOTAINMENT,
      .allowedVlans = (1U << VLAN_INFOTAINMENT) | (1U << VLAN_DIAG),
      .tagOnEgress = false },

    /* Port 4: Uplink to gateway — carries all VLANs tagged */
    { .port = 4, .pvid = VLAN_MGMT,
      .allowedVlans = 0xFFFFU,  /* All VLANs */
      .tagOnEgress = true },

    /* Port 5: Diagnostic port (OBD) */
    { .port = 5, .pvid = VLAN_DIAG,
      .allowedVlans = (1U << VLAN_DIAG),
      .tagOnEgress = false },
};

void EthSwt_ConfigureVlans(void)
{
    for (uint32_t i = 0; i < ARRAY_SIZE(g_vlanConfig); i++)
    {
        const PortVlanConfig_t *cfg = &g_vlanConfig[i];
        EthSwt_SetPortVID(cfg->port, cfg->pvid);
        EthSwt_SetAllowedVlans(cfg->port, cfg->allowedVlans);
        EthSwt_SetEgressTagging(cfg->port, cfg->tagOnEgress);
    }
}
```

**MAC address filtering per port:**

```c
/* Restrict which source MACs are accepted on each port */
const PortMacFilter_t g_macFilters[] = {
    {
        .port = 0,
        .macCount = 1,
        .allowedMacs = {{0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x01}},  /* ADAS ECU */
        .dropUnknownSrc = true,
    },
    {
        .port = 1,
        .macCount = 1,
        .allowedMacs = {{0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x02}},  /* Brake ECU */
        .dropUnknownSrc = true,
    },
    {
        .port = 5,
        .macCount = 0,
        .dropUnknownSrc = false,  /* Diag port accepts any tester MAC */
    },
};

void EthSwt_ConfigureMacFilters(void)
{
    for (uint32_t i = 0; i < ARRAY_SIZE(g_macFilters); i++)
    {
        const PortMacFilter_t *cfg = &g_macFilters[i];
        for (uint8_t m = 0; m < cfg->macCount; m++)
        {
            EthSwt_AddStaticMacEntry(cfg->port, cfg->allowedMacs[m]);
        }
        EthSwt_SetUnknownSrcDropPolicy(cfg->port, cfg->dropUnknownSrc);
    }
}
```

**Port mirroring for diagnostics:**

```c
/* Mirror traffic from monitored ports to a diagnostic capture port */
typedef struct {
    uint8_t monitorPort;       /* Port to copy traffic TO */
    uint8_t sourcePorts;       /* Bitmask of ports to monitor */
    bool    mirrorIngress;     /* Copy incoming frames */
    bool    mirrorEgress;      /* Copy outgoing frames */
} PortMirrorConfig_t;

const PortMirrorConfig_t g_mirrorConfig = {
    .monitorPort  = 5,                  /* Diagnostic/OBD port */
    .sourcePorts  = (1U << 0) | (1U << 1),  /* Mirror ADAS + Brake ports */
    .mirrorIngress = true,
    .mirrorEgress  = true,
};

void EthSwt_ConfigureMirroring(void)
{
    EthSwt_SetMirrorPort(g_mirrorConfig.monitorPort);
    EthSwt_SetMirrorSources(g_mirrorConfig.sourcePorts,
                             g_mirrorConfig.mirrorIngress,
                             g_mirrorConfig.mirrorEgress);
    /* Mirrored frames are tagged with original VLAN for analysis */
}
```

**STP/RSTP for loop prevention:**

```c
/* RSTP configuration — prevents broadcast storms from wiring errors */
void EthSwt_ConfigureRSTP(void)
{
    /* Enable RSTP on the switch */
    EthSwt_RSTP_Enable(true);

    /* Bridge priority — lower = more likely to be root */
    EthSwt_RSTP_SetBridgePriority(32768U);  /* Default priority */

    /* Port-specific settings */
    for (uint8_t port = 0; port < NUM_SWITCH_PORTS; port++)
    {
        EthSwt_RSTP_SetPortPathCost(port, 200000U);  /* 100Mbps default */
        EthSwt_RSTP_SetPortEdge(port, false);         /* Non-edge by default */
    }

    /* Diagnostic port is edge port — no STP on tester connection */
    EthSwt_RSTP_SetPortEdge(5, true);
}
```

**Per-port bandwidth limitation:**

```c
/* Limit ingress rate per port to prevent bandwidth hogging */
void EthSwt_ConfigureBandwidthLimits(void)
{
    /* ADAS ECU: 50 Mbps max ingress (safety data is bounded) */
    EthSwt_SetIngressRateLimit(0, 50000U);  /* kbps */

    /* Infotainment: 80 Mbps (streaming, but must not starve safety) */
    EthSwt_SetIngressRateLimit(3, 80000U);

    /* Diagnostic port: 10 Mbps (prevent diagnostic flood) */
    EthSwt_SetIngressRateLimit(5, 10000U);

    /* Uplink to gateway: no limit (aggregation port) */
    EthSwt_SetIngressRateLimit(4, 0U);  /* 0 = no limit */
}
```

VLAN isolation between safety and infotainment domains is a cybersecurity requirement (UNECE R155). Port mirroring should only be enabled on diagnostic ports — leaving it active on production ports wastes bandwidth. RSTP convergence time (~1s) must be acceptable for the network's availability requirements.

Reference: IEEE 802.1Q (VLANs); IEEE 802.1w (RSTP); AUTOSAR SWS_EthSwt; UNECE R155 (Cybersecurity — network segmentation)
