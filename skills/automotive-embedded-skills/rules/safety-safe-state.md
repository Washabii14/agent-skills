---
title: Always Define and Reach Safe State on Failure
impact: CRITICAL
impactDescription: fundamental safety requirement
tags: safety, iso-26262, safe-state, failure-handling, actuator-control
---

## Always Define and Reach Safe State on Failure

Every safety-relevant module must define a safe state and transition to it on any detected failure. The safe state must disable or limit actuator outputs, set known-safe default values, notify the diagnostic system, and be reachable from any module state.

**Incorrect (no safe state defined, error only logged):**

```c
void HandleCriticalError(void)
{
    LogError(ERR_CRITICAL);
}
```

**Correct (comprehensive safe state entry):**

```c
void EnterSafeState(void)
{
    DisableActuators();
    SetDefaultOutputs();
    NotifyDiagnostic(DTC_SAFE_STATE_ENTERED);
    SetSystemState(STATE_SAFE);
}
```

The safe state must be:
- **Defined** for every safety-relevant module at design time
- **Reachable** from every state in the module's state machine
- **Deterministic** in its outputs (actuators disabled or at known-safe values)
- **Persistent** until explicitly released by a recovery procedure

Call `EnterSafeState()` on: watchdog timeout, redundancy mismatch, CRC failure, invalid state transitions, or any unrecoverable fault.

Reference: ISO 26262 Part 3 — Concept phase (safe state definition); Part 6 — Software safety requirements
