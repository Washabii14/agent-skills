---
title: Environment Variables for Panel Interaction
impact: LOW
impactDescription: Environment variables are the standard bridge between CAPL logic and CANoe panels
tags: capl, environment-variable, panel, canoe, ui, interaction
---

## Environment Variables for Panel Interaction

Use environment variables as the bridge between CAPL and CANoe panels. Environment variables allow the user to interact with the simulation through panel controls (sliders, buttons, inputs) while keeping CAPL logic decoupled from the UI.

**Incorrect (hardcoded values without panel control):**

```capl
variables
{
    int targetSpeed = 100;  /* Cannot be changed from panel at runtime */
}
```

**Correct (environment variable linked to panel control):**

```capl
on envVar EnvSetTargetSpeed
{
    int speed = getValue(this);
    if ((speed >= 0) && (speed <= 250))
    {
        g_targetSpeed = speed;
        UpdateSpeedDisplay();
    }
}
```

Validate environment variable values in the `on envVar` handler before using them. Define sensible defaults and range checks. Use `putValue()` to update environment variables from CAPL to reflect state back to the panel.

```capl
void UpdateSpeedDisplay()
{
    putValue(EnvActualSpeed, g_actualSpeed);
    putValue(EnvSpeedError, g_targetSpeed - g_actualSpeed);
}
```

Reference: Vector CANoe/CANalyzer CAPL Programming Guide — Environment Variables
