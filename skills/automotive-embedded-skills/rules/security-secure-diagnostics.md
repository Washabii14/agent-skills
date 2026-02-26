---
title: Secure UDS Authentication
impact: HIGH
impactDescription: Prevents unauthorized diagnostic access to ECU functions
tags: security, uds, diagnostics, authentication, iso-14229, iso-21434
---

## Secure UDS Authentication

Implement UDS Authentication service (0x29) for diagnostic tool verification before allowing security-sensitive operations. Without authentication, any device on the bus can unlock reprogramming, calibration, or safety-critical parameter modification.

**Incorrect (no authentication before security-sensitive DID write):**

```c
void UDS_WriteDataByIdentifier(const uint8_t *request, uint16_t len)
{
    uint16_t did = ((uint16_t)request[1] << 8U) | request[2];
    DID_WriteData(did, &request[3], len - 3U);
    SendPositiveResponse(SID_WRITE_DATA);
}
```

**Correct (security access required before write):**

```c
void UDS_WriteDataByIdentifier(const uint8_t *request, uint16_t len)
{
    uint16_t did = ((uint16_t)request[1] << 8U) | request[2];

    if (!UDS_IsSessionAllowed(did, g_currentSession))
    {
        SendNegativeResponse(SID_WRITE_DATA, NRC_REQUEST_OUT_OF_RANGE);
        return;
    }

    if (!UDS_IsSecurityUnlocked(did))
    {
        SendNegativeResponse(SID_WRITE_DATA, NRC_SECURITY_ACCESS_DENIED);
        return;
    }

    Std_ReturnType result = DID_WriteData(did, &request[3], len - 3U);
    if (result != E_OK)
    {
        SendNegativeResponse(SID_WRITE_DATA, NRC_CONDITIONS_NOT_CORRECT);
        return;
    }

    SendPositiveResponse(SID_WRITE_DATA);
}
```

Implement seed-key challenge-response (Service 0x27) or PKI-based authentication (Service 0x29) with brute-force protection (delay timer, attempt counter).

Reference: ISO 14229-1:2020 (UDS), ISO/SAE 21434:2021
