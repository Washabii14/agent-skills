---
title: Secure Boot Chain of Trust
impact: CRITICAL
impactDescription: A broken chain of trust allows execution of tampered firmware, compromising vehicle safety and security
tags: boot, secure-boot, hsm, signature-verification, chain-of-trust, rollback-protection, measurement-log, crypto
---

## Secure Boot Chain of Trust

Secure boot establishes a hardware-rooted chain of trust: ROM bootloader (immutable) → verify 1st-stage bootloader → verify 2nd-stage bootloader → verify application. Each stage cryptographically verifies the next before transferring control. HSM-based signature verification, anti-rollback counters, and boot measurement logs are required for automotive security.

**Incorrect (no signature verification, trusting flash content directly):**

```c
/* WRONG: Jumping to next boot stage without any verification */
typedef void (*EntryPoint_t)(void);

void Bootloader_JumpToApp(void)
{
    uint32_t app_addr = *(volatile uint32_t *)APP_START_ADDR;
    EntryPoint_t app_entry = (EntryPoint_t)(*(volatile uint32_t *)(APP_START_ADDR + 4));

    __set_MSP(app_addr);    /* Set stack pointer */
    app_entry();            /* Jump — no integrity check, attacker can run arbitrary code */
}
```

**Correct (HSM-verified secure boot chain):**

```c
/* secure_boot.h — boot stage descriptor */
typedef struct {
    uint32_t    loadAddr;
    uint32_t    size;
    uint32_t    entryPoint;
    uint32_t    version;       /* Anti-rollback version counter */
    uint8_t     signature[256]; /* RSA-2048 or ECDSA-P256 signature */
    uint8_t     hash[32];      /* SHA-256 digest of payload */
} BootImageHeader_t;

typedef enum {
    BOOT_VERIFY_OK,
    BOOT_VERIFY_HASH_FAIL,
    BOOT_VERIFY_SIG_FAIL,
    BOOT_VERIFY_ROLLBACK,
    BOOT_VERIFY_HSM_ERROR
} BootVerifyResult_t;
```

```c
/* secure_boot.c — verification using HSM */
#include "hsm_driver.h"

static BootVerifyResult_t SecureBoot_VerifyImage(const BootImageHeader_t *header)
{
    /* Step 1: Anti-rollback check — version must be >= stored minimum */
    uint32_t minVersion;
    if (HSM_ReadMonotonicCounter(COUNTER_FW_VERSION, &minVersion) != HSM_OK)
    {
        return BOOT_VERIFY_HSM_ERROR;
    }
    if (header->version < minVersion)
    {
        return BOOT_VERIFY_ROLLBACK;
    }

    /* Step 2: Compute hash of image payload */
    uint8_t computedHash[32];
    if (HSM_SHA256((const uint8_t *)header->loadAddr, header->size,
                   computedHash) != HSM_OK)
    {
        return BOOT_VERIFY_HSM_ERROR;
    }

    /* Step 3: Verify hash matches header (defense in depth) */
    if (Crypto_SecureCompare(computedHash, header->hash, 32) != 0)
    {
        return BOOT_VERIFY_HASH_FAIL;
    }

    /* Step 4: Verify signature over hash using HSM-stored public key */
    if (HSM_VerifySignature(HSM_KEY_SLOT_BOOT_PUB,
                            HSM_ALG_ECDSA_P256_SHA256,
                            computedHash, 32,
                            header->signature,
                            sizeof(header->signature)) != HSM_OK)
    {
        return BOOT_VERIFY_SIG_FAIL;
    }

    /* Step 5: Log measurement for boot attestation */
    SecureBoot_ExtendMeasurementLog(header->loadAddr, header->size,
                                    computedHash);

    return BOOT_VERIFY_OK;
}

void SecureBoot_ChainLoad(void)
{
    /* ROM bootloader (this code) is immutable — root of trust */

    /* Stage 1: Verify and load 1st-stage bootloader */
    const BootImageHeader_t *stage1 =
        (const BootImageHeader_t *)STAGE1_HEADER_ADDR;

    BootVerifyResult_t result = SecureBoot_VerifyImage(stage1);
    if (result != BOOT_VERIFY_OK)
    {
        SecureBoot_EnterRecovery(result);  /* Safe state — do NOT proceed */
        return;
    }

    /* Update anti-rollback counter on successful verification */
    HSM_IncrementMonotonicCounter(COUNTER_FW_VERSION, stage1->version);

    /* Transfer control to verified stage 1 */
    SecureBoot_JumpToVerified(stage1->entryPoint);
}
```

**Boot measurement log for attestation:**

```c
/* boot_measurement.c — TPM-style measurement log */
#define MAX_MEASUREMENTS 8

typedef struct {
    uint32_t stageId;
    uint32_t loadAddr;
    uint32_t size;
    uint8_t  hash[32];
} BootMeasurement_t;

static BootMeasurement_t g_measurementLog[MAX_MEASUREMENTS];
static uint32_t g_measurementCount = 0;

/* PCR-style extend: new_hash = SHA256(old_hash || measurement_hash) */
static uint8_t g_extendedPCR[32] = {0};

void SecureBoot_ExtendMeasurementLog(uint32_t addr, uint32_t size,
                                      const uint8_t *hash)
{
    if (g_measurementCount >= MAX_MEASUREMENTS) { return; }

    BootMeasurement_t *entry = &g_measurementLog[g_measurementCount++];
    entry->stageId = g_measurementCount;
    entry->loadAddr = addr;
    entry->size = size;
    (void)memcpy(entry->hash, hash, 32);

    /* Extend PCR: H(PCR || hash) */
    uint8_t concat[64];
    (void)memcpy(concat, g_extendedPCR, 32);
    (void)memcpy(&concat[32], hash, 32);
    HSM_SHA256(concat, 64, g_extendedPCR);
}
```

**C++ wrapper for Adaptive AUTOSAR platforms:**

```cpp
// secure_boot_manager.cpp — Adaptive platform secure boot integration
class SecureBootManager
{
public:
    bool VerifyAndLoad(const BootStage& stage)
    {
        auto hash = hsm_.ComputeHash(stage.PayloadData(), stage.PayloadSize());
        if (!hash) { return false; }

        if (!hsm_.VerifySignature(stage.PublicKeySlot(),
                                   *hash, stage.Signature()))
        {
            ara::log::CreateLogger("SBOOT", "SecureBoot")
                .LogError() << "Signature verification failed for stage "
                            << stage.Name();
            return false;
        }

        measurement_log_.Extend(stage.Id(), *hash);
        return true;
    }

private:
    HsmInterface hsm_;
    BootMeasurementLog measurement_log_;
};
```

Use constant-time comparison (`Crypto_SecureCompare`) for hash verification to prevent timing side-channels. The HSM private key must never leave the hardware security module. Anti-rollback counters must be stored in one-time-programmable (OTP) memory or HSM-protected non-volatile storage.

Reference: ISO 26262 Part 6 (Software safety — startup integrity); UNECE WP.29 R155/R156 (Cybersecurity & Software Update); NIST SP 800-193 (Platform Firmware Resiliency)
