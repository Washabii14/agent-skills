---
title: Always Include Default Case in Switch
impact: MEDIUM
impactDescription: catches unexpected values, defensive programming
tags: misra, switch, default, defensive-programming, state-machine, safety
---

## Always Include Default Case in Switch

Every switch statement must have a default case to handle unexpected values, especially for values received from external interfaces.

**Incorrect (missing default):**

```c
void HandleState(SystemState_t state)
{
    switch (state)
    {
        case STATE_INIT:    DoInit();    break;
        case STATE_RUNNING: DoRunning(); break;
        case STATE_STOP:    DoStop();    break;
    }
}
```

**Correct (with default and error handling):**

```c
void HandleState(SystemState_t state)
{
    switch (state)
    {
        case STATE_INIT:    DoInit();    break;
        case STATE_RUNNING: DoRunning(); break;
        case STATE_STOP:    DoStop();    break;
        default:
            ReportError(ERR_INVALID_STATE);
            EnterSafeState();
            break;
    }
}
```

Reference: MISRA C:2012 Rule 16.4 — Every switch statement shall have a default label.
