---
title: TSN Stream Filtering and Policing (IEEE 802.1Qci)
impact: MEDIUM
impactDescription: Missing stream filtering allows malicious or misconfigured ECUs to flood safety-critical traffic classes, violating isolation guarantees
tags: ethernet, tsn, 802.1qci, stream-filtering, policing, flow-meter, stream-gate, ingress, security, safety
---

## TSN Stream Filtering and Policing (IEEE 802.1Qci)

IEEE 802.1Qci provides per-stream filtering and policing (PSFP) at ingress ports. Stream identification, stream gates (time-based admission), and flow meters (rate limiting) work together to enforce traffic contracts. This is critical for both safety isolation (preventing fault propagation) and security (preventing DoS attacks on the in-vehicle network).

**Incorrect (no ingress filtering — any ECU can send any traffic class):**

```c
/* WRONG: Switch port accepts all traffic without filtering */
void EthSwt_PortInit_Bad(uint8_t port)
{
    EthSwt_SetPortState(port, ETH_PORT_FORWARDING);
    /* No stream filters — a compromised ECU can:
     * - Send frames with PCP=7 (safety priority) flooding safety queues
     * - Exceed its bandwidth allocation
     * - Inject frames into wrong VLANs */
}
```

**Correct (per-stream filtering with gates and metering):**

```c
/* qci_config.h — PSFP data structures */

/* Stream identification: match frames to a stream ID */
typedef struct {
    uint8_t  srcMac[6];         /* Source MAC filter (or wildcard) */
    uint16_t vlanId;            /* VLAN ID filter */
    uint8_t  pcp;               /* Priority code point */
    uint16_t streamHandle;      /* Assigned stream ID */
} StreamId_Entry_t;

/* Stream gate: time-based admission control */
typedef struct {
    uint16_t streamHandle;
    bool     gateOpen;          /* TRUE = frame passes, FALSE = frame dropped */
    uint8_t  ipv;               /* Internal Priority Value override (-1=no override) */
    uint32_t timeInterval_ns;   /* Duration of this gate state */
} StreamGate_Entry_t;

typedef struct {
    StreamGate_Entry_t entries[16];
    uint32_t           entryCount;
    uint32_t           cycleTime_ns;
} StreamGate_Config_t;

/* Flow meter: token bucket rate limiter */
typedef struct {
    uint16_t streamHandle;
    uint32_t cir_kbps;          /* Committed Information Rate */
    uint32_t cbs_bytes;         /* Committed Burst Size */
    uint32_t eir_kbps;          /* Excess Information Rate (0 = strict) */
    uint32_t ebs_bytes;         /* Excess Burst Size */
    bool     dropOnYellow;      /* Drop excess (yellow) traffic */
    bool     markRedAsDrop;     /* Drop or mark non-conforming */
} FlowMeter_Config_t;
```

