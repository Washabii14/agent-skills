---
title: Message Handler Structure
impact: MEDIUM
impactDescription: Structured handlers improve readability, prevent processing of own transmitted messages
tags: capl, message-handler, canoe, simulation, can
---

## Message Handler Structure

Structure message handlers for clarity, with early returns for invalid conditions. Always check message direction to avoid processing your own transmissions.

**Incorrect (monolithic handler with no direction check):**

```capl
on message EngineStatus
{
    /* 50+ lines of processing mixed together */
}
```

**Correct (structured handler with direction filter and delegation):**

```capl
on message EngineStatus
{
    if (this.dir != rx)
        return;

    UpdateEngineStatus(this);
    CheckEngineLimits(this);
    UpdatePanel();
}
```

Delegate complex logic to named functions for readability. Check `this.dir` to distinguish received messages (`rx`) from transmitted ones. Keep handlers focused on dispatching — not on implementing business logic inline.

Reference: Vector CANoe/CANalyzer CAPL Programming Guide
