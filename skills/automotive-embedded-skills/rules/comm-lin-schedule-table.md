---
title: LIN Schedule Table and Response Handling
impact: MEDIUM
impactDescription: correct LIN master/slave communication
tags: comm, lin, schedule-table, master-slave, diagnostics
---

## LIN Schedule Table and Response Handling

Implement LIN schedule tables with proper header/response separation and error detection. The LIN master controls the schedule — each slot defines a frame ID, direction, data length, and timing.

**Incorrect (ad-hoc LIN communication without schedule):**

```c
void LIN_SendFrame(uint8_t id, uint8_t *data)
{
    LIN_Transmit(id, data, 8U);  /* No schedule, fixed length */
}
```

**Correct (structured schedule table):**

```c
typedef struct
{
    uint8_t frameId;
    uint8_t direction;     /* LIN_TX or LIN_RX */
    uint8_t dataLength;
    uint16_t slotTimeMs;
} LinScheduleEntry_t;

static const LinScheduleEntry_t g_linSchedule[] =
{
    { 0x10U, LIN_TX, 8U, 10U },  /* Master request */
    { 0x11U, LIN_RX, 4U, 10U },  /* Slave response */
    { 0x3CU, LIN_TX, 8U, 10U },  /* Diagnostic request (MasterReq) */
    { 0x3DU, LIN_RX, 8U, 10U },  /* Diagnostic response (SlaveResp) */
};
```

Key LIN considerations:
- Respect slot timing per LIN specification (max frame time depends on baud rate)
- Handle no-response errors (slave not responding within slot)
- Use protected identifier (PID) with parity bits
- Reserve frame IDs 0x3C/0x3D for diagnostic (MasterReq/SlaveResp)

Reference: LIN 2.2A Specification; AUTOSAR SWS LIN Interface
