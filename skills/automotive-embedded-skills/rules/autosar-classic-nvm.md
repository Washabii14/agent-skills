---
title: NvM Module — Block Configuration, CRC Protection, and Startup/Shutdown Handling
impact: HIGH
impactDescription: NvM misuse causes data loss on power failure, corrupted calibration data, or excessive flash wear leading to field failures
tags: autosar, classic, nvm, persistent-storage, crc, redundant-block, flash, fee, eeprom, nvram
---

## NvM Module — Block Configuration, CRC Protection, and Startup/Shutdown Handling

The AUTOSAR NvM module (R22-11) provides non-volatile data management for AUTOSAR applications. It abstracts EEPROM/Flash hardware through MemIf/Fee/Ea layers and handles block integrity, redundancy, and job queuing. Incorrect NvM usage leads to data corruption, flash wear-out, and startup failures.

### Block Types

```c
/* NVM_BLOCK_NATIVE:    Single copy, basic storage
 *   - Use for non-critical configuration data
 *
 * NVM_BLOCK_REDUNDANT: Two copies with automatic failover
 *   - Use for safety-critical data (calibration, adaptation values)
 *   - NvM automatically reads backup if primary is corrupted
 *
 * NVM_BLOCK_DATASET:   Array of data sets selectable by index
 *   - Use for multiple variants (e.g., country-specific configs)
 *   - NvM_SetDataIndex() selects active dataset
 */
```

### Block Configuration

**Incorrect (no CRC, no default data, no write protection):**

```c
/* ARXML: NvMBlockDescriptor with minimal config
 *   NvMBlockId: 1
 *   NvMBlockLength: 64
 *   NvMBlockManagementType: NVM_BLOCK_NATIVE
 *   NvMBlockUseCrc: FALSE            <-- No integrity check
 *   NvMRomBlockDataAddress: NULL     <-- No default/ROM data
 *   NvMWriteBlockOnce: FALSE
 */
```

**Correct (CRC-protected, with ROM defaults and callbacks):**

```c
/* ARXML: NvMBlockDescriptor — production-quality config
 *   NvMBlockId: 1
 *   NvMBlockLength: 64
 *   NvMBlockManagementType: NVM_BLOCK_REDUNDANT
 *   NvMBlockUseCrc: TRUE
 *   NvMBlockCrcType: NVM_CRC32
 *   NvMCalcRamBlockCrc: TRUE          -- Detect RAM corruption
 *   NvMRomBlockDataAddress: &CalData_RomDefaults
 *   NvMInitBlockCallback: App_CalData_InitBlock
 *   NvMSingleBlockCallback: App_CalData_JobEndNotification
 *   NvMBlockWriteProt: FALSE
 *   NvMResistantToChangedSw: TRUE     -- Survive SW update
 */

/* ROM default data — loaded when NV block is invalid */
static const CalData_t CalData_RomDefaults = {
    .engineIdleRpm = 800u,
    .maxBoostPressure = 1500u,
    /* ... */
};
```

### Immediate vs Deferred Writes

**Incorrect (using immediate write for non-critical data):**

```c
void App_StoreOdometer(void)
{
    /* Immediate write blocks caller until flash operation completes */
    NvM_WriteBlock(NvMConf_NvMBlock_Odometer, &odoData);
    /* This may take several milliseconds — blocks real-time task */
}
```

**Correct (deferred for normal data, immediate only for crash-relevant):**

```c
/* Deferred write: queued and processed in NvM_MainFunction context */
void App_StoreOdometer(void)
{
    NvM_SetRamBlockStatus(NvMConf_NvMBlock_Odometer, TRUE);
    /* Block marked dirty — written during NvM_WriteAll at shutdown */
}

/* Immediate write: only for data that must survive unexpected power loss */
void App_StoreCrashData(const CrashData_t *data)
{
    (void)memcpy(&crashDataRam, data, sizeof(CrashData_t));

    /* NvMBlockPriority = 0 (highest) + NvMWriteBlockImmediately = TRUE
     * NvM processes this block ahead of the normal job queue */
    Std_ReturnType ret = NvM_WriteBlock(NvMConf_NvMBlock_CrashData,
                                         &crashDataRam);
    if (ret != E_OK)
    {
        /* Queue full — critical error handling */
    }
}
```

### Startup: NvM_ReadAll

```c
/* Called by EcuM during startup (via BswM action list) */
/* EcuM_AL_DriverInitOne -> NvM_ReadAll() */

/* NvM reads ALL configured persistent blocks:
 *   1. Read NV data from Fee/Ea
 *   2. Verify CRC
 *   3. If CRC fails on NATIVE: load ROM defaults, report to Dem
 *   4. If CRC fails on REDUNDANT: try backup copy
 *   5. Call NvMInitBlockCallback if block invalid and no ROM default
 */

void App_CalData_InitBlock(void)
{
    /* Called when block has no valid NV data AND no ROM default */
    (void)memset(&calDataRam, 0, sizeof(CalData_t));
    calDataRam.engineIdleRpm = 800u;  /* Safe defaults */
}

void App_CalData_JobEndNotification(uint8 ServiceId, NvM_RequestResultType JobResult)
{
    if (ServiceId == NVM_READ_BLOCK && JobResult == NVM_REQ_OK)
    {
        CalData_Apply(&calDataRam);
    }
    else if (JobResult == NVM_REQ_INTEGRITY_FAILED)
    {
        Dem_ReportErrorStatus(DTC_NVRAM_INTEGRITY, DEM_EVENT_STATUS_FAILED);
    }
}
```

### Shutdown: NvM_WriteAll

```c
/* Called during EcuM shutdown (via BswM action list):
 *   1. BswM detects ECUM_STATE_GO_SLEEP / ECUM_STATE_GO_OFF
 *   2. Action list calls NvM_WriteAll()
 *   3. NvM writes all blocks with NvM_SetRamBlockStatus(TRUE)
 *   4. BswM polls NvM_GetErrorStatus(NVM_BLOCK_MULTI) for completion
 *   5. Only after NVM_REQ_OK does BswM allow EcuM to power down
 */

/* NvM_WriteAll timeout: configure NvMWriteAllMaxDuration to prevent
 * infinite hang if flash hardware fails */
```

### Job Queue Handling

```c
/* NvM processes one job at a time per priority level
 * NvMJobPriorityMechanism: PRIORITY_BASED
 *   - Priority 0 (immediate) preempts priority 1+ (standard)
 *   - Within same priority: FIFO order
 *
 * NvM_MainFunction() processes jobs — called cyclically (5-10ms)
 * Each call processes one step of the current job
 *
 * Check status before issuing new request: */
NvM_RequestResultType result;
NvM_GetErrorStatus(NvMConf_NvMBlock_CalData, &result);
if (result != NVM_REQ_PENDING)
{
    NvM_WriteBlock(NvMConf_NvMBlock_CalData, &calDataRam);
}
```

Reference: AUTOSAR Classic R22-11 SWS_NvM (SWS NVRAM Manager), SRS_MemHwAb
