---
title: Audio Video Bridging for Infotainment Streaming
impact: LOW-MEDIUM
impactDescription: Incorrect AVB configuration causes audio glitches, video artifacts, or bandwidth starvation of other traffic classes
tags: ethernet, avb, avtp, srp, streaming, infotainment, talker, listener, bandwidth-reservation
---

## Audio Video Bridging for Infotainment Streaming

AVB (Audio Video Bridging) provides guaranteed bandwidth and bounded latency for infotainment streaming over automotive Ethernet. AVTP (Audio/Video Transport Protocol, IEEE 1722) transports media data. SRP (Stream Reservation Protocol, IEEE 802.1Qat) reserves bandwidth along the path from talker to listener. Proper talker/listener lifecycle management prevents bandwidth leaks.

**Incorrect (streaming without bandwidth reservation, ignoring presentation time):**

```c
/* WRONG: Sending media data as raw Ethernet without AVB reservation */
void Audio_Stream_Bad(const int16_t *samples, uint32_t numSamples)
{
    EthFrame_t frame;
    frame.etherType = 0x22F0;  /* AVTP EtherType */
    (void)memcpy(frame.payload, samples, numSamples * sizeof(int16_t));
    ETH_Transmit(&frame, sizeof(frame));
    /* No SRP reservation — switch may drop frames under congestion */
    /* No presentation timestamp — listener has no timing reference */
    /* No sequence number — listener cannot detect packet loss */
}
```

**Correct (full AVB stack with SRP reservation and AVTP framing):**

```c
/* avb_types.h — AVTP and SRP data structures */

/* AVTP common stream header (IEEE 1722-2016) */
typedef struct __attribute__((packed)) {
    uint8_t  subtype;           /* AVTP subtype (0x00 = AAF audio) */
    uint8_t  sv_ver_mr_tv;     /* StreamValid, Version, MediaReset, TimestampValid */
    uint8_t  sequenceNum;       /* Packet sequence counter */
    uint8_t  tu;                /* Timestamp Uncertain */
    uint8_t  streamId[8];       /* Unique stream identifier */
    uint32_t avtp_timestamp;    /* Presentation time (gPTP-based) */
    uint32_t format_specific;   /* Format-dependent (sample rate, channels, etc.) */
    uint16_t stream_data_len;   /* Payload length */
} AVTP_StreamHeader_t;

/* AAF (AVTP Audio Format) specific */
typedef struct __attribute__((packed)) {
    uint8_t  nsr_channels;      /* Nominal sample rate + channel count */
    uint8_t  bit_depth;         /* Bits per sample */
    uint16_t frames_per_pdu;    /* Audio frames per AVTP PDU */
} AVTP_AAF_Format_t;

/* SRP stream reservation */
typedef struct {
    uint8_t  streamId[8];
    uint8_t  destMac[6];
    uint16_t vlanId;
    uint8_t  priority;          /* SR class (A=3, B=2) */
    uint32_t maxFrameSize;
    uint32_t maxIntervalFrames; /* Frames per observation interval */
    uint32_t accumulatedLatency_ns;
} SRP_TalkerAdvertise_t;
```

```c
/* avb_talker.c — audio talker implementation */

#define STREAM_SAMPLE_RATE    48000U
#define STREAM_CHANNELS       2U
#define STREAM_BIT_DEPTH      16U
#define FRAMES_PER_PDU        6U     /* 6 frames × 2 ch × 2 bytes = 24 bytes/PDU */
#define CLASS_A_INTERVAL_US   125U   /* Class A: 8000 PDUs/sec */
#define PRESENTATION_OFFSET_NS 2000000U  /* 2ms presentation latency */

static uint8_t g_sequenceNum = 0U;
static bool g_streamReserved = false;

/* Step 1: Register stream with SRP (before sending any data) */
void AVB_Talker_RegisterStream(void)
{
    SRP_TalkerAdvertise_t talkerAdv = {
        .streamId  = {0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x01, 0x00, 0x01},
        .destMac   = {0x91, 0xE0, 0xF0, 0x00, 0x01, 0x00},  /* AVB multicast */
        .vlanId    = VLAN_INFOTAINMENT,
        .priority  = 3,            /* SR Class A */
        .maxFrameSize = sizeof(AVTP_StreamHeader_t) + sizeof(AVTP_AAF_Format_t)
                        + (FRAMES_PER_PDU * STREAM_CHANNELS * (STREAM_BIT_DEPTH / 8)),
        .maxIntervalFrames = 1,    /* One frame per Class A interval */
        .accumulatedLatency_ns = 0,
    };

    SRP_RegisterTalker(&talkerAdv);
    /* Wait for SRP Listener Ready indication before streaming */
}

/* SRP callback: listener has reserved bandwidth */
void SRP_ListenerReady_Callback(const uint8_t *streamId)
{
    g_streamReserved = true;
}

/* Step 2: Send AVTP audio frames at Class A interval */
void AVB_Talker_SendAudio(const int16_t *samples)
{
    if (!g_streamReserved) { return; }

    /* Build AVTP frame */
    uint8_t pdu[256];
    uint32_t offset = 0;

    /* Ethernet header with VLAN tag */
    EthVlanHeader_t *eth = (EthVlanHeader_t *)pdu;
    (void)memcpy(eth->destMac, g_streamDestMac, 6);
    (void)memcpy(eth->srcMac, g_localMac, 6);
    eth->vlanTag.tpid = htons(0x8100);
    eth->vlanTag.pcp_vid = htons((3U << 13) | VLAN_INFOTAINMENT);  /* PCP=3 */
    eth->etherType = htons(0x22F0);  /* AVTP */
    offset += sizeof(EthVlanHeader_t);

    /* AVTP stream header */
    AVTP_StreamHeader_t *avtp = (AVTP_StreamHeader_t *)&pdu[offset];
    avtp->subtype = 0x02;       /* AAF subtype */
    avtp->sv_ver_mr_tv = 0x81;  /* StreamValid=1, TimestampValid=1 */
    avtp->sequenceNum = g_sequenceNum++;
    avtp->tu = 0;
    (void)memcpy(avtp->streamId, g_streamId, 8);

    /* Presentation timestamp: current gPTP time + offset */
    uint64_t gptp_now = gPTP_GetCurrentTime();
    avtp->avtp_timestamp = htonl((uint32_t)(gptp_now + PRESENTATION_OFFSET_NS));

    /* AAF format info */
    avtp->stream_data_len = htons(FRAMES_PER_PDU * STREAM_CHANNELS
                                   * (STREAM_BIT_DEPTH / 8));
    offset += sizeof(AVTP_StreamHeader_t);

    /* AAF format header */
    AVTP_AAF_Format_t *aaf = (AVTP_AAF_Format_t *)&pdu[offset];
    aaf->nsr_channels = (0x05 << 4) | STREAM_CHANNELS;  /* 48kHz, 2ch */
    aaf->bit_depth = STREAM_BIT_DEPTH;
    aaf->frames_per_pdu = htons(FRAMES_PER_PDU);
    offset += sizeof(AVTP_AAF_Format_t);

    /* Audio payload — network byte order (big-endian) */
    for (uint32_t i = 0; i < FRAMES_PER_PDU * STREAM_CHANNELS; i++)
    {
        int16_t sample_be = htons(samples[i]);
        (void)memcpy(&pdu[offset], &sample_be, sizeof(int16_t));
        offset += sizeof(int16_t);
    }

    ETH_Transmit(pdu, offset);
}
```

