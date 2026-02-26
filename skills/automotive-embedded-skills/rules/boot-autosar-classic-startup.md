---
title: Classic AUTOSAR Boot Sequence
impact: HIGH
impactDescription: Incorrect AUTOSAR startup phase ordering causes BSW module initialization failures, missing RTE connections, or unresponsive SWCs
tags: boot, autosar, classic, ecum, bswm, schm, rte, startup, mcal
---

## Classic AUTOSAR Boot Sequence

Classic AUTOSAR defines a strict startup order: `EcuM_Init()` → StartupOne (MCAL drivers) → StartupTwo (BSW modules) → `BswM` mode arbitration → `SchM` scheduling → `RTE_Start()` → SWC Runnable initialization. Each phase has prerequisites that must be satisfied before proceeding.

**Incorrect (initializing BSW modules before MCAL, or starting RTE before SchM):**

```c
/* WRONG: BSW modules initialized before their MCAL dependencies */
void EcuM_Init(void)
{
    /* Missing: MCAL init (Mcu, Port, Dio) */
    CanIf_Init(&CanIf_Config);    /* FAIL: CAN driver not initialized */
    Com_Init(&Com_Config);         /* FAIL: CanIf not ready */
    Rte_Start();                   /* FAIL: SchM not started, no scheduling */
}
```

**Correct (proper AUTOSAR startup phase ordering):**

```c
/* EcuM_Init — entry point after C runtime init */
void EcuM_Init(void)
{
    /* === STARTUP ONE: MCAL Initialization === */
    Mcu_Init(&Mcu_Config);
    Mcu_InitClock(McuClockSettingConfig_0);
    while (Mcu_GetPllStatus() != MCU_PLL_LOCKED) { /* wait */ }
    Mcu_DistributePllClock();

    Port_Init(&Port_Config);
    Dio_Init(&Dio_Config);
    Gpt_Init(&Gpt_Config);
    Wdg_Init(&Wdg_Config);

    /* === STARTUP TWO: BSW Module Initialization === */
    /* Communication stack — bottom up */
    Can_Init(&Can_Config);
    CanIf_Init(&CanIf_Config);
    CanSM_Init(&CanSM_Config);
    CanTp_Init(&CanTp_Config);
    PduR_Init(&PduR_Config);
    Com_Init(&Com_Config);

    /* Diagnostic stack */
    Dcm_Init(&Dcm_Config);
    Dem_Init();
    Dem_SetOperationCycleState(DemConf_OperationCycle_POWER, DEM_CYCLE_STATE_START);

    /* NvM stack */
    Fee_Init(&Fee_Config);
    NvM_Init(&NvM_Config);
    NvM_ReadAll();

    /* === BSW Mode Manager === */
    BswM_Init(&BswM_Config);

    /* === Schedule Manager — enables BSW MainFunction scheduling === */
    SchM_Init();

    /* === RTE Start — connects SWC ports === */
    Rte_Start();

    /* === SWC Initialization Runnables (triggered by RTE) === */
    /* RTE calls <Swc>_Init() runnables based on INIT events */

    /* === EcuM enters RUN mode === */
    EcuM_SetState(ECUM_STATE_APP_RUN);
}
```

**Mode request flow for run/sleep transitions:**

```c
/* BswM rule: handle EcuM mode indications */
void BswM_ActionList_RunMode(void)
{
    ComM_CommunicationAllowed(ComMConf_ComMChannel_CAN, TRUE);
    CanSM_RequestComMode(ComMConf_ComMChannel_CAN, COMM_FULL_COMMUNICATION);
    /* Enable periodic Com transmission */
    Com_IpduGroupStart(ComConf_IpduGroup_TxPduGroup, TRUE);
}

/* Mode request from SWC to EcuM */
void AppSwc_RequestShutdown(void)
{
    /* SWC requests mode change through RTE → BswM */
    Rte_Switch_ModePort_currentMode(RTE_MODE_EcuM_Mode_SLEEP);
    /* BswM evaluates rules and triggers shutdown action list */
}
```

Verify startup order with AUTOSAR Module Dependency Matrix. `NvM_ReadAll()` can be asynchronous — poll `NvM_GetStatus()` or use a BswM rule to gate on completion before allowing SWCs to read NvM data.

Reference: AUTOSAR SWS_EcuM (EcuM State Machine); AUTOSAR SWS_BswM (Mode Arbitration); ISO 26262 Part 6 (Software unit design — initialization)
