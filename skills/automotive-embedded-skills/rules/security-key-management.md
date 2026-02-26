---
title: Cryptographic Key Management
impact: CRITICAL
impactDescription: Protects secret material using secure hardware storage
tags: security, crypto, key-management, hsm, she, iso-21434
---

## Cryptographic Key Management

Store cryptographic keys in secure hardware (HSM/SHE) when available. Never store keys in plain text in Flash. Keys exposed in Flash can be extracted via debugger, flash dump, or side-channel attacks.

**Incorrect (key stored in plain text):**

```c
static const uint8_t g_aesKey[16] = {
    0x2B, 0x7E, 0x15, 0x16, 0x28, 0xAE, 0xD2, 0xA6,
    0xAB, 0xF7, 0x15, 0x88, 0x09, 0xCF, 0x4F, 0x3C
};
```

**Correct (key loaded from HSM):**

```c
Std_ReturnType Crypto_LoadKeyFromHsm(uint16_t keySlot,
                                       CryptoKey_t *key)
{
    if (Hsm_IsAvailable() != TRUE)
    {
        ReportDtc(DTC_HSM_NOT_AVAILABLE);
        return E_NOT_OK;
    }

    if (Hsm_ReadKey(keySlot, key->material, &key->length) != HSM_OK)
    {
        return E_NOT_OK;
    }

    key->isHsmBacked = TRUE;
    return E_OK;
}
```

Key management practices:
- Use HSM key slots for all symmetric and asymmetric keys
- Implement key derivation for session keys (avoid using master keys directly)
- Support key rotation via secure diagnostic services
- Zeroize key material in RAM after use

Reference: ISO/SAE 21434:2021, AUTOSAR SWS Crypto Service Manager
