---
title: Node Simulation with State Machines
impact: MEDIUM
impactDescription: State machine-based simulation nodes produce realistic, maintainable ECU behavior
tags: capl, simulation, state-machine, node, canoe, ecu-simulation
---

## Node Simulation with State Machines

Design simulation nodes using state machines for realistic ECU behavior. A state machine makes the simulation's behavior explicit, traceable, and easy to extend with new states or fault injection.

**Incorrect (flat logic without state tracking):**

```capl
on timer cyclicTimer
{
    if (g_initialized && !g_faulted)
    {
        output(msg_Normal);
    }
    else if (g_faulted)
    {
        output(msg_Fault);
    }
    setTimer(cyclicTimer, 10);
}
```

**Correct (state machine with explicit transitions):**

```capl
variables
{
    enum SimState { INIT, RUNNING, FAULT, OFF };
    enum SimState currentState = INIT;
}

void TransitionTo(enum SimState newState)
{
    write("State transition: %d -> %d", currentState, newState);
    currentState = newState;
}

on timer cyclicTimer
{
    switch (currentState)
    {
        case INIT:
            SendInitMessages();
            TransitionTo(RUNNING);
            break;
        case RUNNING:
            SendNormalMessages();
            break;
        case FAULT:
            SendFaultMessages();
            break;
        case OFF:
            /* Send nothing */
            break;
    }
    setTimer(cyclicTimer, 10);
}
```

Log state transitions for debugging. Use the state machine to control which messages are sent, what signal values are used, and how the node responds to incoming messages. External events (panel buttons, diagnostic requests) trigger transitions via `TransitionTo()`.

Reference: Vector CANoe/CANalyzer CAPL Programming Guide — Simulation Nodes
