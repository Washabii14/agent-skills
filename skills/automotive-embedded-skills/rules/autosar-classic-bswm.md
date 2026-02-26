---
title: BswM — BSW Mode Manager Arbitration and Action Lists
impact: HIGH
impactDescription: Incorrect BswM rules cause wrong mode transitions, stranded communication stacks, or failed startup/shutdown sequences
tags: autosar, classic, bswm, mode-management, arbitration, action-list, comm, ecum, bsw
---

## BswM — BSW Mode Manager Arbitration and Action Lists

The BSW Mode Manager (BswM) is the central mode arbitration entity in AUTOSAR Classic R22-11. It receives mode indications from EcuM, ComM, DCM, NvM and others, evaluates rules against those indications, and executes action lists that control the BSW stack behavior. Misconfigured BswM rules are a leading cause of integration failures.

### Mode Request and Rule Evaluation

**Incorrect (bypassing BswM to directly control COM):**

```c
void App_EnableComm(void)
{
    Com_IpduGroupStart(COM_IPDU_GRP_ALL, FALSE);  /* Direct call — BswM unaware */
    ComM_RequestComMode(COMM_CHANNEL_0, COMM_FULL_COMMUNICATION);
}
```

**Correct (mode indication triggers BswM rule evaluation):**

```c
/* ComM indicates mode change to BswM */
void BswM_ComM_CurrentMode(NetworkHandleType Network, ComM_ModeType RequestedMode)
{
    /* BswM evaluates configured rules:
     *
     * Rule: "ComFullComm_Ch0"
     *   Condition: ComM_CurrentMode(Ch0) == COMM_FULL_COMMUNICATION
     *   Action List:
     *     1. Com_IpduGroupStart(TxGroup_Ch0)
     *     2. Com_IpduGroupStart(RxGroup_Ch0)
     *     3. PduR_EnableRouting(RoutingPath_Ch0)
     */
}

/* Application requests communication through ComM, not directly */
void App_EnableComm(void)
{
    ComM_RequestComMode(COMM_CHANNEL_0, COMM_FULL_COMMUNICATION);
    /* ComM -> BswM_ComM_CurrentMode -> Rule evaluation -> Action list */
}
```

### Action List Configuration

**Incorrect (action list with wrong execution order):**

```c
/* Pseudo-configuration: starting COM before PDU Router is ready */
/*
 * ActionList: AL_StartupComplete
 *   Action 1: Com_IpduGroupStart(ALL)       <-- COM sends before routing exists
 *   Action 2: PduR_EnableRouting(ALL)
 *   Action 3: CanSM_RequestComMode(FULL)
 */
```

**Correct (bottom-up activation order):**

```c
/* Pseudo-configuration: activate from bottom layer upward */
/*
 * ActionList: AL_StartupComplete
 *   Action 1: CanSM_RequestComMode(CAN_CH0, COMM_FULL_COMMUNICATION)
 *   Action 2: PduR_EnableRouting(RoutingPath_Ch0)
 *   Action 3: Com_IpduGroupStart(RxGroup_Ch0, TRUE)
 *   Action 4: Com_IpduGroupStart(TxGroup_Ch0, FALSE)
 *
 * Rationale: Bus transceiver -> CAN driver -> PDU routing -> COM layer
 */
```

### BswM Integration with EcuM Shutdown

**Incorrect (NvM_WriteAll not coordinated with shutdown):**

```c
/* EcuM goes to SLEEP before NvM finishes writing */
void ShutdownSequence(void)
{
    NvM_WriteAll();
    EcuM_GoSleep();  /* NvM may still be writing — data corruption risk */
}
```

**Correct (BswM rule waits for NvM_WriteAll completion):**

```c
/*
 * Rule: "ShutdownReady"
 *   Condition: EcuM_CurrentState == ECUM_STATE_GO_SLEEP
 *              AND NvM_WriteAll_Status == NVM_REQ_OK
 *   Action List: AL_GoSleep
 *     Action 1: Com_IpduGroupStop(ALL)
 *     Action 2: ComM_RequestComMode(ALL, COMM_NO_COMMUNICATION)
 *     Action 3: BswM_ActionList_GoSleep()
 *
 * BswM_NvM_CurrentJobMode() callback updates the NvM status.
 * EcuM only proceeds to SLEEP when BswM confirms all conditions met.
 */
```

### Deferred vs Immediate Rule Evaluation

```c
/* Immediate: evaluated in the context of the indication call */
/* Use for safety-critical mode transitions */

/* Deferred: evaluated in BswM_MainFunction() context */
/* Use for non-time-critical transitions to avoid long call chains */

void BswM_MainFunction(void)
{
    /* Evaluates all deferred rules, executes triggered action lists */
    /* Called cyclically from SchM (typically 5-10ms) */
}
```

Reference: AUTOSAR Classic R22-11 SWS_BswM (SWS BSW Mode Manager), SRS_ModeMgm
