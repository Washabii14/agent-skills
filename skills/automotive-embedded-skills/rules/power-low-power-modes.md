---
title: MCU Low-Power Mode Selection
impact: MEDIUM
impactDescription: Selecting the wrong low-power mode causes either excessive current draw or loss of volatile state, RAM contents, or peripheral context
tags: power, low-power, sleep, standby, stop, deep-sleep, ram-retention, wakeup, mcu
---

## MCU Low-Power Mode Selection

Automotive MCUs provide multiple low-power modes with different trade-offs between current consumption, wakeup latency, and retained state. Common modes: SLEEP (CPU stopped, peripherals run), STOP (all clocks stopped, RAM retained), STANDBY (only RTC/wakeup pins, RAM may be lost), SHUTDOWN/deep sleep (minimum leakage). Choose based on required wakeup latency and state preservation needs.

**Incorrect (using STANDBY when RAM data must be preserved):**

```c
/* WRONG: STANDBY mode loses RAM — application state destroyed */
void Power_EnterLowPower_Bad(void)
{
    /* Application has state in global variables that must survive sleep */
    g_engineState = ENGINE_STATE_IDLE;
    g_sensorCalibration = calibrated_value;

    /* STANDBY: RAM is not retained on most MCU families */
    PWR->CR1 |= PWR_CR1_PDDS;  /* Power-down deep sleep */
    SCB->SCR |= SCB_SCR_SLEEPDEEP_Msk;
    __WFI();

    /* After wakeup: g_engineState and g_sensorCalibration are GONE
     * MCU resets from Reset_Handler, not from here */
}
```

**Correct (mode selection based on requirements):**

```c
/* low_power.h — mode capabilities matrix */
/*
 * Mode        | Current  | Wakeup Latency | RAM     | RTC  | Wakeup Sources
 * ------------|----------|----------------|---------|------|----------------
 * SLEEP       | ~5 mA    | < 1 μs         | Yes     | Yes  | Any interrupt
 * STOP        | ~10 μA   | ~5 μs          | Yes     | Yes  | EXTI, RTC, comparator
 * STANDBY     | ~1 μA    | ~1 ms (reset)  | Partial | Yes  | Wakeup pins, RTC
 * SHUTDOWN    | ~100 nA  | ~5 ms (reset)  | No      | No   | Wakeup pins only
 *
 * Values are illustrative — consult your MCU datasheet.
 */

typedef enum {
    LP_MODE_SLEEP,     /* CPU WFI, peripherals running */
    LP_MODE_STOP,      /* All clocks stopped, RAM retained, SRAM powered */
    LP_MODE_STANDBY,   /* Backup domain only, wakeup is a reset */
    LP_MODE_SHUTDOWN   /* Minimum power, full reset on wakeup */
} LowPowerMode_t;
```

```c
/* low_power.c — safe mode entry with state preservation */

void LowPower_EnterSleep(void)
{
    /* SLEEP mode: CPU clock stopped, peripherals continue running.
     * Use when: waiting for next timer tick or interrupt with fast response. */
    SCB->SCR &= ~SCB_SCR_SLEEPDEEP_Msk;  /* Clear SLEEPDEEP bit */
    __DSB();
    __WFI();  /* Wait For Interrupt — CPU resumes on any enabled IRQ */
    /* Execution continues here immediately after interrupt */
}

void LowPower_EnterStop(void)
{
    /* STOP mode: All clocks stopped, RAM and registers retained.
     * Wakeup via EXTI lines, RTC alarm, or comparator.
     * Use when: ECU must retain state but can tolerate ~5μs wakeup. */

    /* Save peripheral state that may be lost */
    uint32_t savedRCC = RCC->CFGR;

    /* Configure wakeup sources before entering STOP */
    EXTI->IMR |= EXTI_IMR_IM11;   /* Enable CAN Rx wakeup on EXTI11 */
    EXTI->RTSR |= EXTI_RTSR_RT11; /* Rising edge trigger */

    /* Configure voltage regulator for low-power in STOP */
    PWR->CR1 |= PWR_CR1_LPMS_STOP1;  /* STOP1 with low-power regulator */

    SCB->SCR |= SCB_SCR_SLEEPDEEP_Msk;
    __DSB();
    __WFI();

    /* === Wakeup: execution continues here === */
    SCB->SCR &= ~SCB_SCR_SLEEPDEEP_Msk;

    /* Restore system clock — STOP mode defaults to HSI */
    SystemClock_Restore(savedRCC);
}

void LowPower_EnterStandby(void)
{
    /* STANDBY mode: Only backup domain powered. Wakeup causes reset.
     * Use when: long sleep (hours/days), state can be reconstructed. */

    /* Save critical data to backup registers (retained in STANDBY) */
    RTC->BKP0R = g_bootCount;
    RTC->BKP1R = g_wakeupReason;
    RTC->BKP2R = STANDBY_MARKER_VALUE;  /* Mark intentional standby */

    /* Enable wakeup pin */
    PWR->CSR |= PWR_CSR_EWUP1;  /* WKUP1 pin enables wakeup */

    /* Configure RTC alarm as alternative wakeup */
    RTC_SetAlarm(RTC_GetTime() + WAKEUP_INTERVAL_SEC);

    /* Clear wakeup flag */
    PWR->SCR |= PWR_SCR_CWUF;

    /* Enter STANDBY */
    PWR->CR1 |= PWR_CR1_PDDS;
    SCB->SCR |= SCB_SCR_SLEEPDEEP_Msk;
    __DSB();
    __WFI();

    /* Never reached — wakeup causes reset */
}
```

