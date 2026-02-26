---
title: Redundant Checks for Critical Control Paths
impact: CRITICAL
impactDescription: ASIL C/D requirement for diverse redundancy
tags: safety, iso-26262, redundancy, asil-c, asil-d, critical-control
---

## Redundant Checks for Critical Control Paths

For ASIL C and D, critical control outputs must be validated by diverse redundant checks. Use two independently developed calculation paths and compare results before applying actuator outputs.

**Incorrect (single calculation path for safety-critical output):**

```c
void SetBrakePressure(float requested)
{
    float pressure = CalculateBrakePressure(requested);
    HwDriver_SetBrakePressure(pressure);
}
```

**Correct (diverse redundant calculation with comparison):**

```c
void SetBrakePressure(float requested)
{
    /* Primary calculation */
    float primary = CalculateBrakePressure_Primary(requested);

    /* Diverse redundant calculation */
    float secondary = CalculateBrakePressure_Secondary(requested);

    float diff = primary - secondary;
    if (diff < 0.0f) { diff = -diff; }

    if (diff > BRAKE_TOLERANCE)
    {
        ReportError(ERR_BRAKE_REDUNDANCY);
        ApplyEmergencyBrake();
        return;
    }

    HwDriver_SetBrakePressure(primary);
}
```

The two calculation paths should use diverse algorithms or be developed independently. If results disagree beyond tolerance, enter safe state.

Reference: ISO 26262 Part 6, Table 5 — Methods for software architectural design
