---
title: Partial Networking and Selective Wakeup
impact: MEDIUM
impactDescription: Incorrect partial networking configuration causes unnecessary ECU wakeups, draining the 12V battery, or missed critical wakeup frames
tags: power, partial-networking, can, transceiver, selective-wakeup, wuf, pnc, comm, nm
---

## Partial Networking and Selective Wakeup

Partial networking allows ECUs to remain in low-power mode while the CAN bus is active, waking only on specific frames. CAN transceivers with selective wakeup (PN) filter incoming frames against a configured Wake-Up Frame (WUF) pattern. ComM Partial Network Clusters (PNC) coordinate which ECUs participate in which communication clusters.

**Incorrect (ECU wakes on every CAN frame, PN filter not configured):**

```c
/* WRONG: Transceiver wakes on any bus activity */
void CanTrcv_Init_Bad(void)
{
    CanTrcv_SetOpMode(CANTRCV_TRCV_MODE_NORMAL);
    /* No WUF configured — ECU wakes on every CAN frame on the bus */
    /* Battery drain: ECU cycles sleep/wake thousands of times per second */
}

/* WRONG: ComM always requests FULL_COMMUNICATION */
void App_Init_Bad(void)
{
    ComM_RequestComMode(COMM_CHANNEL_CAN, COMM_FULL_COMMUNICATION);
    /* Never releases — ECU can never enter partial networking sleep */
}
```

**Correct (selective wakeup with WUF and PNC configuration):**

```c
/* CAN transceiver selective wakeup configuration */
typedef struct {
    uint32_t wufId;            /* CAN ID that triggers wakeup */
    uint32_t wufIdMask;        /* Mask for ID matching (0 = don't care) */
    uint8_t  wufDlc;           /* Expected DLC */
    uint8_t  wufDataMask[8];   /* Byte-level mask for data matching */
    uint8_t  wufDataPattern[8]; /* Expected data pattern */
} CanTrcv_PnConfig_t;

const CanTrcv_PnConfig_t CanTrcv_PnConfig = {
    .wufId       = 0x510U,         /* NM wakeup frame ID */
    .wufIdMask   = 0x7FFU,         /* Exact match */
    .wufDlc      = 8U,
    .wufDataMask    = {0xFF, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00},
    .wufDataPattern = {0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00},
    /* Wakes only when frame 0x510, byte0 = 0x01 (this ECU's PNC bit) */
};

void CanTrcv_Init_WithPN(void)
{
    /* Configure selective wakeup filter in transceiver */
    CanTrcv_SetPnConfiguration(&CanTrcv_PnConfig);

    /* Enable partial networking mode */
    CanTrcv_SetOpMode(CANTRCV_TRCV_MODE_STANDBY);
    CanTrcv_EnablePnWakeup(TRUE);
}
```

**ComM Partial Network Cluster (PNC) management:**

```c
/* PNC configuration — maps application features to network clusters */
/*
 * PNC_CLIMATE  = bit 0 → Climate control ECU, HVAC actuator ECU
 * PNC_LIGHTING = bit 1 → Body controller, headlamp ECU
 * PNC_DIAG     = bit 7 → All ECUs (diagnostic always wakes)
 *
 * NM message carries PNC bit vector — only ECUs whose PNC bits
 * are set will wake up.
 */

#define PNC_CLIMATE    0U
#define PNC_LIGHTING   1U
#define PNC_INFOTAINMENT 2U
#define PNC_DIAG       7U

/* SWC requests communication for its PNC */
void ClimateCtrl_Activate(void)
{
    /* Request only the climate PNC — other ECUs stay asleep */
    ComM_RequestComMode(COMM_PNC_CLIMATE, COMM_FULL_COMMUNICATION);
    /* NM will set bit 0 in outgoing NM frame's PNC vector */
}

void ClimateCtrl_Deactivate(void)
{
    /* Release climate PNC — if no other SWC needs it, ECU can sleep */
    ComM_RequestComMode(COMM_PNC_CLIMATE, COMM_NO_COMMUNICATION);
}
```

**Network-triggered wakeup flow:**

```c
/* Complete PN wakeup sequence */
/*
 * 1. ECU is in STANDBY, transceiver filtering CAN frames
 * 2. Another ECU sends NM frame with this ECU's PNC bit set
 * 3. Transceiver matches WUF pattern → asserts WAKE pin
 * 4. MCU wakes from STOP/STANDBY mode
 * 5. EcuM detects wakeup source = CAN transceiver
 * 6. Wakeup validation: read transceiver wakeup reason
 * 7. If valid WUF match → proceed to RUN
 * 8. If bus activity but no WUF match → spurious, go back to sleep
 */

void EcuM_CheckWakeup_CanPN(void)
{
    CanTrcv_TrcvWakeupReasonType reason;
    CanTrcv_GetTrcvWakeupReason(CANTRCV_CHANNEL_0, &reason);

    switch (reason)
    {
        case CANTRCV_WU_BY_PIN:
            /* Hardware wakeup pin — always valid */
            EcuM_ValidateWakeupEvent(ECUM_WKSOURCE_POWER);
            break;

        case CANTRCV_WU_BY_BUS_WUF:
            /* Selective wakeup frame matched — valid PN wakeup */
            EcuM_ValidateWakeupEvent(ECUM_WKSOURCE_CAN_PN);
            break;

        case CANTRCV_WU_BY_BUS:
            /* General bus activity, no WUF match — spurious for PN ECU */
            /* Do NOT validate — EcuM will return to sleep */
            break;

        default:
            break;
    }
}
```

**Transceiver SPI communication for PN configuration (bare-metal):**

```c
/* Many automotive CAN transceivers (TJA1145, TLE9263) use SPI for
 * PN configuration. Configure BEFORE entering standby mode. */
void CanTrcv_ConfigurePN_SPI(void)
{
    /* Write WUF ID register */
    SPI_WriteReg(CANTRCV_REG_CAN_ID, (uint8_t)(CanTrcv_PnConfig.wufId >> 8));
    SPI_WriteReg(CANTRCV_REG_CAN_ID + 1, (uint8_t)(CanTrcv_PnConfig.wufId));

    /* Write WUF data mask and pattern */
    for (uint8_t i = 0U; i < 8U; i++)
    {
        SPI_WriteReg(CANTRCV_REG_DATA_MASK_0 + i, CanTrcv_PnConfig.wufDataMask[i]);
        SPI_WriteReg(CANTRCV_REG_DATA_0 + i, CanTrcv_PnConfig.wufDataPattern[i]);
    }

    /* Enable PN and enter standby */
    SPI_WriteReg(CANTRCV_REG_MODE, CANTRCV_MODE_STANDBY | CANTRCV_PN_ENABLE);
}
```

Verify WUF configuration against the network design document (DBC/ARXML). PNC bit assignments must be consistent across all ECUs in the vehicle. Test spurious wakeup rejection — a misconfigured mask can cause either missed wakeups or constant false wakeups.

Reference: AUTOSAR SWS_ComM (Partial Network Clusters); AUTOSAR SWS_CanTrcv (Selective Wakeup); ISO 11898-2 (CAN physical layer wakeup)
