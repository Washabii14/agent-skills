---
title: PDU Router — Routing Paths, Gateway Routing, and TP Routing
impact: MEDIUM
impactDescription: Misconfigured PDU routing causes lost messages, gateway loops, or TP segmentation failures across bus systems
tags: autosar, classic, pdu-router, pdur, gateway, tp-routing, routing-path, can, ethernet
---

## PDU Router — Routing Paths, Gateway Routing, and TP Routing

The PDU Router (PduR) in AUTOSAR Classic R22-11 is the central I-PDU switching entity. It routes I-PDUs between COM, DCM, LDCOM, and the interface/TP modules (CanIf, CanTp, FrIf, SoAd, etc.). Routing path misconfiguration is a common integration defect.

### Routing Path Configuration

**Incorrect (hardcoded PDU forwarding in application code):**

```c
void OnCanRx(PduIdType rxPdu, const PduInfoType *pduInfo)
{
    /* Manual forwarding — bypasses PduR entirely */
    FrIf_Transmit(FR_TX_PDU_42, pduInfo);
}
```

**Correct (PduR routing paths configured declaratively):**

```c
/* ARXML PduRRoutingPath configuration:
 *
 * PduRRoutingPath: Route_EngineStatus_CAN_to_FR
 *   PduRSrcPdu:
 *     PduRSrcPduRef: /CanIf/RxPdu_EngineStatus
 *     PduRSrcModule: CanIf
 *   PduRDestPdu:
 *     PduRDestPduRef: /FrIf/TxPdu_EngineStatus
 *     PduRDestModule: FrIf
 *     PduRTransmissionConfirmation: TRUE
 *
 * PduR_CanIfRxIndication() -> PduR routes -> FrIf_Transmit()
 */

/* Application only interacts through COM, never through PduR directly */
void App_SendEngineStatus(void)
{
    Com_SendSignal(ComConf_ComSignal_EngineStatus, &status);
    /* COM -> PduR -> CanIf -> CAN Driver (per routing table) */
}
```

### Gateway Routing (Bus-to-Bus)

**Incorrect (gateway with no buffer, blocking fast bus with slow bus):**

```c
/*
 * PduRRoutingPath: GW_CAN_to_LIN
 *   PduRSrcPdu: CanIf RxPdu (500 kbit/s CAN)
 *   PduRDestPdu: LinIf TxPdu (19.2 kbit/s LIN)
 *   NO buffering configured — PduR drops PDUs when LIN busy
 */
```

**Correct (gateway with FIFO buffering and flow management):**

```c
/*
 * PduRRoutingPath: GW_CAN_to_LIN
 *   PduRSrcPdu: CanIf RxPdu
 *   PduRDestPdu: LinIf TxPdu
 *   PduRQueueDepth: 4                     -- Buffer speed mismatch
 *   PduRDefaultValue: [0x00, ...]         -- Init/fallback value
 *   PduRTransmissionConfirmation: TRUE     -- Flow control
 *
 * 1:N fanout also supported:
 *   PduRDestPdu[0]: FrIf_TxPdu_Mirror     -- Mirror to FlexRay
 *   PduRDestPdu[1]: SoAd_TxPdu_Log        -- Mirror to Ethernet logger
 */
```

### Transport Protocol Routing

**Incorrect (large PDU sent via IF path instead of TP path):**

```c
/* Routing a 512-byte diagnostic response through CanIf (8-byte max) */
/*
 * PduRRoutingPath: Route_DiagResp
 *   PduRSrcPdu: Dcm TxPdu (512 bytes)
 *   PduRDestModule: CanIf              <-- WRONG: IF module, no segmentation
 */
```

**Correct (use TP path for multi-frame PDUs):**

```c
/*
 * PduRRoutingPath: Route_DiagResp
 *   PduRSrcPdu: Dcm TxPdu (up to 4095 bytes)
 *   PduRDestModule: CanTp              <-- Correct: TP handles segmentation
 *   PduRDestPdu: CanTp_TxNSdu_DiagResp
 *
 * CanTp segments into First Frame + Consecutive Frames
 * Flow Control from tester manages block size and timing
 */

/* For gateway TP routing (e.g., DoIP to CAN diagnostics): */
/*
 * PduRRoutingPath: GW_DoIP_to_CAN_Diag
 *   PduRSrcModule: SoAd (DoIP)   -- TP source
 *   PduRDestModule: CanTp         -- TP destination
 *   PduR handles buffer management between different TP speeds
 *   PduRTpBufferSize: 4096
 */
```

### Minimum Delay Between Transmissions

```c
/*
 * PduRMinimumDelay: Prevents bus overload from rapid retransmissions
 *
 * PduRRoutingPath: Route_Periodic_100ms
 *   PduRMinimumDelay: 8ms  -- Suppress retransmits within 8ms
 *
 * Critical for CAN buses: prevents Tx queue overflow when
 * COM transmission mode triggers faster than bus can absorb
 */
```

Reference: AUTOSAR Classic R22-11 SWS_PduR (SWS PDU Router), SRS_PduR