```c
/* qci_config.c — configure PSFP on switch ingress port */

/* Stream identification table — maps frames to stream handles */
const StreamId_Entry_t g_streamIdTable[] = {
    /* Radar ECU safety stream */
    {
        .srcMac      = {0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x01},
        .vlanId      = VLAN_SAFETY,
        .pcp         = 7,
        .streamHandle = STREAM_RADAR_SAFETY,
    },
    /* Camera ECU sensor stream */
    {
        .srcMac      = {0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x02},
        .vlanId      = VLAN_SENSOR,
        .pcp         = 4,
        .streamHandle = STREAM_CAMERA_DATA,
    },
    /* Diagnostic tester stream */
    {
        .srcMac      = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF}, /* wildcard */
        .vlanId      = VLAN_DIAG,
        .pcp         = 2,
        .streamHandle = STREAM_DIAGNOSTIC,
    },
};

/* Stream gate — safety stream only accepted during its Qbv window */
const StreamGate_Config_t g_radarGate = {
    .cycleTime_ns = 1000000U,  /* 1ms, synchronized with Qbv cycle */
    .entryCount = 2,
    .entries = {
        /* Gate open during safety window (first 200μs of cycle) */
        {
            .streamHandle = STREAM_RADAR_SAFETY,
            .gateOpen = true,
            .ipv = 7,
            .timeInterval_ns = 200000U,
        },
        /* Gate closed for remainder — drops out-of-window frames */
        {
            .streamHandle = STREAM_RADAR_SAFETY,
            .gateOpen = false,
            .ipv = -1,
            .timeInterval_ns = 800000U,
        },
    },
};

/* Flow meter — rate-limit each stream */
const FlowMeter_Config_t g_flowMeters[] = {
    {
        .streamHandle = STREAM_RADAR_SAFETY,
        .cir_kbps     = 10000,    /* 10 Mbps committed rate */
        .cbs_bytes    = 2048,     /* 2KB burst */
        .eir_kbps     = 0,        /* No excess — strict enforcement */
        .ebs_bytes    = 0,
        .dropOnYellow = true,
        .markRedAsDrop = true,
    },
    {
        .streamHandle = STREAM_CAMERA_DATA,
        .cir_kbps     = 50000,    /* 50 Mbps */
        .cbs_bytes    = 8192,
        .eir_kbps     = 5000,     /* 5 Mbps excess allowed */
        .ebs_bytes    = 4096,
        .dropOnYellow = false,    /* Mark but don't drop excess */
        .markRedAsDrop = true,
    },
    {
        .streamHandle = STREAM_DIAGNOSTIC,
        .cir_kbps     = 1000,     /* 1 Mbps — diagnostic is low-bandwidth */
        .cbs_bytes    = 1522,
        .eir_kbps     = 0,
        .ebs_bytes    = 0,
        .dropOnYellow = true,
        .markRedAsDrop = true,
    },
};
```

**Applying PSFP configuration to switch ports:**

```c
void TSN_Qci_ConfigurePort(uint8_t port)
{
    /* Step 1: Configure stream identification */
    for (uint32_t i = 0; i < ARRAY_SIZE(g_streamIdTable); i++)
    {
        EthSwt_Qci_AddStreamFilter(port, &g_streamIdTable[i]);
    }

    /* Step 2: Configure stream gates (synchronized to gPTP) */
    uint64_t gptp_time = gPTP_GetCurrentTime();
    uint64_t baseTime = ((gptp_time / g_radarGate.cycleTime_ns) + 2)
                        * g_radarGate.cycleTime_ns;
    EthSwt_Qci_SetStreamGate(port, STREAM_RADAR_SAFETY,
                              &g_radarGate, baseTime);

    /* Step 3: Configure flow meters */
    for (uint32_t i = 0; i < ARRAY_SIZE(g_flowMeters); i++)
    {
        EthSwt_Qci_SetFlowMeter(port, &g_flowMeters[i]);
    }

    /* Step 4: Set default action for unmatched streams */
    EthSwt_Qci_SetDefaultAction(port, QCI_ACTION_DROP);
}
```

**Monitoring and diagnostics:**

```c
/* Read PSFP counters for diagnostic reporting */
typedef struct {
    uint32_t framesPassedGate;
    uint32_t framesDroppedGate;   /* Dropped by stream gate (wrong timing) */
    uint32_t framesDroppedMeter;  /* Dropped by flow meter (rate exceeded) */
    uint32_t framesDroppedFilter; /* Dropped by stream filter (unknown stream) */
} Qci_PortStats_t;

void TSN_Qci_ReadStats(uint8_t port, Qci_PortStats_t *stats)
{
    EthSwt_Qci_GetCounters(port, stats);

    /* Report excessive drops as diagnostic events */
    if (stats->framesDroppedGate > QCI_GATE_DROP_THRESHOLD)
    {
        Dem_ReportErrorStatus(DEM_EVENT_QCI_GATE_DROPS,
                              DEM_EVENT_STATUS_FAILED);
    }
    if (stats->framesDroppedMeter > QCI_METER_DROP_THRESHOLD)
    {
        Dem_ReportErrorStatus(DEM_EVENT_QCI_METER_DROPS,
                              DEM_EVENT_STATUS_FAILED);
    }
}
```

Stream gate cycle times must be synchronized with Qbv gate control lists on the same port. The default action for unmatched streams should be DROP in security-sensitive deployments. Flow meter CIR/CBS values must match the traffic contract defined in the network design.

Reference: IEEE 802.1Qci-2017 (Per-Stream Filtering and Policing); AUTOSAR SWS_EthSwt; UNECE R155 (Cybersecurity — network isolation)
