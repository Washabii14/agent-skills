---
title: EcuM — ECU State Manager Startup/Shutdown and Wakeup Handling
impact: HIGH
impactDescription: Incorrect EcuM sequencing causes failed startup, uncontrolled shutdown, or missed wakeup events leading to field failures
tags: autosar, classic, ecum, startup, shutdown, sleep, wakeup, bsw, state-management
---

## EcuM — ECU State Manager Startup/Shutdown and Wakeup Handling

The ECU State Manager (EcuM) orchestrates the full ECU lifecycle: startup initialization, RUN state management, sleep entry, and wakeup validation. AUTOSAR R22-11 defines the Flexible EcuM (superseding the older Fixed variant). Misuse of EcuM APIs leads to partial initialization, uncontrolled shutdown, or wakeup sources that are never validated — all safety-critical in automotive systems.

### Startup Sequence

**Incorrect (calling BSW modules before EcuM_Init completes):**

```c
void main(void)
{
    Com_Init(&ComConfig);      /* COM initialized too early — Dem/SchM not ready */
    EcuM_Init(&EcuM_Config);
    /* ... */
}
```

**Correct (EcuM_Init drives the full startup sequence):**

```c
void main(void)
{
    EcuM_Init(&EcuM_Config);
    /* EcuM_Init internally calls:
     *   1. EcuM_AL_SetProgrammableInterrupts()
     *   2. EcuM_AL_DriverInitZero()  — pre-OS drivers (Mcu, Port, Dio)
     *   3. Os_Init / StartOS()
     *   4. EcuM_StartupTwo() from OS startup hook:
     *        - EcuM_AL_DriverInitOne()  — post-OS drivers (SchM, BswM, etc.)
     *        - BswM_Init()
     *        - SchM_Init()
     */
    /* Never returns — OS takes control */
}
```

### Wakeup Source Validation

**Incorrect (trusting wakeup source without validation):**

```c
void EcuM_CheckWakeup(EcuM_WakeupSourceType wakeupSource)
{
    /* Immediately accept any wakeup — no debounce/validation */
    EcuM_SetWakeupEvent(wakeupSource);
}
```

**Correct (two-phase wakeup: set pending, then validate):**

```c
void EcuM_CheckWakeup(EcuM_WakeupSourceType wakeupSource)
{
    /* Phase 1: Mark wakeup as pending — starts validation timer */
    EcuM_SetWakeupEvent(wakeupSource);
}

/* Called by the driver (e.g., CanTrcv) after confirming the wakeup source */
void EcuM_ValidateWakeupEvent(EcuM_WakeupSourceType wakeupSource)
{
    /* Phase 2: Wakeup confirmed — EcuM transitions to RUN */
    /* If validation timeout expires before this call, wakeup is rejected */
}
```

### Sleep / Shutdown Handling

**Incorrect (shutting down without releasing RUN requests):**

```c
void App_RequestShutdown(void)
{
    /* Direct call — but other SWCs still hold RUN requests */
    EcuM_GoDown(ECUM_USER_App);
}
```

**Correct (release all RUN users, let EcuM orchestrate shutdown):**

```c
void App_RequestShutdown(void)
{
    EcuM_ReleaseRUN(ECUM_USER_App);
    /* When all RUN users release, EcuM transitions:
     *   RUN -> POST_RUN -> SHUTDOWN/SLEEP
     * BswM evaluates mode conditions and executes action lists
     * NvM_WriteAll, ComM_DeInit, etc. happen via BswM action lists
     */
}

void EcuM_AL_SwitchOff(void)
{
    /* Callout: final hardware-specific power-down */
    Mcu_PerformReset();
}
```

### Flexible vs Fixed EcuM

The Fixed EcuM (deprecated) used hardcoded phase sequences. The Flexible EcuM (R22-11 standard) delegates mode arbitration to BswM, providing configurable startup/shutdown behavior via BswM rules and action lists.

```c
/* EcuM Flexible: mode requests flow through BswM */
void EcuM_Callout_StartupTwo(void)
{
    /* BswM evaluates startup rules and executes appropriate action lists */
    BswM_EcuM_CurrentState(ECUM_STATE_STARTUP_TWO);
}
```

Reference: AUTOSAR Classic R22-11 SWS_EcuM (SWS ECU State Manager), SRS_EcuM
