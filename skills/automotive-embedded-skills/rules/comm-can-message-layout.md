---
title: CAN Message Layout and DBC Conventions
impact: MEDIUM
impactDescription: ensures interoperability and maintainability
tags: comm, can, dbc, signal-layout, message-structure
---

## CAN Message Layout and DBC Conventions

Follow DBC-defined signal layout. Use signal names from the database, not magic byte offsets. Code generated from DBC/ARXML ensures signal packing, byte order, scaling, and offset are always consistent with the network design.

**Incorrect (magic numbers for signal extraction):**

```c
uint16_t GetSpeed(const uint8_t data[8])
{
    return ((uint16_t)data[2] << 8) | data[3];  /* What signal is this? */
}
```

**Correct (named signal access aligned with DBC):**

```c
uint16_t GetVehicleSpeed_kmph(const VehicleSpeed_Msg_t *msg)
{
    uint16_t rawSpeed = msg->VehicleSpeed_Raw;
    return (uint16_t)(rawSpeed * VEHICLE_SPEED_FACTOR + VEHICLE_SPEED_OFFSET);
}
```

Use auto-generated message structs from DBC/ARXML tooling. Manually mapping byte offsets is error-prone, especially with mixed endianness (Motorola/Intel byte order) signals.

Reference: ISO 11898-1; AUTOSAR COM module — Signal packing and unpacking
