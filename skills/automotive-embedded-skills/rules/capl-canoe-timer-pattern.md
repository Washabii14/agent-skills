---
title: Timer Patterns
impact: MEDIUM
impactDescription: Incorrect timer usage causes missed events or unintended cyclic behavior
tags: capl, timer, cyclic, one-shot, canoe, timing
---

## Timer Patterns

Use timers correctly for cyclic and one-shot operations. Re-arm a timer inside its handler for cyclic behavior; omit re-arming for one-shot behavior.

**Incorrect (ambiguous timer pattern):**

```capl
on timer myTimer
{
    DoWork();
    /* Is this cyclic or one-shot? Unclear without re-arm */
}
```

**Correct (explicit cyclic and one-shot patterns):**

```capl
variables
{
    msTimer cyclicTimer;
    msTimer debounceTimer;
    int timerPeriodMs = 100;
}

on start
{
    setTimer(cyclicTimer, timerPeriodMs);
}

on timer cyclicTimer
{
    SendCyclicMessage();
    setTimer(cyclicTimer, timerPeriodMs);  /* Re-arm for cyclic */
}

on timer debounceTimer
{
    /* One-shot: no re-arm */
    ProcessDebouncedInput();
}
```

For cyclic timers, always re-arm at the end of the handler. For one-shot timers, do not re-arm — document the intent. Use `cancelTimer()` to stop a cyclic timer when no longer needed.

Reference: Vector CANoe/CANalyzer CAPL Programming Guide
