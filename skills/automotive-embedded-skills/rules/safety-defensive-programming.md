---
title: Defensive Programming at Module Boundaries
impact: HIGH
impactDescription: catches interface violations early
tags: safety, iso-26262, defensive-programming, input-validation, module-boundaries
---

## Defensive Programming at Module Boundaries

Validate all inputs at public function boundaries. Never trust data from external modules or communication interfaces. For safety-relevant actuator commands, check range, freshness (message counter), and plausibility before applying.

**Incorrect (trusting external data):**

```c
void ApplyTorqueRequest(const TorqueMsg_t *msg)
{
    SetMotorTorque(msg->requestedTorque);
}
```

**Correct (defensive validation):**

```c
void ApplyTorqueRequest(const TorqueMsg_t *msg)
{
    if (msg == NULL)
    {
        EnterSafeState();
        return;
    }

    if ((msg->requestedTorque < TORQUE_MIN) ||
        (msg->requestedTorque > TORQUE_MAX))
    {
        ReportError(ERR_TORQUE_RANGE);
        SetMotorTorque(TORQUE_SAFE_VALUE);
        return;
    }

    if (msg->messageCounter == g_lastMsgCounter)
    {
        ReportError(ERR_MSG_STALE);
        return;
    }
    g_lastMsgCounter = msg->messageCounter;

    SetMotorTorque(msg->requestedTorque);
}
```

Always validate: NULL pointers, range limits, message counters/freshness, and CRC where applicable.

Reference: ISO 26262 Part 6 — Software architectural design safety mechanisms