**AVB listener — receiving and rendering audio:**

```c
/* avb_listener.c */

/* Step 1: Subscribe to stream via SRP */
void AVB_Listener_Subscribe(const uint8_t *streamId)
{
    SRP_RegisterListener(streamId, SRP_LISTENER_READY);
}

/* Step 2: Receive and buffer AVTP frames */
void AVB_Listener_ReceiveFrame(const uint8_t *frame, uint16_t len)
{
    const AVTP_StreamHeader_t *avtp =
        (const AVTP_StreamHeader_t *)(frame + sizeof(EthVlanHeader_t));

    /* Check sequence number for packet loss detection */
    static uint8_t expectedSeq = 0;
    if (avtp->sequenceNum != expectedSeq)
    {
        uint8_t lost = avtp->sequenceNum - expectedSeq;
        AVB_Stats_RecordLoss(lost);
        /* Insert silence for lost packets */
        AudioBuffer_InsertSilence(lost * FRAMES_PER_PDU);
    }
    expectedSeq = avtp->sequenceNum + 1;

    /* Extract presentation timestamp */
    uint32_t presentTime = ntohl(avtp->avtp_timestamp);

    /* Buffer audio with presentation timestamp */
    const int16_t *samples = (const int16_t *)(frame + sizeof(EthVlanHeader_t)
                              + sizeof(AVTP_StreamHeader_t)
                              + sizeof(AVTP_AAF_Format_t));
    AudioBuffer_Enqueue(samples, FRAMES_PER_PDU * STREAM_CHANNELS,
                         presentTime);
}

/* Step 3: Render audio at presentation time */
void AVB_Listener_RenderTask(void)
{
    uint32_t gptp_now = (uint32_t)gPTP_GetCurrentTime();

    int16_t renderBuf[FRAMES_PER_PDU * STREAM_CHANNELS];
    uint32_t presentTime;

    if (AudioBuffer_PeekTime(&presentTime))
    {
        int32_t delta = (int32_t)(presentTime - gptp_now);

        if (delta <= 0)
        {
            /* Presentation time reached — render now */
            AudioBuffer_Dequeue(renderBuf, FRAMES_PER_PDU * STREAM_CHANNELS);
            AudioDAC_Output(renderBuf, FRAMES_PER_PDU);
        }
        /* else: wait — frame is not due yet */
    }
}
```

**Stream teardown — release SRP reservation:**

```c
/* Clean stream shutdown — must deregister to release bandwidth */
void AVB_Talker_StopStream(void)
{
    g_streamReserved = false;
    SRP_DeregisterTalker(g_streamId);
    /* Switch releases reserved bandwidth on all hops */
}

void AVB_Listener_Unsubscribe(const uint8_t *streamId)
{
    SRP_DeregisterListener(streamId);
    AudioBuffer_Flush();
}
```

Always register with SRP before sending AVTP frames — unregistered streams may be dropped by QoS-aware switches. The presentation time offset must account for worst-case network latency plus jitter buffer depth. Deregister streams on shutdown to release bandwidth reservations.

Reference: IEEE 1722-2016 (AVTP); IEEE 802.1Qat (SRP); IEEE 802.1BA (Audio Video Bridging Systems); AUTOSAR SWS_EthTSyn
