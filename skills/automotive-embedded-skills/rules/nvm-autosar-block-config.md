---
title: AUTOSAR NvM Block Configuration
impact: HIGH
impactDescription: Misconfigured NvM blocks cause data loss on power failure, corrupted safety-critical calibration, or excessive startup time
tags: nvm, autosar, flash, eeprom, persistent-storage, crc, ram-mirror, safety, redundant-block
---

## AUTOSAR NvM Block Configuration

AUTOSAR NvM supports three block types: Native (single copy), Redundant (dual copy with automatic failover), and Dataset (array of equal-sized data sets). Correct configuration of block CRC, RAM mirroring, and immediate write for safety data is essential. `NvM_ReadAll` at startup and `NvM_WriteAll` at shutdown must fit within timing budgets.

**Incorrect (safety data in native block, no CRC, no immediate write):**

```c
/* WRONG: Safety-critical data uses a native block without CRC */
/* NvM Configuration — unsafe */
const NvM_BlockDescriptor_t NvM_BlockDescriptors[] = {
    [NVM_BLOCK_SAFETY_PARAMS] = {
        .blockType       = NVM_BLOCK_NATIVE,    /* Single copy — no redundancy */
        .blockCrcType    = NVM_CRC_NONE,        /* No integrity check */
        .ramBlockDataAddr = &SafetyParams_Ram,
        .romDefaultAddr  = NULL,                 /* No default — undefined on first boot */
        .writeProtection = FALSE,
        .resistantToChange = FALSE,
        .immediateWrite  = FALSE,                /* Deferred — may be lost on power cut */
    },
};
```

**Correct (redundant block with CRC for safety data, native with CRC for non-safety):**

```c
/* NvM block type selection guide:
 * - Native:    Non-safety data, single copy, CRC recommended
 * - Redundant: ASIL-rated data, two copies with automatic failover
 * - Dataset:   Multiple data sets (e.g., user profiles, variant configs)
 */

/* RAM mirrors */
static SafetyParams_t    SafetyParams_Ram;
static CalibrationData_t CalData_Ram;
static uint8_t           VariantCfg_Ram[VARIANT_CFG_SIZE];

/* ROM defaults — used when NvM block is empty or corrupted */
static const SafetyParams_t SafetyParams_Default = {
    .maxTorque   = 100U,
    .minTemp     = -40,
    .maxTemp     = 150,
    .checksumSeed = 0xA5A5U
};

const NvM_BlockDescriptor_t NvM_BlockDescriptors[] = {
    /* Redundant block for ASIL data — dual copy, CRC-32, immediate write */
    [NVM_BLOCK_SAFETY_PARAMS] = {
        .blockType        = NVM_BLOCK_REDUNDANT,
        .blockCrcType     = NVM_CRC32,
        .blockLength      = sizeof(SafetyParams_t),
        .ramBlockDataAddr = &SafetyParams_Ram,
        .romDefaultAddr   = &SafetyParams_Default,
        .writeProtection  = FALSE,
        .resistantToChange = TRUE,      /* Survives NvM_InvalidateAll */
        .immediateWrite   = TRUE,       /* Written immediately, not deferred */
        .writeVerification = TRUE,      /* Read-back verify after write */
        .calcRamBlockCrc  = TRUE,       /* Detect RAM corruption between writes */
    },

    /* Native block for calibration — single copy, CRC-16, deferred write OK */
    [NVM_BLOCK_CALIBRATION] = {
        .blockType        = NVM_BLOCK_NATIVE,
        .blockCrcType     = NVM_CRC16,
        .blockLength      = sizeof(CalibrationData_t),
        .ramBlockDataAddr = &CalData_Ram,
        .romDefaultAddr   = &CalData_Default,
        .writeProtection  = FALSE,
        .immediateWrite   = FALSE,
        .calcRamBlockCrc  = TRUE,
    },

    /* Dataset block for variant configurations (e.g., 4 variants) */
    [NVM_BLOCK_VARIANT_CFG] = {
        .blockType        = NVM_BLOCK_DATASET,
        .blockCrcType     = NVM_CRC16,
        .blockLength      = VARIANT_CFG_SIZE,
        .numDatasets      = 4U,
        .ramBlockDataAddr = &VariantCfg_Ram,
        .romDefaultAddr   = NULL,
    },
};
```

**NvM_ReadAll at startup — timing management:**

```c
/* Startup: NvM_ReadAll must complete before SWCs access NvM data */
void EcuM_StartupTwo(void)
{
    NvM_Init(&NvM_Config);
    NvM_ReadAll();  /* Asynchronous — queues all blocks for reading */
}

/* BswM rule: gate SWC access on NvM_ReadAll completion */
void BswM_Rule_NvMReadAllComplete(void)
{
    NvM_RequestResultType result;
    NvM_GetErrorStatus(NVM_BLOCK_MULTI, &result);

    if (result == NVM_REQ_OK || result == NVM_REQ_RESTORED_FROM_ROM)
    {
        /* All blocks loaded — allow SWC initialization */
        BswM_RequestMode(BSWM_NVM_READALL_COMPLETE, TRUE);
    }
    else if (result == NVM_REQ_PENDING)
    {
        /* Still loading — check again next cycle */
    }
    else
    {
        /* Error — some blocks failed, use defaults */
        BswM_RequestMode(BSWM_NVM_READALL_DEGRADED, TRUE);
    }
}
```

**NvM_WriteAll at shutdown — ensuring data persistence:**

```c
/* Shutdown: NvM_WriteAll must complete before power is cut */
void EcuM_OnGoOffTwo(void)
{
    NvM_WriteAll();  /* Queue all modified blocks */

    /* Wait for completion within timing budget */
    NvM_RequestResultType result;
    uint32_t timeout = NVM_WRITEALL_TIMEOUT_MS;

    do {
        NvM_MainFunction();    /* Drive NvM state machine */
        Fee_MainFunction();    /* Drive Fee state machine */
        Fls_MainFunction();    /* Drive flash driver */

        NvM_GetErrorStatus(NVM_BLOCK_MULTI, &result);
        timeout--;
    } while ((result == NVM_REQ_PENDING) && (timeout > 0U));

    if (result != NVM_REQ_OK)
    {
        /* Log error — some blocks may not have been written */
        Dem_ReportErrorStatus(DEM_EVENT_NVM_WRITEALL_FAILED,
                              DEM_EVENT_STATUS_FAILED);
    }
}
```

RAM block CRC (`calcRamBlockCrc`) detects RAM corruption between write cycles — essential for ASIL applications. Immediate-write blocks bypass the queue and write through Fee/Fls synchronously. Time `NvM_ReadAll` on your target hardware and ensure it fits within the startup time budget (typically < 500ms).

Reference: AUTOSAR SWS_NvM (NvM Block Management); ISO 26262 Part 6 (Data integrity for safety-related elements)
