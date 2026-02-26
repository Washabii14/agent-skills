---
title: CAN Interface and Transport Protocol — Callbacks, Timing, and Flow Control
impact: MEDIUM
impactDescription: Incorrect CanIf/CanTp configuration causes missed frames, diagnostic timeouts, or multi-frame transfer failures
tags: autosar, classic, canif, cantp, can, transport-protocol, flow-control, n-pdu, timing, iso15765
---

## CAN Interface and Transport Protocol — Callbacks, Timing, and Flow Control

CanIf (CAN Interface) provides hardware-independent access to CAN controllers and transceivers. CanTp (CAN Transport Protocol, ISO 15765-2) handles segmentation and reassembly of messages larger than a single CAN frame. In R22-11, both are critical for diagnostic communication and gateway routing.

### CanIf — Tx Confirmation and Rx Indication

**Incorrect (polling for CAN transmission without using callbacks):**

```c
void App_SendCanMsg(void)
{
    PduInfoType pdu = { .SduDataPtr = data, .SduLength = 8u };
    CanIf_Transmit(CANIF_TX_PDU_ID, &pdu);

    /* Busy-wait for confirmation — blocks task, wastes CPU */
    while (!g_txConfirmed) {}
}
```

**Correct (asynchronous Tx confirmation callback):**

```c
/* CanIf calls upper layer on successful transmission */
void PduR_CanIfTxConfirmation(PduIdType TxPduId)
{
    /* PduR routes confirmation to COM/DCM as configured */
    /* COM uses this to manage transmission mode retry logic */
}

/* Rx indication — CanIf dispatches received PDU to upper layer */
void PduR_CanIfRxIndication(PduIdType RxPduId, const PduInfoType *PduInfoPtr)
{
    /* PduR routes to:
     *   - COM (for signal I-PDUs)
     *   - CanTp (for diagnostic/TP N-PDUs based on N-SDU addressing)
     *   - XCP, J1939Tp, etc. per routing table
     */
}

/* Configure CanIf Rx PDU filtering: */
/*
 * CanIfRxPduCfg:
 *   CanIfRxPduId: 0x7DF              -- Functional diag request
 *   CanIfRxPduCanIdType: STANDARD
 *   CanIfRxPduUpperLayerRef: CanTp    -- Route to TP layer
 *   CanIfRxPduHrhRef: HRH_CAN0       -- Hardware receive handle
 */
```

### CanTp — N-PDU Timing Parameters

**Incorrect (no timeout configuration — transfer can hang indefinitely):**

```c
/* CanTp N-SDU with no timing limits
 * CanTpRxNSdu: NSdu_DiagReq
 *   CanTpRxNSduId: 0x7DF
 *   -- No N_Ar, N_Br, N_Cr specified — defaults may be too long
 */
```

**Correct (proper ISO 15765-2 timing per bus speed and tester requirements):**

```c
/* CanTp timing parameters (all in seconds):
 *
 * CanTpRxNSdu: NSdu_DiagReq
 *   N_Ar: 0.025   -- Time for receiver to transmit Flow Control
 *   N_Br: 0.010   -- Receiver processing before sending FC
 *   N_Cr: 0.150   -- Max time between Consecutive Frames from sender
 *
 * CanTpTxNSdu: NSdu_DiagResp
 *   N_As: 0.025   -- Time for sender to transmit any N-PDU
 *   N_Bs: 0.150   -- Max wait for Flow Control from receiver
 *   N_Cs: 0.010   -- Sender processing between Consecutive Frames
 *
 * Timeout triggers:
 *   N_Cr timeout -> CanTp aborts reception, notifies PduR
 *   N_Bs timeout -> CanTp aborts transmission, notifies PduR
 */
```

### Flow Control Handling

**Incorrect (BlockSize=0 with insufficient Rx buffer):**

```c
/* CanTp FC parameters:
 *   CanTpBs (BlockSize) = 0      -- Sender transmits ALL CFs without pause
 *   CanTpRxWftMax = 0            -- No wait frames allowed
 *   CanTpRxBufSize = 64          -- Only 64 bytes buffer
 *
 * Problem: 4095-byte diagnostic request overflows 64-byte buffer
 */
```

**Correct (BlockSize matched to available buffer, STmin for bus load):**

```c
/*
 * CanTpRxNSdu: NSdu_DiagReq
 *   CanTpBs: 8               -- Request FC every 8 CFs (56 bytes on classic CAN)
 *   CanTpSTmin: 5             -- 5ms minimum separation between CFs
 *   CanTpRxWftMax: 10         -- Allow up to 10 Wait frames if buffer temporarily full
 *   CanTpRxBufSize: 4096      -- Sufficient for largest expected request
 *   CanTpRxPaddingActivation: ON
 *   CanTpRxPaddingByte: 0xCC
 *
 * Flow Control frame format (FC):
 *   Byte 0: 0x30 | FS         -- FS=0 ContinueToSend, FS=1 Wait, FS=2 Overflow
 *   Byte 1: BS (BlockSize)
 *   Byte 2: STmin
 */

/* For CAN FD with 64-byte payload: */
/*
 * CanTpBs: 4               -- Fewer blocks needed (256 bytes per block)
 * CanTpSTmin: 2             -- Shorter gap — CAN FD is faster
 * CanTpNSduLength: 4095     -- Max ISO 15765-2 PDU
 */
```

### Error Notification

```c
/* CanTp notifies PduR on transfer failure */
void PduR_CanTpRxIndication(PduIdType RxPduId, Std_ReturnType Result)
{
    if (Result != E_OK)
    {
        /* Transfer failed — possible causes:
         *   - N_Cr timeout (sender stopped sending CFs)
         *   - FC Overflow (receiver buffer full)
         *   - Invalid sequence number
         *   - Unexpected PDU type
         */
        Dem_ReportErrorStatus(DTC_CANTP_RX_FAILURE, DEM_EVENT_STATUS_FAILED);
    }
    else
    {
        /* Complete N-SDU received — route to Dcm for processing */
    }
}

void PduR_CanTpTxConfirmation(PduIdType TxPduId, Std_ReturnType Result)
{
    if (Result != E_OK)
    {
        /* Transmission failed — N_Bs timeout or CAN bus error */
    }
}
```

Reference: AUTOSAR Classic R22-11 SWS_CanIf, SWS_CanTp; ISO 15765-2 (CAN Transport Protocol)