**Detecting wakeup reason after STANDBY reset:**

```c
/* Called at boot to determine if this is a cold start or STANDBY wakeup */
void Boot_CheckWakeupReason(void)
{
    if (PWR->CSR & PWR_CSR_SBF)
    {
        /* Woke from STANDBY — not a cold boot */
        PWR->SCR |= PWR_SCR_CSBF;  /* Clear standby flag */

        if (RTC->BKP2R == STANDBY_MARKER_VALUE)
        {
            /* Intentional standby wakeup — restore context from backup regs */
            g_bootCount = RTC->BKP0R;
            g_wakeupReason = RTC->BKP1R;
            App_ResumeFromStandby();
            return;
        }
    }

    /* Cold boot or unexpected reset */
    App_ColdStart();
}
```

**RAM retention in selective power domains:**

```c
/* Some MCUs offer partial RAM retention in STANDBY/STOP2 */
/* Place critical data in retained SRAM region */
__attribute__((section(".retained_ram")))
static uint32_t g_retainedState[16];

/* Linker script must map .retained_ram to SRAM2/backup SRAM */
/* MEMORY { RETAINED_RAM (rw) : ORIGIN = 0x20040000, LENGTH = 4K } */

void LowPower_SaveRetainedState(void)
{
    g_retainedState[0] = RETENTION_MAGIC;
    g_retainedState[1] = g_ecuState;
    g_retainedState[2] = g_errorCounter;
    __DSB();  /* Ensure writes complete before entering low-power */
}

bool LowPower_RestoreRetainedState(void)
{
    if (g_retainedState[0] != RETENTION_MAGIC)
    {
        return false;  /* RAM was not retained */
    }
    g_ecuState = g_retainedState[1];
    g_errorCounter = g_retainedState[2];
    g_retainedState[0] = 0U;  /* Clear magic to detect next reset type */
    return true;
}
```

**C++ abstraction for low-power modes:**

```cpp
class LowPowerManager
{
public:
    void Enter(LowPowerMode_t mode)
    {
        PreSleepHook();
        switch (mode)
        {
            case LP_MODE_SLEEP:   EnterSleep();   break;
            case LP_MODE_STOP:    EnterStop();     break;
            case LP_MODE_STANDBY: EnterStandby();  break;
            default: break;
        }
        PostWakeupHook();
    }

private:
    void PreSleepHook()
    {
        WDG_Suspend();
        save_context();
    }

    void PostWakeupHook()
    {
        restore_context();
        WDG_Resume();
        SystemCoreClockUpdate();
    }
};
```

Always issue `__DSB()` before `__WFI()` to ensure all pending memory writes complete before the CPU stops. After STOP mode wakeup, reconfigure the system clock — the MCU reverts to the default internal oscillator. Use backup registers (RTC domain) to pass data through STANDBY resets.

Reference: MCU Reference Manual — Power Control chapters; ISO 26262 Part 5 (Hardware — power modes); AUTOSAR SWS_Mcu (Mcu_SetMode)
