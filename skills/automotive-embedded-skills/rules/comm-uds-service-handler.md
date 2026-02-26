---
title: UDS Diagnostic Service Handling
impact: HIGH
impactDescription: diagnostics compliance per ISO 14229
tags: comm, uds, diagnostics, iso-14229, nrc, did, service-handler
---

## UDS Diagnostic Service Handling

Implement diagnostic services with proper NRC (Negative Response Code) handling per ISO 14229. Every service handler must check message length, session validity, and security access before processing the request.

**Incorrect (no validation, no NRC handling):**

```c
void UDS_ReadData(const uint8_t *request, uint16_t len,
                  uint8_t *response, uint16_t *respLen)
{
    uint16_t did = (request[1] << 8) | request[2];
    DID_ReadData(did, &response[3], respLen);
    response[0] = 0x62U;
    response[1] = request[1];
    response[2] = request[2];
}
```

**Correct (full NRC-compliant service handler):**

```c
void UDS_ReadDataByIdentifier(const uint8_t *request, uint16_t len,
                               uint8_t *response, uint16_t *respLen)
{
    uint16_t did;

    if (len < 3U)
    {
        SendNegativeResponse(SID_READ_DATA, NRC_INCORRECT_MSG_LENGTH);
        return;
    }

    did = ((uint16_t)request[1] << 8U) | request[2];

    if (!UDS_IsSessionAllowed(did, g_currentSession))
    {
        SendNegativeResponse(SID_READ_DATA, NRC_REQUEST_OUT_OF_RANGE);
        return;
    }

    if (!UDS_IsSecurityUnlocked(did))
    {
        SendNegativeResponse(SID_READ_DATA, NRC_SECURITY_ACCESS_DENIED);
        return;
    }

    Std_ReturnType result = DID_ReadData(did, &response[3], respLen);
    if (result != E_OK)
    {
        SendNegativeResponse(SID_READ_DATA, NRC_CONDITIONS_NOT_CORRECT);
        return;
    }

    response[0] = SID_READ_DATA + 0x40U;
    response[1] = request[1];
    response[2] = request[2];
    *respLen += 3U;
}
```

Standard NRC check order: message length -> service support -> sub-function support -> session check -> security check -> conditions check -> request data validation.

Reference: ISO 14229-1:2020 (UDS — Unified Diagnostic Services)
