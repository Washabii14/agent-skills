---
title: Table-Driven State Machines
impact: MEDIUM
impactDescription: Table-driven state machines are scalable, verifiable, and resistant to missed transitions
tags: arch, state-machine, table-driven, design-pattern, scalability, verification
---

## Table-Driven State Machines

Use table-driven state machines instead of nested if/else or large switch statements. A transition table makes all valid transitions explicit and verifiable — missing transitions are immediately visible.

**Incorrect (ad-hoc state handling with nested conditions):**

```c
void HandleEvent(EventType_t event)
{
    if (g_state == STATE_INIT && event == EVT_POWER_ON)
    {
        g_state = STATE_RUNNING;
    }
    else if (g_state == STATE_RUNNING && event == EVT_FAULT)
    {
        g_state = STATE_FAULT;
    }
    /* Easy to miss transitions as states grow */
}
```

**Correct (table-driven with explicit transition mapping):**

```c
typedef Std_ReturnType (*StateHandler_t)(const Event_t *event,
                                          SystemState_t *nextState);

typedef struct
{
    SystemState_t currentState;
    EventType_t   event;
    StateHandler_t handler;
} StateTransition_t;

static const StateTransition_t g_transitions[] =
{
    { STATE_INIT,    EVT_POWER_ON,   Handler_InitToRun     },
    { STATE_RUNNING, EVT_FAULT,      Handler_RunToFault    },
    { STATE_RUNNING, EVT_SHUTDOWN,   Handler_RunToShutdown },
    { STATE_FAULT,   EVT_RECOVERY,   Handler_FaultToInit   },
};
```

The transition table can be reviewed against the design specification. Adding new states or events means adding rows, not modifying control flow. Static analysis tools can verify table completeness.

Reference: ISO 26262 Part 6 — Software detailed design, AUTOSAR State Management
