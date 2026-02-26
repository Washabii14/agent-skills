---
title: CAN FD Extended Data Handling
impact: MEDIUM
impactDescription: correct next-gen CAN communication
tags: comm, can-fd, dlc, data-length, iso-11898
---

## CAN FD Extended Data Handling

CAN FD supports up to 64 bytes per frame (vs. 8 for Classic CAN) and higher bit rates in the data phase. Handle DLC-to-length mapping correctly since CAN FD DLC values above 8 are non-linear (12->16, 13->20, 14->24, 15->32, etc.).

**Incorrect (assuming DLC == data length):**

```c
void ProcessCanFdFrame(const CanFd_Frame_t *frame)
{
    uint8_t buffer[64];
    (void)memcpy(buffer, frame->data, frame->dlc);  /* DLC 15 != 15 bytes */
}
```

**Correct (DLC-to-length conversion):**

```c
static const uint8_t g_canFdDlcToLen[] =
{
    0U, 1U, 2U, 3U, 4U, 5U, 6U, 7U, 8U,
    12U, 16U, 20U, 24U, 32U, 48U, 64U
};

uint8_t CanFd_DlcToLength(uint8_t dlc)
{
    if (dlc > 15U) { return 64U; }
    return g_canFdDlcToLen[dlc];
}

void ProcessCanFdFrame(const CanFd_Frame_t *frame)
{
    uint8_t dataLen = CanFd_DlcToLength(frame->dlc);
    uint8_t buffer[64];
    (void)memcpy(buffer, frame->data, dataLen);
    ProcessPayload(buffer, dataLen);
}
```

Always use the conversion table. Treating DLC as byte count for values >8 causes buffer overread or underread.

Reference: ISO 11898-1:2015 — CAN FD data link layer
