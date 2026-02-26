---
title: Secure Boot Chain Verification
impact: CRITICAL
impactDescription: Prevents unauthorized firmware execution via cryptographic verification
tags: security, secure-boot, firmware, signature, iso-21434, crypto
---

## Secure Boot Chain Verification

Verify firmware integrity at every stage of the boot chain using cryptographic signatures. Each boot stage must verify the next before transferring control, forming an unbroken chain of trust from ROM bootloader to application.

**Incorrect (no boot verification):**

```c
void Bootloader_Start(void)
{
    void (*appEntry)(void) = (void (*)(void))APP_START_ADDR;
    appEntry();
}
```

**Correct (signature-verified boot chain):**

```c
typedef struct
{
    uint32_t firmwareAddr;
    uint32_t firmwareSize;
    uint8_t  signature[256];
    uint16_t signatureLen;
    uint8_t  publicKeyHash[32];
} SecureBoot_Entry_t;

Std_ReturnType SecureBoot_VerifyStage(const SecureBoot_Entry_t *entry)
{
    uint8_t computedHash[32];
    Crypto_HashSha256((const uint8_t *)entry->firmwareAddr,
                      entry->firmwareSize, computedHash);

    if (Crypto_VerifySignatureRsa(computedHash, sizeof(computedHash),
                                   entry->signature, entry->signatureLen,
                                   entry->publicKeyHash) != CRYPTO_OK)
    {
        ReportDtc(DTC_SECURE_BOOT_FAILURE);
        EnterSafeState();
        return E_NOT_OK;
    }

    return E_OK;
}
```

If verification fails, the ECU must not execute the unverified firmware. Enter safe state and report a DTC for workshop diagnosis.

Reference: ISO/SAE 21434:2021 — Road vehicles — Cybersecurity engineering
