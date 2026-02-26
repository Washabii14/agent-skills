---
title: State Machine Integrity Protection
impact: HIGH
impactDescription: prevents illegal state transitions
tags: safety, iso-26262, state-machine, integrity, memory-corruption
---

## State Machine Integrity Protection

Protect state machine transitions from corruption caused by memory faults or software bugs. Use magic values to detect RAM corruption of state variables, and validate that transitions follow the allowed transition table.

**Incorrect (unprotected state variable):**

```c
static SystemState_t g_state;

void SM_SetState(SystemState_t newState)
{
    g_state = newState;
}
```

**Correct (integrity-protected state machine):**

```c
#define STATE_MAGIC (0xA5A5U)

typedef struct
{
    uint16_t magic;
    SystemState_t currentState;
    SystemState_t previousState;
    uint32_t transitionCount;
} ProtectedStateMachine_t;

Std_ReturnType SM_Transition(ProtectedStateMachine_t *sm,
                              SystemState_t newState)
{
    if (sm->magic != STATE_MAGIC)
    {
        EnterSafeState();
        return E_NOT_OK;
    }

    if (!IsTransitionValid(sm->currentState, newState))
    {
        ReportError(ERR_INVALID_TRANSITION);
        return E_NOT_OK;
    }

    sm->previousState = sm->currentState;
    sm->currentState = newState;
    sm->transitionCount++;

    return E_OK;
}
```

The magic value detects RAM corruption. The transition validation table prevents illegal jumps caused by software bugs or external interference.

Reference: ISO 26262 Part 6 — Control flow monitoring, Data integrity
