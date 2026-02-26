---
title: EcuM Sleep and Wakeup Cycle
impact: HIGH
impactDescription: Incorrect sleep/wakeup handling causes missed wakeup events, excessive current draw, or corrupted state on resume
tags: power, ecum, sleep, wakeup, startup, shutdown, mode-management, autosar
---

## EcuM Sleep and Wakeup Cycle

The EcuM manages the ECU power state lifecycle: STARTUP → RUN → SLEEP → WAKEUP → (back to RUN or SHUTDOWN). Wakeup sources must be configured before entering sleep, validated after wakeup, and the system must cleanly transition through all intermediate states. Spurious wakeups must be detected and handled.

**Incorrect (entering sleep without saving NvM, no wakeup source validation):**

```c
/* WRONG: Skipping critical shutdown steps */
void EcuM_GoSleep_Bad(void)
{
    /* Missing: NvM_WriteAll — unsaved data lost */
    /* Missing: Disable communication orderly */
    /* Missing: Configure wakeup sources */

    Mcu_SetMode(MCU_MODE_SLEEP);  /* Immediate sleep — data loss risk */
}

void EcuM_WakeupRestart_Bad(void)
{
    /* No wakeup source validation — spurious wakeup accepted */
    EcuM_SetState(ECUM_STATE_RUN);  /* May enter RUN on noise/glitch */
}
```

**Correct (proper EcuM sleep/wakeup state machine):**

```c
/* EcuM state machine — AUTOSAR-aligned */
typedef enum {
    ECUM_STATE_STARTUP,
    ECUM_STATE_APP_RUN,
    ECUM_STATE_APP_POST_RUN,
    ECUM_STATE_PREP_SHUTDOWN,
    ECUM_STATE_GO_SLEEP,
    ECUM_STATE_SLEEP,
    ECUM_STATE_WAKEUP_ONE,
    ECUM_STATE_WAKEUP_VALIDATION,
    ECUM_STATE_WAKEUP_REACTION,
    ECUM_STATE_GO_OFF_ONE,
    ECUM_STATE_OFF
} EcuM_State_t;
```

```c
/* ecum_sleep.c — proper sleep entry */

void EcuM_GoSleep(void)
{
    /* Phase 1: Prepare shutdown — notify BswM */
    BswM_EcuM_CurrentState(ECUM_STATE_PREP_SHUTDOWN);

    /* Phase 2: Disable communication orderly */
    ComM_DeInit();
    CanSM_RequestComMode(COMM_CHANNEL_CAN, COMM_NO_COMMUNICATION);

    /* Phase 3: Save persistent data */
    NvM_WriteAll();
    EcuM_WaitForNvMWriteAll();  /* Poll until complete or timeout */

    /* Phase 4: Deinitialize BSW modules (reverse of startup order) */
    Com_DeInit();
    PduR_DeInit();
    CanIf_DeInit();

    /* Phase 5: Configure wakeup sources */
    EcuM_EnableWakeupSources(ECUM_WKSOURCE_CAN_RX |
                              ECUM_WKSOURCE_POWER_SWITCH |
                              ECUM_WKSOURCE_TIMER);

    /* Phase 6: Set MCU to low-power mode */
    EcuM_SetState(ECUM_STATE_SLEEP);
    Mcu_SetMode(MCU_MODE_SLEEP);

    /* === CPU stops here — resumes on wakeup interrupt === */

    /* Phase 7: Wakeup detected — MCU returns from sleep */
    EcuM_SetState(ECUM_STATE_WAKEUP_ONE);
}
```

**Wakeup source configuration and validation:**

```c
/* ecum_wakeup.c */

/* Wakeup source bitmask definitions */
#define ECUM_WKSOURCE_CAN_RX        (1U << 0)
#define ECUM_WKSOURCE_LIN_RX        (1U << 1)
#define ECUM_WKSOURCE_POWER_SWITCH   (1U << 2)
#define ECUM_WKSOURCE_TIMER          (1U << 3)
#define ECUM_WKSOURCE_DIAG_REQUEST   (1U << 4)

static uint32_t g_pendingWakeupEvents = 0U;
static uint32_t g_validatedWakeupEvents = 0U;

/* Called from ISR when wakeup source triggers */
void EcuM_SetWakeupEvent(uint32_t wakeupSource)
{
    g_pendingWakeupEvents |= wakeupSource;
}

/* Called periodically after wakeup to validate sources */
void EcuM_CheckWakeup(uint32_t wakeupSource)
{
    switch (wakeupSource)
    {
        case ECUM_WKSOURCE_CAN_RX:
            /* Validate: was there a real CAN frame or just bus noise? */
            if (CanTrcv_CheckWakeFlag() == CANTRCV_WU_BY_BUS)
            {
                g_validatedWakeupEvents |= ECUM_WKSOURCE_CAN_RX;
            }
            break;

        case ECUM_WKSOURCE_POWER_SWITCH:
            /* Validate: debounce the power switch signal */
            if (Dio_ReadChannel(DIO_CH_POWER_SW) == STD_HIGH)
            {
                g_validatedWakeupEvents |= ECUM_WKSOURCE_POWER_SWITCH;
            }
            break;

        case ECUM_WKSOURCE_TIMER:
            /* Timer wakeup is always valid */
            g_validatedWakeupEvents |= ECUM_WKSOURCE_TIMER;
            break;

        default:
            break;
    }
}

/* Wakeup reaction — decide to stay awake or go back to sleep */
void EcuM_WakeupReaction(void)
{
    if (g_validatedWakeupEvents != 0U)
    {
        /* Valid wakeup — proceed to RUN state */
        EcuM_SetState(ECUM_STATE_APP_RUN);
        EcuM_PerformStartup();  /* Re-init BSW, communication */
    }
    else
    {
        /* Spurious wakeup — go back to sleep */
        g_pendingWakeupEvents = 0U;
        EcuM_GoSleep();
    }
}
```

**Wakeup validation timeout:**

```c
/* Wakeup validation has a timeout — if source not validated within
 * timeout, treat as spurious and return to sleep */
#define ECUM_WAKEUP_VALIDATION_TIMEOUT_MS  500U

void EcuM_WakeupValidationTask(void)
{
    static uint32_t validationTimer = 0U;

    if (g_ecumState == ECUM_STATE_WAKEUP_VALIDATION)
    {
        /* Check each pending source */
        uint32_t pending = g_pendingWakeupEvents & ~g_validatedWakeupEvents;
        for (uint32_t src = 1U; src != 0U; src <<= 1)
        {
            if (pending & src)
            {
                EcuM_CheckWakeup(src);
            }
        }

        validationTimer++;
        if (g_validatedWakeupEvents != 0U)
        {
            EcuM_WakeupReaction();
            validationTimer = 0U;
        }
        else if (validationTimer >= ECUM_WAKEUP_VALIDATION_TIMEOUT_MS)
        {
            /* Timeout — spurious wakeup */
            validationTimer = 0U;
            EcuM_WakeupReaction();  /* Will go back to sleep */
        }
    }
}
```

Always validate wakeup sources to avoid spurious wakeup loops that drain the vehicle battery. The wakeup validation timeout must be short enough to meet automotive sleep current requirements but long enough for legitimate sources to be detected (typically 100–500ms).

Reference: AUTOSAR SWS_EcuM (EcuM Fixed/Flex); ISO 26262 Part 6 (Safe state — sleep/wakeup); OEM-specific sleep current requirements (typically < 100μA)
