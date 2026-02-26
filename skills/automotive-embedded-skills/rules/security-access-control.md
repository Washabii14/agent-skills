---
title: Security Domain Access Control
impact: HIGH
impactDescription: Enforces freedom from interference between security domains
tags: security, access-control, domain, partition, mpu, iso-21434
---

## Security Domain Access Control

Enforce access control between security domains to prevent unauthorized cross-domain data flow. Safety-critical domains (braking, steering) must be isolated from non-critical domains (infotainment, connectivity) to maintain freedom from interference.

**Incorrect (no domain isolation):**

```c
/* Any module can access safety-critical data */
extern BrakeControl_t g_brakeData;

void Infotainment_DisplayBrakeStatus(void)
{
    g_brakeData.pressure = 0.0f;  /* Accidental or malicious write */
}
```

**Correct (MPU-enforced domain isolation):**

```c
/* ASIL D partition — protected memory region */
#pragma section ".ASIL_D_DATA"
static BrakeControl_t g_brakeData;
#pragma section

/* QM partition — separate memory region */
#pragma section ".QM_DATA"
static InfotainmentData_t g_infoData;
#pragma section

/* Cross-domain access only via controlled API */
Std_ReturnType SafetyGateway_GetBrakeStatus(float *pressure)
{
    if (pressure == NULL) { return E_NOT_OK; }

    OS_EnterCritical();
    *pressure = g_brakeData.pressure;
    OS_ExitCritical();

    return E_OK;
}
```

Use MPU (Memory Protection Unit) regions to enforce domain boundaries at hardware level. Configure the RTOS to assign different MPU regions to tasks of different ASIL levels.

Reference: ISO/SAE 21434:2021, ISO 26262 Part 6 — Freedom from interference
