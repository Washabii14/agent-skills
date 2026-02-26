---
title: COM Module — Signal Packing, Transmission Modes, and Deadline Monitoring
impact: HIGH
impactDescription: Wrong signal packing or transmission mode causes corrupted bus data, missed messages, or excessive bus load
tags: autosar, classic, com, signal, pdu, transmission-mode, deadline-monitoring, callback, signal-group
---

## COM Module — Signal Packing, Transmission Modes, and Deadline Monitoring

The AUTOSAR COM module (R22-11) handles signal-level access for application SWCs: packing signals into I-PDUs, unpacking received I-PDUs into signals, managing transmission modes, and monitoring reception deadlines. Errors here directly corrupt bus communication.

### Signal Packing / Unpacking

**Incorrect (manual bit manipulation instead of COM API):**

```c
void App_SendEngineSpeed(uint16 rpm)
{
    uint8 pdu[8];
    pdu[0] = (uint8)(rpm & 0xFFu);
    pdu[1] = (uint8)((rpm >> 8u) & 0xFFu);
    /* Endianness, bit position from ARXML ignored — silent data corruption */
    CanIf_Transmit(PDU_ENGINE_STATUS, pdu);
}
```

**Correct (use Com_SendSignal for type-safe, config-driven packing):**

```c
void App_SendEngineSpeed(uint16 rpm)
{
    /* COM packs signal per ARXML: bit position, byte order, length, type */
    uint8 status = Com_SendSignal(ComConf_ComSignal_EngineSpeed, &rpm);

    if (status == COM_SERVICE_NOT_AVAILABLE)
    {
        /* I-PDU group not started — handle gracefully */
    }
    /* COM handles endianness (Opaque/BigEndian/LittleEndian) per config */
}

void App_ReadEngineSpeed(void)
{
    uint16 rpm = 0u;
    uint8 status = Com_ReceiveSignal(ComConf_ComSignal_EngineSpeed, &rpm);

    if (status == E_OK)
    {
        EngineCtrl_SetTargetRpm(rpm);
    }
}
```

### Signal Groups (Consistent Read/Write)

**Incorrect (reading group signals individually — inconsistent snapshot):**

```c
void App_ReadWheelSpeeds(void)
{
    Com_ReceiveSignal(SIG_WHL_FL, &fl);
    /* Task preemption here — other signals may update */
    Com_ReceiveSignal(SIG_WHL_FR, &fr);
    Com_ReceiveSignal(SIG_WHL_RL, &rl);
    Com_ReceiveSignal(SIG_WHL_RR, &rr);
}
```

**Correct (use signal group API for atomic access):**

```c
void App_ReadWheelSpeeds(void)
{
    /* Step 1: Copy shadow buffer atomically */
    uint8 status = Com_ReceiveSignalGroup(ComConf_ComSignalGroup_WheelSpeeds);

    if (status == E_OK)
    {
        /* Step 2: Read individual group signals from shadow buffer */
        Com_ReceiveSignalGroupArray(ComConf_ComSignalGroup_WheelSpeeds, shadowBuf);
        /* All four wheel speeds are from the same reception instant */
    }
}

void App_SendWheelSpeeds(void)
{
    /* Step 1: Write signals to shadow buffer */
    Com_UpdateShadowSignal(ComConf_ComGroupSignal_FL, &fl);
    Com_UpdateShadowSignal(ComConf_ComGroupSignal_FR, &fr);
    Com_UpdateShadowSignal(ComConf_ComGroupSignal_RL, &rl);
    Com_UpdateShadowSignal(ComConf_ComGroupSignal_RR, &rr);

    /* Step 2: Transfer shadow buffer to I-PDU atomically */
    Com_SendSignalGroup(ComConf_ComSignalGroup_WheelSpeeds);
}
```

### Transmission Modes

```c
/* PERIODIC: I-PDU transmitted every ComTxModeTimePeriod (e.g., 10ms)
 *   - Use for cyclic signals (engine speed, vehicle speed)
 *   - ComTxModeRepetitionPeriod for fast retransmissions after update
 *
 * DIRECT: I-PDU transmitted immediately on Com_SendSignal()
 *   - Use for event-driven signals (door open, fault active)
 *   - ComTxModeNumberOfRepetitions for reliability
 *
 * MIXED: Periodic baseline + immediate on value change
 *   - Use for signals needing both guaranteed update rate and fast reaction
 *
 * NONE: No automatic transmission — manual via Com_TriggerIPDUSend()
 */

/* Configure via ARXML ComTxMode container, not in code */
```

### Reception Deadline Monitoring

**Incorrect (no timeout detection for safety-critical signals):**

```c
void App_ProcessBrakeSignal(void)
{
    Com_ReceiveSignal(SIG_BRAKE_PRESSURE, &pressure);
    BrakeCtrl_Apply(pressure);  /* Stale value used if sender stopped */
}
```

**Correct (enable COM reception deadline monitoring):**

```c
/* ARXML Configuration:
 *   ComRxDataTimeoutAction = SUBSTITUTE (use init/substitute value)
 *   ComTimeout = 100ms (expected period + tolerance)
 *   ComTimeoutNotification = App_BrakeSigTimeout
 */

void App_BrakeSigTimeout(void)
{
    /* COM replaces signal with ComSignalInitValue automatically */
    BrakeCtrl_EnterSafeState();
    Dem_ReportErrorStatus(DTC_BRAKE_SIG_TIMEOUT, DEM_EVENT_STATUS_FAILED);
}

/* COM callback on successful reception — re-enables normal processing */
void App_BrakeSigRxIndication(void)
{
    BrakeCtrl_ExitSafeState();
}
```

Reference: AUTOSAR Classic R22-11 SWS_COM (SWS Communication), SRS_Com
