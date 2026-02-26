---
title: Error Detection and Plausibility Checks
impact: HIGH
impactDescription: detects sensor/actuator failures
tags: safety, iso-26262, plausibility, sensor-validation, error-detection
---

## Error Detection and Plausibility Checks

Implement plausibility checks on sensor readings and actuator feedback to detect hardware failures. Validate that values are within physical range and that the rate of change does not exceed what is physically possible (gradient check).

**Incorrect (no validation on sensor input):**

```c
void ReadTemperature(float *temp)
{
    *temp = ADC_ReadChannel(TEMP_CHANNEL) * TEMP_SCALE;
}
```

**Correct (plausibility-checked sensor reading):**

```c
typedef struct
{
    float currentValue;
    float previousValue;
    float maxDeltaPerCycle;
    float minValid;
    float maxValid;
} PlausibilityCheck_t;

boolean IsPlausible(PlausibilityCheck_t *check, float newValue)
{
    float delta;

    if ((newValue < check->minValid) || (newValue > check->maxValid))
    {
        return FALSE;  /* Out of physical range */
    }

    delta = newValue - check->previousValue;
    if (delta < 0.0f) { delta = -delta; }

    if (delta > check->maxDeltaPerCycle)
    {
        return FALSE;  /* Gradient too steep — likely sensor fault */
    }

    check->previousValue = check->currentValue;
    check->currentValue = newValue;
    return TRUE;
}
```

Check both absolute range and rate-of-change. On failure, substitute a default safe value and raise a DTC.

Reference: ISO 26262 Part 6 — Software unit design and implementation
