---
title: XCP Measurement and Calibration Protocol
impact: MEDIUM
impactDescription: Enables real-time parameter tuning and signal measurement
tags: integration, xcp, measurement, calibration, protocol, canape, inca
---

## XCP Measurement and Calibration Protocol

Implement XCP (Universal Measurement and Calibration Protocol) for real-time parameter tuning and signal measurement. XCP enables calibration tools to read/write ECU variables at runtime without reflashing.

**Incorrect (no address validation in XCP handler):**

```c
Std_ReturnType Xcp_HandleUpload(const uint8_t *request,
                                  uint8_t *response, uint16_t *respLen)
{
    uint32_t addr = *(uint32_t *)&request[4];
    uint8_t len = request[1];
    memcpy(&response[1], (const void *)addr, len);
    *respLen = 1U + len;
    return E_OK;
}
```

**Correct (validated XCP short upload):**

```c
typedef struct
{
    uint32_t address;
    uint8_t  addressExtension;
    uint8_t  dataSize;
} Xcp_DaqEntry_t;

Std_ReturnType Xcp_HandleShortUpload(const uint8_t *request,
                                       uint8_t *response, uint16_t *respLen)
{
    uint8_t numBytes = request[1];
    uint32_t addr    = ((uint32_t)request[4] << 24U) |
                       ((uint32_t)request[5] << 16U) |
                       ((uint32_t)request[6] << 8U)  |
                       (uint32_t)request[7];

    if (numBytes > XCP_MAX_CTO_SIZE - 1U)
    {
        Xcp_SendError(XCP_ERR_OUT_OF_RANGE);
        return E_NOT_OK;
    }

    if (!Xcp_IsAddressReadable(addr, numBytes))
    {
        Xcp_SendError(XCP_ERR_ACCESS_DENIED);
        return E_NOT_OK;
    }

    response[0] = XCP_PID_RESPONSE;
    (void)memcpy(&response[1], (const void *)addr, numBytes);
    *respLen = 1U + numBytes;
    return E_OK;
}
```

Always validate XCP memory access against an allowed address range list. Unrestricted memory access via XCP is a security vulnerability — an attacker could read cryptographic keys or modify safety parameters.

Reference: ASAM XCP Protocol Layer Specification
