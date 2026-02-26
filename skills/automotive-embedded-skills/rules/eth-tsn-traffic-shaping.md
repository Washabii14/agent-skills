---
title: TSN Traffic Shaping (IEEE 802.1Qbv)
impact: HIGH
impactDescription: Incorrect gate control lists cause safety-critical frames to be blocked or delayed beyond their deadline, violating real-time guarantees
tags: ethernet, tsn, 802.1qbv, traffic-shaping, gate-control, time-aware, scheduling, guard-band, priority
---

## TSN Traffic Shaping (IEEE 802.1Qbv)

IEEE 802.1Qbv defines time-aware traffic shaping where gate control lists (GCLs) open and close transmission gates for each traffic class according to a periodic schedule. This guarantees bounded latency for safety-critical traffic by reserving exclusive time windows. Guard bands prevent lower-priority traffic from blocking gate transitions.

**Incorrect (no guard band, overlapping traffic classes, wrong gate timing):**

```c
/* WRONG: Safety traffic shares window with best-effort, no guard band */
const GCL_Entry_t gcl_bad[] = {
    /* Gate state: all queues open simultaneously */
    { .gateState = 0xFF,   /* All 8 queues open */
      .timeInterval_ns = 1000000 },  /* 1ms — everything fights for bandwidth */
    /* No guard band — best-effort frame in transmission blocks safety frame */
};
```

**Correct (dedicated time windows with guard bands):**

```c
/* tsn_qbv.h — traffic class to queue mapping */
/*
 * Traffic Class (TC)  | Priority | Queue | Usage
 * --------------------|----------|-------|------------------
 * TC7 (highest)       | 7        | Q7    | Safety-critical (ASIL)
 * TC6                 | 6        | Q6    | Time-sync (gPTP)
 * TC5                 | 5        | Q5    | Control data
 * TC4                 | 4        | Q4    | Sensor data
 * TC2-3               | 2-3      | Q2-3  | Best-effort
 * TC0-1 (lowest)      | 0-1      | Q0-1  | Background/diagnostics
 */

#define GATE_Q7  (1U << 7)  /* Safety-critical */
#define GATE_Q6  (1U << 6)  /* gPTP */
#define GATE_Q5  (1U << 5)  /* Control */
#define GATE_Q4  (1U << 4)  /* Sensor */
#define GATE_BE  (0x0FU)    /* Q0-Q3: best-effort */
#define GATE_ALL (0xFFU)

typedef struct {
    uint8_t  gateState;        /* Bitmask: 1 = gate open for queue */
    uint32_t timeInterval_ns;  /* Duration of this gate state */
} GCL_Entry_t;

typedef struct {
    GCL_Entry_t entries[8];
    uint32_t    entryCount;
    uint32_t    cycleTime_ns;       /* Total GCL cycle period */
    uint64_t    baseTime_ns;        /* Aligned to gPTP time */
    uint32_t    guardBand_ns;       /* Maximum frame transmission time */
} GCL_Config_t;
```

```c
/* tsn_qbv.c — gate control list with proper guard bands */

/* Guard band = max frame size / link speed
 * For 100Mbps: 1522 bytes * 8 bits / 100Mbps = 121.76μs ≈ 125μs
 * For 1Gbps:   1522 bytes * 8 bits / 1Gbps   = 12.18μs  ≈ 15μs  */
#define GUARD_BAND_100M_NS  125000U
#define GUARD_BAND_1G_NS     15000U

/* 1ms cycle, 100Mbps link */
const GCL_Config_t g_gclConfig = {
    .cycleTime_ns = 1000000U,  /* 1ms cycle */
    .guardBand_ns = GUARD_BAND_100M_NS,
    .entryCount = 4,
    .entries = {
        /* Window 1: Safety-critical only (200μs exclusive) */
        {
            .gateState = GATE_Q7 | GATE_Q6,  /* Only safety + gPTP */
            .timeInterval_ns = 200000U,
        },
        /* Window 2: Control + sensor data (300μs) */
        {
            .gateState = GATE_Q5 | GATE_Q4,
            .timeInterval_ns = 300000U,
        },
        /* Window 3: Best-effort traffic (375μs) */
        {
            .gateState = GATE_BE,
            .timeInterval_ns = 375000U,  /* 500 - 125 guard band */
        },
        /* Window 4: Guard band — all gates closed
         * Prevents BE frame from spanning into next safety window */
        {
            .gateState = 0x00U,  /* All gates closed */
            .timeInterval_ns = GUARD_BAND_100M_NS,
        },
    },
};
```

