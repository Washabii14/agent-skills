---
title: Separate Configuration from Logic
impact: MEDIUM
impactDescription: Configuration separation enables ECU variant management and runtime calibration
tags: arch, configuration, calibration, variant-management, separation-of-concerns
---

## Separate Configuration from Logic

Separate calibration/configuration data from application logic for ECU variant management. Configuration data should be in dedicated files/structures, allowing different parameter sets per ECU variant without changing application code.

**Incorrect (hardcoded parameters in logic):**

```c
void SensorTask_10ms(void)
{
    float raw = ReadSensorRaw();
    float filtered = ApplyFilter(raw, 0.85f);       /* Magic number */

    if ((filtered < -40.0f) || (filtered > 150.0f))  /* Magic bounds */
    {
        filtered = 25.0f;                             /* Magic default */
    }
}
```

**Correct (configuration struct separate from logic):**

```c
/* Configuration — separate file, calibratable */
const SensorConfig_t g_sensorConfig =
{
    .samplingRateMs  = 10U,
    .filterCoeff     = 0.85f,
    .minValid        = -40.0f,
    .maxValid        = 150.0f,
    .defaultValue    = 25.0f,
};

/* Logic — uses configuration */
void SensorTask_10ms(void)
{
    float raw = ReadSensorRaw();
    float filtered = ApplyFilter(raw, g_sensorConfig.filterCoeff);

    if (!IsInRange(filtered, g_sensorConfig.minValid,
                   g_sensorConfig.maxValid))
    {
        filtered = g_sensorConfig.defaultValue;
    }
}
```

Place configuration structures in dedicated `_cfg.c` / `_cfg.h` files. Mark calibratable parameters with compiler-specific section attributes for A2L/calibration tool integration. Different ECU variants only swap the configuration file.

Reference: AUTOSAR ECU Configuration — Parameter definition patterns
