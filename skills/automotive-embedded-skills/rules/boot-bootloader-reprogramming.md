---
title: UDS Bootloader Reprogramming Sequence
impact: HIGH
impactDescription: Incorrect flash sequence bricks the ECU or leaves it in an unbootable state requiring physical intervention
tags: boot, bootloader, uds, flash, reprogramming, security-access, diagnostic, fota
---

## UDS Bootloader Reprogramming Sequence

The UDS flash reprogramming sequence follows a strict service order: `0x10` (DiagnosticSessionControl → Programming) → `0x27` (SecurityAccess) → `0x34` (RequestDownload) → `0x36` (TransferData) → `0x37` (RequestTransferExit) → `0x31` (RoutineControl for erase/verify). Skipping steps or incorrect ordering can brick the ECU.

**Incorrect (missing security access, no erase before write, no verification):**

```c
/* WRONG: Skipping critical steps in flash sequence */
void FlashUpdate_BadSequence(void)
{
    UDS_DiagSessionControl(0x02);     /* Programming session */
    /* Missing: SecurityAccess — flash writes will be rejected */
    /* Missing: Erase before write — corrupted flash */
    UDS_RequestDownload(0x00044000, 0x10000);
    UDS_TransferData(1, firmware_data, sizeof(firmware_data));
    UDS_RequestTransferExit();
    /* Missing: CRC/signature verification — may boot corrupted image */
    UDS_EcuReset(0x01);               /* Hard reset into potentially bad firmware */
}
```

**Correct (full UDS flash sequence with all safety steps):**

```c
/* Complete UDS flash reprogramming — tester/client side */
typedef enum {
    FLASH_OK = 0,
    FLASH_ERR_SESSION,
    FLASH_ERR_SECURITY,
    FLASH_ERR_ERASE,
    FLASH_ERR_DOWNLOAD,
    FLASH_ERR_TRANSFER,
    FLASH_ERR_VERIFY,
    FLASH_ERR_DEPENDENCY
} FlashResult_t;

FlashResult_t FlashUpdate_Execute(const uint8_t *fw_data, uint32_t fw_size,
                                   uint32_t target_addr)
{
    UDS_Response_t resp;

    /* Step 0: Pre-conditions — disable normal communication */
    resp = UDS_ControlDTCSetting(0x02);         /* 0x85: DTC off */
    resp = UDS_CommunicationControl(0x03, 0x01); /* 0x28: disable Tx/Rx */

    /* Step 1: Enter programming session (0x10 02) */
    resp = UDS_DiagSessionControl(0x02);
    if (resp.nrc != NRC_OK) { return FLASH_ERR_SESSION; }

    /* Step 2: Security access — seed/key exchange (0x27) */
    uint8_t seed[8];
    resp = UDS_SecurityAccess_RequestSeed(0x01, seed, sizeof(seed));
    if (resp.nrc != NRC_OK) { return FLASH_ERR_SECURITY; }

    uint8_t key[8];
    SecurityAlgo_ComputeKey(seed, key);
    resp = UDS_SecurityAccess_SendKey(0x02, key, sizeof(key));
    if (resp.nrc != NRC_OK) { return FLASH_ERR_SECURITY; }

    /* Step 3: Check programming pre-conditions (0x31 — RoutineControl) */
    resp = UDS_RoutineControl_Start(ROUTINE_CHECK_PRECONDITIONS);
    if (resp.nrc != NRC_OK) { return FLASH_ERR_DEPENDENCY; }

    /* Step 4: Erase target flash memory (0x31 — RoutineControl) */
    resp = UDS_RoutineControl_Start_WithParams(ROUTINE_ERASE_MEMORY,
                                                target_addr, fw_size);
    if (resp.nrc != NRC_OK) { return FLASH_ERR_ERASE; }

    /* Step 5: Request download (0x34) */
    uint32_t block_size;
    resp = UDS_RequestDownload(0x00,                /* compression: none */
                               0x44,                /* addr=4 bytes, len=4 bytes */
                               target_addr, fw_size,
                               &block_size);
    if (resp.nrc != NRC_OK) { return FLASH_ERR_DOWNLOAD; }

    /* Step 6: Transfer data in blocks (0x36) */
    uint8_t block_seq = 1;
    uint32_t offset = 0;
    while (offset < fw_size)
    {
        uint32_t chunk = (fw_size - offset > block_size)
                         ? block_size : (fw_size - offset);
        resp = UDS_TransferData(block_seq, &fw_data[offset], chunk);
        if (resp.nrc != NRC_OK) { return FLASH_ERR_TRANSFER; }
        block_seq++;  /* Wraps 0xFF → 0x00 per ISO 14229 */
        offset += chunk;

        WDG_Trigger();  /* Keep watchdog alive during long transfers */
    }

    /* Step 7: Request transfer exit (0x37) */
    resp = UDS_RequestTransferExit();
    if (resp.nrc != NRC_OK) { return FLASH_ERR_TRANSFER; }

    /* Step 8: Verify programmed data — CRC or signature (0x31) */
    resp = UDS_RoutineControl_Start(ROUTINE_CHECK_PROGRAMMING_INTEGRITY);
    if (resp.nrc != NRC_OK) { return FLASH_ERR_VERIFY; }

    /* Step 9: Reset ECU to boot into new firmware (0x11) */
    resp = UDS_EcuReset(0x01);  /* hardReset */

    return FLASH_OK;
}
```

**Bootloader-side request handler (ECU firmware):**

```c
/* bootloader_uds.c — ECU-side service handling */
static bool g_securityUnlocked = false;
static FlashState_t g_flashState = FLASH_IDLE;

void Bootloader_HandleRequestDownload(const UDS_Request_t *req)
{
    if (!g_securityUnlocked)
    {
        UDS_SendNegativeResponse(req->sid, NRC_SECURITY_ACCESS_DENIED);
        return;
    }

    uint32_t addr = ExtractAddress(req);
    uint32_t size = ExtractSize(req);

    if (!Flash_IsValidRange(addr, size))
    {
        UDS_SendNegativeResponse(req->sid, NRC_REQUEST_OUT_OF_RANGE);
        return;
    }

    g_flashState = FLASH_DOWNLOADING;
    g_expectedSize = size;
    g_targetAddr = addr;

    uint32_t maxBlock = FLASH_BUFFER_SIZE - 2;  /* minus SID + blockSeq */
    UDS_SendPositiveResponse_Download(maxBlock);
}
```

The block sequence counter in `TransferData` (0x36) increments from 1 and wraps from 0xFF to 0x00. Tester-present (0x3E) may be needed to keep the programming session alive during long erase/write operations.

Reference: ISO 14229-1 (UDS services); ISO 15765-2 (CAN transport layer); AUTOSAR SWS_FlashDriver
