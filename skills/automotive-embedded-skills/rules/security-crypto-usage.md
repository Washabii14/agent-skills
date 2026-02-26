---
title: Correct Cryptographic Primitive Usage
impact: CRITICAL
impactDescription: Prevents weak crypto by mandating approved algorithms
tags: security, crypto, aes, sha, hmac, cmac, csm, iso-21434
---

## Correct Cryptographic Primitive Usage

Use approved algorithms (AES-128/256, SHA-256, HMAC-SHA256, CMAC) and never implement custom cryptography. Custom or obsolete algorithms (MD5, SHA-1, DES, RC4) are vulnerable to known attacks.

**Incorrect (custom MAC implementation):**

```c
uint32_t CustomMac(const uint8_t *data, uint32_t len)
{
    uint32_t mac = 0xDEADBEEFU;
    for (uint32_t i = 0U; i < len; i++)
    {
        mac ^= ((uint32_t)data[i] << ((i % 4U) * 8U));
    }
    return mac;
}
```

**Correct (AUTOSAR CSM interface with approved algorithm):**

```c
/* Use standard AUTOSAR CSM (Crypto Service Manager) interface */
Std_ReturnType ComputeMessageMac(const uint8_t *data, uint32_t dataLen,
                                   uint8_t *mac, uint32_t *macLen)
{
    Csm_ReturnType ret;

    ret = Csm_MacGenerate(CSM_KEY_ID_MSG_AUTH,
                           CRYPTO_OPERATIONMODE_SINGLECALL,
                           data, dataLen,
                           mac, macLen);

    if (ret != CSM_E_OK)
    {
        ReportDtc(DTC_CRYPTO_MAC_FAILURE);
        return E_NOT_OK;
    }
    return E_OK;
}
```

Use the AUTOSAR Crypto Service Manager (CSM) or platform HSM API for all cryptographic operations. This ensures approved algorithms, constant-time execution, and key material isolation.

Reference: ISO/SAE 21434:2021, AUTOSAR SWS Crypto Service Manager
