---
title: DBC/ARXML and Code Synchronization
impact: HIGH
impactDescription: Prevents runtime signal mismatches between database and code
tags: integration, dbc, arxml, synchronization, can, signals, code-generation
---

## DBC/ARXML and Code Synchronization

Keep DBC (CAN database) / ARXML definitions synchronized with source code signal structures to prevent runtime communication failures. A mismatch between database signal layout and code extraction logic causes silent data corruption on the bus.

**Incorrect (manual signal extraction, not synced with DBC):**

```c
uint16_t GetSpeed(const uint8_t data[8])
{
    return ((uint16_t)data[2] << 8) | data[3];  /* Assumes byte 2-3, but DBC says byte 0-1 */
}
```

**Correct (generated from DBC/ARXML, matches database):**

```c
/* Auto-generated from Vehicle.dbc — DO NOT EDIT MANUALLY */
typedef struct
{
    uint16_t VehicleSpeed_Raw;    /* Bits 0-15, factor 0.01, offset 0 */
    uint8_t  GearPosition;       /* Bits 16-19 */
    uint8_t  BrakeActive;        /* Bit 20 */
} VehicleStatus_Msg_t;

uint16_t GetVehicleSpeed_kmph(const VehicleStatus_Msg_t *msg)
{
    return (uint16_t)(msg->VehicleSpeed_Raw * VEHICLE_SPEED_FACTOR
                      + VEHICLE_SPEED_OFFSET);
}
```

Generate message structs and signal access functions from DBC/ARXML using tools (Vector CANdb++, comtypes, cantools). Add a CI check that regenerates code from the database and fails if the generated code differs from what is committed.

Reference: AUTOSAR COM specification, ISO 11898-1:2015
