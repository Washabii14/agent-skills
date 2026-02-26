---
title: Minimize Interrupt Latency
impact: HIGH
impactDescription: preserves system responsiveness
tags: realtime, interrupt, latency, isr, critical-section
---

## Minimize Interrupt Latency

Keep ISR execution time to a minimum. Read hardware, set flags, return. Long ISRs increase interrupt latency for all other interrupts, which can cause missed deadlines in higher-priority tasks and lost hardware events.

**Incorrect (heavy processing in ISR):**

```c
void ADC_CompleteISR(void)
{
    uint16_t raw = ADC_ReadResult();
    float voltage = (float)raw * ADC_REF_VOLTAGE / ADC_MAX_VALUE;
    float filtered = ApplyKalmanFilter(voltage);  /* Heavy math */
    CheckLimits(filtered);
    UpdateDisplay(filtered);  /* I/O in ISR */
    ADC_ClearFlag();
}
```

**Correct (minimal ISR with deferred processing):**

```c
void ADC_CompleteISR(void)
{
    g_adcRawValue = ADC_ReadResult();
    g_adcDataReady = TRUE;
    ADC_ClearFlag();
}

void Task_1ms(void)
{
    if (g_adcDataReady)
    {
        g_adcDataReady = FALSE;
        float voltage = ConvertToVoltage(g_adcRawValue);
        float filtered = ApplyKalmanFilter(voltage);
        CheckLimits(filtered);
    }
}
```

Guidelines for low interrupt latency:
- Keep critical sections (interrupts disabled) as short as possible
- Never call blocking functions from ISR context
- Use lock-free data structures (ring buffers, flags) for ISR-to-task communication
- Profile ISR execution time and set a hard budget (e.g., <10us for 1ms tasks)

Reference: AUTOSAR OS — Category 1/2 ISR design; ARM Cortex-M NVIC programming guidelines
