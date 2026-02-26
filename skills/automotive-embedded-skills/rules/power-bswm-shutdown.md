---
title: BswM Ordered Shutdown Sequence
impact: MEDIUM
impactDescription: Unordered shutdown causes NvM data loss, incomplete communication teardown, or diagnostic faults stored against shutdown noise
tags: power, bswm, shutdown, nvm, comm, low-power, sleep, mode-management, autosar
---

## BswM Ordered Shutdown Sequence

BswM executes ordered shutdown action lists to safely transition to low-power mode: request no-communication → wait for Tx confirmation → save NvM → disable DTCs → deinitialize BSW modules → enter MCU low-power mode. Skipping or misordering steps causes data loss or communication errors.

**Incorrect (immediate sleep without ordered teardown):**

```c
/* WRONG: No ordered shutdown — data loss and communication errors */
void System_Shutdown_Bad(void)
{
    Mcu_SetMode(MCU_MODE_SLEEP);  /* Immediate — NvM not saved */
    /* Pending CAN messages dropped mid-transmission */
    /* DEM may log false DTCs from abrupt peripheral shutdown */
}
```

**Correct (BswM-orchestrated shutdown with action lists):**

```c
/* BswM shutdown rules and action lists */

/* Rule: When EcuM requests PREP_SHUTDOWN, execute shutdown sequence */
const BswM_Rule_t BswM_ShutdownRules[] = {
    {
        .ruleId = BSWM_RULE_SHUTDOWN_START,
        .condition = BSWM_COND_ECUM_STATE_PREP_SHUTDOWN,
        .trueAction = BSWM_ACTIONLIST_SHUTDOWN_PHASE1,
        .falseAction = BSWM_NO_ACTION,
    },
    {
        .ruleId = BSWM_RULE_COMM_SILENT,
        .condition = BSWM_COND_ALL_CHANNELS_NO_COM,
        .trueAction = BSWM_ACTIONLIST_SHUTDOWN_PHASE2,
        .falseAction = BSWM_NO_ACTION,
    },
    {
        .ruleId = BSWM_RULE_NVM_COMPLETE,
        .condition = BSWM_COND_NVM_WRITEALL_DONE,
        .trueAction = BSWM_ACTIONLIST_SHUTDOWN_PHASE3,
        .falseAction = BSWM_NO_ACTION,
    },
};
```

```c
/* Phase 1: Communication teardown */
void BswM_ActionList_ShutdownPhase1(void)
{
    /* Step 1: Disable DTC setting — prevent false DTCs during shutdown */
    Dem_SetDTCSetting(DEM_DTC_GROUP_ALL, DEM_DTC_SETTING_OFF);

    /* Step 2: Stop COM signal transmission */
    Com_IpduGroupStop(COM_IPDU_GROUP_TX_ALL);

    /* Step 3: Request no-communication for all channels */
    ComM_RequestComMode(COMM_CHANNEL_CAN, COMM_NO_COMMUNICATION);
    ComM_RequestComMode(COMM_CHANNEL_LIN, COMM_NO_COMMUNICATION);

    /* Step 4: Wait for pending Tx confirmations (up to timeout) */
    /* BswM will evaluate BSWM_RULE_COMM_SILENT on next cycle */
}

/* Phase 2: Save NvM data */
void BswM_ActionList_ShutdownPhase2(void)
{
    /* All communication is down — safe to save NvM */
    NvM_WriteAll();
    /* BswM will poll NvM status on each cycle */
}

/* Phase 3: Final deinit and sleep */
void BswM_ActionList_ShutdownPhase3(void)
{
    /* Step 1: Deinitialize BSW modules in reverse startup order */
    Dcm_DeInit();
    Com_DeInit();
    PduR_DeInit();
    CanIf_DeInit();
    Can_DeInit();

    /* Step 2: Configure wakeup sources */
    EcuM_EnableWakeupSources(ECUM_WKSOURCE_CAN_RX |
                              ECUM_WKSOURCE_POWER_SWITCH);

    /* Step 3: Set hardware outputs to safe state */
    Dio_WriteChannel(DIO_CH_ACTUATOR_EN, STD_LOW);
    Dio_WriteChannel(DIO_CH_RELAY_CTRL, STD_LOW);

    /* Step 4: Enter low-power mode */
    EcuM_GoSleep();
}
```

**ComM no-communication handling:**

```c
/* ComM state machine for channel shutdown */
void ComM_MainFunction_Channel(ComM_ChannelIdType channel)
{
    switch (ComM_GetChannelState(channel))
    {
        case COMM_FULL_COMMUNICATION:
            if (ComM_IsNoCommunicationRequested(channel))
            {
                /* Transition through SILENT first for graceful teardown */
                ComM_SetChannelState(channel, COMM_SILENT_COMMUNICATION);
                CanSM_RequestComMode(channel, COMM_SILENT_COMMUNICATION);
            }
            break;

        case COMM_SILENT_COMMUNICATION:
            /* Wait for bus idle and pending Tx completion */
            if (CanSM_GetCurrentComMode(channel) == COMM_SILENT_COMMUNICATION)
            {
                ComM_SetChannelState(channel, COMM_NO_COMMUNICATION);
                CanSM_RequestComMode(channel, COMM_NO_COMMUNICATION);

                /* Notify BswM that channel is silent */
                BswM_ComM_CurrentMode(channel, COMM_NO_COMMUNICATION);
            }
            break;

        case COMM_NO_COMMUNICATION:
            /* Channel is down — transceiver can enter standby */
            CanTrcv_SetOpMode(CANTRCV_TRCV_MODE_STANDBY);
            break;

        default:
            break;
    }
}
```

**Shutdown timing budget:**

```c
/* Shutdown timing must fit within power supply hold-up time.
 *
 * Typical timing budget (from ignition-off to sleep):
 *   Phase 1: ComM no-com + NM shutdown     ~200ms
 *   Phase 2: NvM_WriteAll                   ~300ms (depends on block count)
 *   Phase 3: BSW deinit + enter sleep       ~50ms
 *   Total:                                  ~550ms
 *
 * Hardware hold-up capacitor must sustain power for total + margin.
 * If NvM_WriteAll exceeds budget, use NvM_WritePrioBlock for critical data.
 */

#define SHUTDOWN_TIMEOUT_MS  800U  /* Total budget with margin */

void EcuM_MonitorShutdownTiming(void)
{
    static uint32_t shutdownStart = 0U;

    if (shutdownStart == 0U)
    {
        shutdownStart = Timer_GetMs();
    }

    if ((Timer_GetMs() - shutdownStart) > SHUTDOWN_TIMEOUT_MS)
    {
        /* Emergency: force sleep without completing NvM */
        NvM_CancelWriteAll();
        BswM_ActionList_ShutdownPhase3();
    }
}
```

The transition `FULL_COM → SILENT_COM → NO_COM` must be respected — jumping directly to `NO_COM` can cause CAN bus errors from abruptly terminated transmissions. Monitor shutdown timing with a hardware timer to enforce the power hold-up budget.

Reference: AUTOSAR SWS_BswM (Shutdown Action Lists); AUTOSAR SWS_ComM (Communication Modes); OEM-specific shutdown timing requirements
