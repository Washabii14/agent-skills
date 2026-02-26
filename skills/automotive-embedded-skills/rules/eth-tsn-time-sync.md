---
title: TSN Time Synchronization (IEEE 802.1AS / gPTP)
impact: HIGH
impactDescription: Clock synchronization errors exceeding 1μs cause safety-critical TSN traffic to miss gate windows, leading to frame drops or deadline violations
tags: ethernet, tsn, gptp, 802.1as, time-sync, grandmaster, ptp, automotive, clock
---

## TSN Time Synchronization (IEEE 802.1AS / gPTP)

IEEE 802.1AS (gPTP) synchronizes all devices in a TSN domain to a common time base. A grandmaster clock is selected via the Best Master Clock Algorithm (BMCA). Sync, Follow_Up, and Pdelay messages establish and maintain synchronization. Automotive requires <1μs accuracy across the in-vehicle Ethernet network.

**Incorrect (no peer delay measurement, no rate correction):**

```c
/* WRONG: Applying sync offset without path delay or rate correction */
void gPTP_HandleSync_Bad(const gPTP_SyncMsg_t *sync)
{
    /* Directly setting local clock — no delay compensation */
    g_localTime = sync->originTimestamp;
    /* Error accumulates: no peer delay, no frequency drift correction */
    /* Actual offset could be 10-100μs — violates <1μs requirement */
}
```

**Correct (full gPTP implementation with peer delay and rate ratio):**

```c
/* gptp.h — gPTP data structures */
typedef struct {
    int64_t  seconds;
    int32_t  nanoseconds;
} gPTP_Timestamp_t;

typedef struct {
    gPTP_Timestamp_t syncTxTime;       /* T1: master transmit time */
    gPTP_Timestamp_t syncRxTime;       /* T2: slave receive time */
    gPTP_Timestamp_t pdelayReqTxTime;  /* T3: pdelay request Tx */
    gPTP_Timestamp_t pdelayReqRxTime;  /* T4: pdelay request Rx (remote) */
    gPTP_Timestamp_t pdelayRespTxTime; /* T5: pdelay response Tx (remote) */
    gPTP_Timestamp_t pdelayRespRxTime; /* T6: pdelay response Rx */
    int64_t          peerDelay;        /* Computed peer delay (ns) */
    double           neighborRateRatio; /* Frequency ratio to neighbor */
} gPTP_PortData_t;

static gPTP_PortData_t g_portData;
```

```c
/* gptp.c — peer delay measurement (Pdelay mechanism) */

void gPTP_SendPdelayReq(void)
{
    gPTP_PdelayReqMsg_t req = {0};
    req.header.messageType = GPTP_MSG_PDELAY_REQ;
    req.header.sequenceId = g_pdelaySeqId++;

    /* Hardware timestamping captures precise Tx time */
    ETH_SendWithTimestamp(&req, sizeof(req), &g_portData.pdelayReqTxTime);
}

void gPTP_HandlePdelayResp(const gPTP_PdelayRespMsg_t *resp)
{
    /* T4 (remote Rx time of our request) from response */
    g_portData.pdelayReqRxTime = resp->requestReceiptTimestamp;
}

void gPTP_HandlePdelayRespFollowUp(const gPTP_PdelayRespFollowUpMsg_t *fup)
{
    /* T5 (remote Tx time of response) */
    g_portData.pdelayRespTxTime = fup->responseOriginTimestamp;

    /* T6 was captured by hardware when we received the Pdelay_Resp */

    /* Compute peer delay:
     * peerDelay = ((T6 - T3) - (T5 - T4)) / 2
     * This eliminates clock offset between peers */
    int64_t round_trip = Timestamp_DiffNs(&g_portData.pdelayRespRxTime,
                                           &g_portData.pdelayReqTxTime);
    int64_t remote_processing = Timestamp_DiffNs(&g_portData.pdelayRespTxTime,
                                                  &g_portData.pdelayReqRxTime);
    g_portData.peerDelay = (round_trip - remote_processing) / 2;

    /* Compute neighbor rate ratio for frequency correction */
    g_portData.neighborRateRatio =
        (double)(Timestamp_DiffNs(&g_portData.pdelayRespRxTime,
                                   &g_portData.pdelayRespRxTimePrev)) /
        (double)(Timestamp_DiffNs(&g_portData.pdelayRespTxTime,
                                   &g_portData.pdelayRespTxTimePrev));
}
```

