---
title: A2L/ASAP2 Calibration Descriptions
impact: MEDIUM
impactDescription: Enables measurement and calibration tool interoperability
tags: integration, a2l, asap2, calibration, inca, canape, measurement
---

## A2L/ASAP2 Calibration Descriptions

Generate and maintain A2L files that accurately describe calibration parameters and measurement signals for tools like INCA, CANape. The A2L description must match the C source variable names, types, and memory addresses exactly.

**Incorrect (A2L out of sync with source):**

```c
/* C source: variable renamed */
float Cal_TargetPressure_bar;
```

```
/* A2L: still references old name — tool cannot find variable */
/begin CHARACTERISTIC
    CalParam_BoostPressure   /* Stale name */
    ...
/end CHARACTERISTIC
```

**Correct (A2L matches C source exactly):**

```
/* A2L parameter definition — must match C source exactly */
/begin CHARACTERISTIC
    CalParam_BoostPressureTarget  /* Name matches C variable */
    "Target boost pressure"
    VALUE
    0x00080100                     /* Address from linker map */
    DAMOS_FW                       /* Deposit format */
    0.0                            /* Max diff */
    CalParam_Conv_Pressure         /* Conversion formula */
    0.0                            /* Lower limit */
    3.0                            /* Upper limit (bar) */
/end CHARACTERISTIC
```

Automate A2L generation from linker map files and source annotations to prevent manual sync errors. Validate A2L against the ELF/MAP file in CI to catch address mismatches before release.

Reference: ASAM MCD-2MC (A2L/ASAP2) — Measurement and Calibration Data Exchange
