---
title: Diagnostics — Dcm Service Handlers, Dem Event Management, and DTC Handling
impact: HIGH
impactDescription: Incorrect diagnostic implementation causes failed OBD compliance, workshop tool incompatibility, or undetected faults in the field
tags: autosar, classic, dcm, dem, fim, uds, dtc, diagnostics, obd, snapshot, freeze-frame
---

## Diagnostics — Dcm Service Handlers, Dem Event Management, and DTC Handling

AUTOSAR Classic R22-11 diagnostics span Dcm (Diagnostic Communication Manager — UDS protocol handling), Dem (Diagnostic Event Manager — fault storage), and FiM (Function Inhibition Manager). Together they implement ISO 14229 UDS services, manage DTCs, and control function availability based on fault status.

### Dcm — UDS Service Handler Mapping

**Incorrect (implementing UDS services outside Dcm framework):**

```c
void App_ProcessDiagRequest(const uint8 *data, uint16 len)
{
    if (data[0] == 0x22u)  /* ReadDataByIdentifier */
    {
        /* Manual parsing — no session/security checks, no NRC handling */
        uint16 did = ((uint16)data[1] << 8u) | data[2];
        App_ReadDid(did, responseBuffer);
    }
}
```

**Correct (Dcm callout functions with proper NRC handling):**

```c
/* Dcm routes UDS 0x22 to configured DID read function */
Std_ReturnType App_ReadDid_EngineSpeed(
    Dcm_OpStatusType OpStatus,
    uint8 *Data,
    Dcm_NegativeResponseCodeType *ErrorCode)
{
    if (OpStatus == DCM_INITIAL)
    {
        uint16 rpm = EngineCtrl_GetRpm();
        Data[0] = (uint8)(rpm >> 8u);
        Data[1] = (uint8)(rpm & 0xFFu);
        return E_OK;
    }

    if (OpStatus == DCM_CANCEL)
    {
        return E_OK;
    }

    *ErrorCode = DCM_E_CONDITIONSNOTCORRECT;
    return E_NOT_OK;
}

/* Dcm handles: session check, security access, response formatting,
 * pending (0x78) responses for async operations, NRC propagation */
```

### Dcm — Session and Security Access

```c
/* Dcm configuration controls access per session/security level:
 *
 * DcmDspDid: DID_0xF190 (VIN)
 *   DcmDspDidReadSessionRef: DEFAULT_SESSION, EXTENDED_SESSION
 *   DcmDspDidReadSecurityLevelRef: LEVEL_0 (no security)
 *
 * DcmDspDid: DID_0xFD01 (CalibrationData)
 *   DcmDspDidWriteSessionRef: PROGRAMMING_SESSION
 *   DcmDspDidWriteSecurityLevelRef: LEVEL_1
 *
 * Dcm automatically sends NRC 0x31 (requestOutOfRange) or
 * NRC 0x33 (securityAccessDenied) when preconditions not met
 */
```

### Dem — Event Reporting and DTC Status

**Incorrect (directly setting DTC status bits):**

```c
void App_CheckSensor(void)
{
    if (SensorFailed())
    {
        g_dtcStatus[DTC_SENSOR_FAIL] |= 0x01u;  /* Manual bit manipulation */
    }
}
```

**Correct (use Dem_ReportErrorStatus / Dem_SetEventStatus):**

```c
void App_CheckSensor(void)
{
    if (SensorFailed())
    {
        Dem_ReportErrorStatus(DemConf_DemEventParameter_SensorFault,
                              DEM_EVENT_STATUS_FAILED);
        /* Dem handles:
         *   - Debouncing (counter-based or time-based)
         *   - DTC status byte update (testFailed, confirmedDTC, etc.)
         *   - Snapshot/freeze-frame capture
         *   - FiM notification for function inhibition
         *   - Aging counter management
         */
    }
    else
    {
        Dem_ReportErrorStatus(DemConf_DemEventParameter_SensorFault,
                              DEM_EVENT_STATUS_PASSED);
    }
}
```

### DTC Status Byte (ISO 14229-1)

```c
/* Bit 0: testFailed             — current test result
 * Bit 1: testFailedThisOperationCycle
 * Bit 2: pendingDTC             — failed but not yet confirmed
 * Bit 3: confirmedDTC           — fault confirmed (trip counter reached)
 * Bit 4: testNotCompletedSinceLastClear
 * Bit 5: testFailedSinceLastClear
 * Bit 6: testNotCompletedThisOperationCycle
 * Bit 7: warningIndicatorRequested (MIL lamp)
 *
 * Never manipulate directly — Dem manages transitions per configured
 * debounce thresholds, operation cycles, and aging criteria
 */
```

### Snapshot (Freeze-Frame) and Extended Data

```c
/* Dcm configuration for snapshot capture on DTC storage:
 *
 * DemFreezeFrameClass: FF_Standard
 *   DemDidRef: DID_EngineSpeed
 *   DemDidRef: DID_VehicleSpeed
 *   DemDidRef: DID_CoolantTemp
 *   DemMaxNumberOfFreezeFrameRecords: 2
 *
 * DemExtendedDataClass: ExtData_Standard
 *   DemExtendedDataRecordRef: Occurrence_Counter
 *   DemExtendedDataRecordRef: Aging_Counter
 *   DemExtendedDataRecordRef: OperationCycle_Counter
 */

/* Application provides data via callback */
Std_ReturnType App_ReadFreezeFrame_EngineSpeed(uint8 *Data)
{
    uint16 rpm = EngineCtrl_GetRpm();
    Data[0] = (uint8)(rpm >> 8u);
    Data[1] = (uint8)(rpm & 0xFFu);
    return E_OK;
}
```

### FiM — Function Inhibition

```c
/* FiM inhibits application functions based on DTC status */
void App_CruiseControl_Cyclic(void)
{
    boolean permission = FALSE;
    FiM_GetFunctionPermission(FiMConf_FiMFId_CruiseControl, &permission);

    if (permission == FALSE)
    {
        /* CruiseControl inhibited due to active fault
         * (e.g., brake sensor DTC confirmed) */
        CruiseCtrl_Disable();
        return;
    }

    CruiseCtrl_Execute();
}
```

Reference: AUTOSAR Classic R22-11 SWS_Dcm, SWS_Dem, SWS_FiM; ISO 14229 (UDS)