**Programming the switch/MAC gate control list:**

```c
/* Apply GCL to Ethernet switch port or MAC */
void TSN_Qbv_Configure(uint8_t port, const GCL_Config_t *config)
{
    /* Disable Qbv during reconfiguration */
    EthSwt_Qbv_Disable(port);

    /* Set base time aligned to gPTP grandmaster */
    uint64_t gptp_time = gPTP_GetCurrentTime();
    uint64_t nextCycleStart = ((gptp_time / config->cycleTime_ns) + 2)
                              * config->cycleTime_ns;
    EthSwt_Qbv_SetBaseTime(port, nextCycleStart);

    /* Set cycle time */
    EthSwt_Qbv_SetCycleTime(port, config->cycleTime_ns);

    /* Program gate control list entries */
    for (uint32_t i = 0U; i < config->entryCount; i++)
    {
        EthSwt_Qbv_SetGCLEntry(port, i,
                                config->entries[i].gateState,
                                config->entries[i].timeInterval_ns);
    }

    /* Enable Qbv — takes effect at next base time */
    EthSwt_Qbv_Enable(port);
}

/* Map VLAN priority to traffic class (PCP → TC mapping) */
void TSN_ConfigurePriorityMapping(uint8_t port)
{
    /* IEEE 802.1Q default priority mapping */
    const uint8_t pcp_to_tc[8] = {
        /* PCP: 0  1  2  3  4  5  6  7 */
                1, 0, 2, 3, 4, 5, 6, 7  /* TC assignment */
    };

    for (uint8_t pcp = 0U; pcp < 8U; pcp++)
    {
        EthSwt_SetPriorityMapping(port, pcp, pcp_to_tc[pcp]);
    }
}
```

**Frame tagging at the application layer:**

```c
/* Application must tag frames with correct VLAN priority */
void App_SendSafetyFrame(const uint8_t *data, uint16_t len)
{
    EthFrame_t frame;
    frame.vlanTag.pcp = 7U;          /* Highest priority → TC7 → Q7 */
    frame.vlanTag.dei = 0U;
    frame.vlanTag.vid = VLAN_SAFETY;
    frame.etherType = ETHTYPE_SAFETY_PROTOCOL;

    (void)memcpy(frame.payload, data, len);
    ETH_Transmit(&frame, sizeof(EthFrame_t));
}
```

**C++ configuration builder:**

```cpp
class GCLBuilder
{
public:
    GCLBuilder& SetCycleTime(uint32_t ns) { config_.cycleTime_ns = ns; return *this; }
    GCLBuilder& SetGuardBand(uint32_t ns) { config_.guardBand_ns = ns; return *this; }

    GCLBuilder& AddWindow(uint8_t gates, uint32_t duration_ns)
    {
        if (config_.entryCount < 8)
        {
            config_.entries[config_.entryCount++] = {gates, duration_ns};
        }
        return *this;
    }

    GCL_Config_t Build()
    {
        uint32_t total = 0;
        for (uint32_t i = 0; i < config_.entryCount; i++)
            total += config_.entries[i].timeInterval_ns;

        // Verify cycle time matches sum of windows
        assert(total == config_.cycleTime_ns);
        return config_;
    }

private:
    GCL_Config_t config_{};
};

// Usage
auto gcl = GCLBuilder()
    .SetCycleTime(1'000'000)
    .SetGuardBand(125'000)
    .AddWindow(GATE_Q7 | GATE_Q6, 200'000)
    .AddWindow(GATE_Q5 | GATE_Q4, 300'000)
    .AddWindow(GATE_BE, 375'000)
    .AddWindow(0x00, 125'000)  // Guard band
    .Build();
```

The guard band duration must equal the maximum frame transmission time at the port speed. GCL base time must be synchronized to gPTP time — if time sync is lost, traffic shaping degrades. Verify the sum of all gate intervals equals the cycle time exactly.

Reference: IEEE 802.1Qbv-2015 (Time-Aware Shaping); AUTOSAR SWS_EthSwt; ISO 26262 Part 6 (Deterministic communication)
