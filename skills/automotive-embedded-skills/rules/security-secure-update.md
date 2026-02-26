---
title: Secure OTA/Reflash with Signature Verification
impact: CRITICAL
impactDescription: Prevents malicious firmware installation via OTA or workshop reflash
tags: security, ota, reflash, signature, firmware, iso-21434
---

## Secure OTA/Reflash with Signature Verification

All firmware updates must be cryptographically signed and verified before flashing. Whether delivered over-the-air (OTA) or via workshop diagnostic tool, unsigned firmware must be rejected to prevent malicious code execution.

**Incorrect (no signature check before flash):**

```c
void Bootloader_FlashFirmware(const uint8_t *data, uint32_t size)
{
    Flash_Erase(APP_START_ADDR, size);
    Flash_Write(APP_START_ADDR, data, size);
    Bootloader_Reset();
}
```

**Correct (verify signature before flashing):**

```c
Std_ReturnType Bootloader_FlashFirmware(const uint8_t *data, uint32_t size,
                                          const uint8_t *signature,
                                          uint16_t sigLen)
{
    uint8_t hash[32];
    Crypto_HashSha256(data, size, hash);

    if (Crypto_VerifySignatureRsa(hash, sizeof(hash),
                                   signature, sigLen,
                                   g_otaPublicKeyHash) != CRYPTO_OK)
    {
        ReportDtc(DTC_OTA_SIGNATURE_INVALID);
        return E_NOT_OK;
    }

    /* Version rollback protection */
    if (Fw_GetVersion(data) <= Fw_GetCurrentVersion())
    {
        ReportDtc(DTC_OTA_ROLLBACK_ATTEMPT);
        return E_NOT_OK;
    }

    Flash_Erase(APP_START_ADDR, size);
    Flash_Write(APP_START_ADDR, data, size);

    /* Verify written data matches source */
    if (memcmp((const void *)APP_START_ADDR, data, size) != 0)
    {
        ReportDtc(DTC_FLASH_VERIFY_FAILURE);
        return E_NOT_OK;
    }

    return E_OK;
}
```

Implement anti-rollback protection (version monotonic counter) and post-flash verification (read-back compare or CRC) to ensure integrity of the flashed image.

Reference: ISO/SAE 21434:2021 — Road vehicles — Cybersecurity engineering
