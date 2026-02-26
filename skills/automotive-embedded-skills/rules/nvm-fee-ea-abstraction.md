---
title: Fee and Ea Abstraction Layers
impact: MEDIUM
impactDescription: Incorrect Fee/Ea configuration causes flash corruption, excessive garbage collection times, or blocked NvM operations
tags: nvm, fee, ea, flash, eeprom, abstraction, garbage-collection, virtual-addressing, autosar
---

## Fee and Ea Abstraction Layers

Fee (Flash EEPROM Emulation) and Ea (EEPROM Abstraction) provide virtual addressing over raw flash/EEPROM. Fee handles flash-specific concerns (sector erase, wear leveling, garbage collection) while Ea handles external EEPROM. Both provide a uniform block-based interface to NvM. Understanding their interaction with underlying Fls/Eep drivers is critical.

**Incorrect (blocking on Fee operations, ignoring garbage collection timing):**

```c
/* WRONG: Synchronously waiting in application context */
void App_SaveData(void)
{
    NvM_WriteBlock(NVM_BLOCK_APP_DATA, &appData);

    /* Blocking wait — Fee garbage collection can take 100+ ms */
    NvM_RequestResultType status;
    do {
        NvM_GetErrorStatus(NVM_BLOCK_APP_DATA, &status);
    } while (status == NVM_REQ_PENDING);
    /* This blocks the entire task, causing deadline misses */
}

/* WRONG: Fee virtual page size mismatch with physical flash */
const Fee_BlockConfiguration_t Fee_BlockConfig[] = {
    {
        .blockNumber = 1,
        .blockSize   = 100,     /* Not aligned to virtual page size */
        /* Fee will pad to page boundary anyway, wasting space */
    },
};
```

**Correct (asynchronous NvM/Fee operation with proper configuration):**

```c
/* Fee architecture:
 *
 *  NvM (block-level)
 *    |
 *  Fee (virtual addressing, wear leveling, GC)
 *    |
 *  Fls (raw flash driver)
 *
 *  NvM (block-level)
 *    |
 *  Ea  (virtual addressing for EEPROM)
 *    |
 *  Eep (raw EEPROM driver, typically SPI/I2C)
 */

/* Fee configuration — proper block sizing */
const Fee_BlockConfiguration_t Fee_BlockConfig[] = {
    {
        .blockNumber  = FEE_BLOCK_SAFETY_PARAMS,
        .blockSize    = FEE_ALIGN_TO_VPAGE(sizeof(SafetyParams_t)),
        .immediateData = TRUE,    /* Priority block — survives partial GC */
    },
    {
        .blockNumber  = FEE_BLOCK_CALIBRATION,
        .blockSize    = FEE_ALIGN_TO_VPAGE(sizeof(CalibrationData_t)),
        .immediateData = FALSE,
    },
};

/* Fee general configuration */
const Fee_GeneralConfiguration_t Fee_GeneralConfig = {
    .virtualPageSize     = 8U,          /* Must match Fls minimum write unit */
    .numSectors          = 2U,          /* Minimum 2 for GC — active + spare */
    .sectorSize          = 0x10000U,    /* 64KB — must match physical sector */
    .gcRestartPoint      = 90U,         /* Start GC when 90% of sector is used */
};
```

**Asynchronous operation pattern with MainFunction processing:**

```c
/* Cyclic task drives Fee and Fls state machines */
void Task_NvM_10ms(void)
{
    /* Order matters: Fls first, then Fee, then NvM */
    Fls_MainFunction();       /* Process pending flash operations */
    Fee_MainFunction();       /* Process Fee state machine (incl. GC) */
    NvM_MainFunction();       /* Process NvM queue */
}

/* Application uses callback notification instead of polling */
void NvM_SingleBlockCallback(NvM_BlockIdType blockId,
                              NvM_RequestResultType result)
{
    switch (blockId)
    {
        case NVM_BLOCK_SAFETY_PARAMS:
            if (result == NVM_REQ_OK)
            {
                SafetyParams_Valid = TRUE;
            }
            else
            {
                Dem_ReportErrorStatus(DEM_NVM_SAFETY_WRITE_FAILED,
                                      DEM_EVENT_STATUS_FAILED);
            }
            break;
        default:
            break;
    }
}
```

**Block invalidation and handling stale data:**

```c
/* Invalidate a block — marks it as empty in Fee's virtual address space */
void App_FactoryReset(void)
{
    NvM_InvalidateNvBlock(NVM_BLOCK_USER_SETTINGS);
    /* Next NvM_ReadBlock will load ROM default or return NVM_REQ_NV_INVALIDATED */
}

/* Fee internal: invalidation writes an invalidation marker,
   does NOT erase the physical flash — that happens during GC */

/* Ea (EEPROM) equivalent — same NvM API, different underlying driver */
/* Ea handles byte-addressable EEPROM without sector erase complexity */
const Ea_BlockConfiguration_t Ea_BlockConfig[] = {
    {
        .blockNumber = EA_BLOCK_ODOMETER,
        .blockSize   = sizeof(uint32_t),
        .immediateData = TRUE,
    },
};
```

**Garbage collection timing awareness:**

```c
/* Fee GC can stall NvM writes — plan for worst-case timing */
/*
 * GC phases:
 * 1. Copy valid blocks from full sector to spare sector
 * 2. Erase full sector (sector erase time: 50-200ms typical)
 * 3. Swap active/spare sector pointers
 *
 * During GC, new write requests are queued but not processed.
 * Immediate-data blocks get priority during GC.
 */

/* Budget estimation:
 * - Sector erase: ~100ms (varies by flash family)
 * - Block copy: ~1ms per block
 * - Total GC time: erase + (num_valid_blocks * copy_time)
 * - Must fit within shutdown timing budget for NvM_WriteAll
 */
```

Size Fee blocks to align with the virtual page size to avoid wasted padding. Reserve at least one spare sector for garbage collection. Monitor Fee sector fill level with `Fee_GetSectorStatus()` to avoid unexpected GC during safety-critical operations.

Reference: AUTOSAR SWS_Fee; AUTOSAR SWS_Ea; AUTOSAR SWS_Fls; AUTOSAR SWS_Eep