**Sync/Follow_Up handling with offset correction:**

```c
/* Slave clock synchronization */
void gPTP_HandleSync(const gPTP_SyncMsg_t *sync,
                      const gPTP_Timestamp_t *rxTimestamp)
{
    g_portData.syncRxTime = *rxTimestamp;  /* T2: hardware Rx timestamp */
}

void gPTP_HandleFollowUp(const gPTP_FollowUpMsg_t *fup)
{
    /* T1: precise Tx timestamp from grandmaster */
    g_portData.syncTxTime = fup->preciseOriginTimestamp;

    /* Compute clock offset:
     * offset = (T2 - T1) - peerDelay - correctionField
     * correctionField accumulates residence time through switches */
    int64_t rawOffset = Timestamp_DiffNs(&g_portData.syncRxTime,
                                          &g_portData.syncTxTime);
    int64_t correctedOffset = rawOffset
                              - g_portData.peerDelay
                              - fup->header.correctionField;

    /* Apply clock discipline — PI controller for smooth convergence */
    gPTP_ClockServo(correctedOffset, g_portData.neighborRateRatio);
}

/* PI controller for clock adjustment */
static void gPTP_ClockServo(int64_t offsetNs, double rateRatio)
{
    static int64_t integralTerm = 0;
    const int64_t Kp = 1;          /* Proportional gain (phase) */
    const int64_t Ki = 1;          /* Integral gain (frequency) */
    const int64_t maxIntegral = 1000; /* Anti-windup limit (ns) */

    /* Phase correction */
    int64_t phaseAdj = Kp * offsetNs / 16;

    /* Frequency correction */
    integralTerm += Ki * offsetNs / 256;
    if (integralTerm > maxIntegral) { integralTerm = maxIntegral; }
    if (integralTerm < -maxIntegral) { integralTerm = -maxIntegral; }

    /* Apply to hardware timer or PTP clock */
    int64_t totalAdj = phaseAdj + integralTerm;
    ETH_PTP_AdjustClock(totalAdj);

    /* Also adjust frequency using rate ratio */
    ETH_PTP_AdjustFrequency(rateRatio);
}
```

**Grandmaster clock selection (BMCA):**

```c
/* Best Master Clock Algorithm — determines which device is grandmaster */
typedef struct {
    uint8_t  priority1;        /* 0-255, lower is better */
    uint8_t  clockClass;       /* 6=primary, 7=holdover, 52=app-specific */
    uint8_t  clockAccuracy;    /* IEEE defined accuracy enum */
    uint16_t offsetScaledLogVariance;
    uint8_t  priority2;        /* Tiebreaker */
    uint8_t  clockIdentity[8]; /* Unique clock ID */
} gPTP_ClockQuality_t;

/* Automotive: gateway ECU or domain controller is typically GM
 * with priority1 = 128 (default) or lower */
const gPTP_ClockQuality_t g_localClockQuality = {
    .priority1 = 128,
    .clockClass = 248,     /* Default */
    .clockAccuracy = 0x22, /* 250ns */
    .priority2 = 128,
};
```

Use hardware timestamping (PTP hardware clock in the Ethernet MAC) — software timestamps introduce jitter that makes <1μs accuracy impossible. The Pdelay mechanism measures link delay independently on each hop, enabling accurate multi-hop synchronization through TSN switches.

Reference: IEEE 802.1AS-2020 (Timing and Synchronization); AUTOSAR SWS_EthTSyn; ISO 26262 Part 6 (Time-critical network communication)
