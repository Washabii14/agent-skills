---
title: Signal Access via Database Symbols
impact: MEDIUM
impactDescription: Raw byte access breaks when DBC layout changes and makes code unreadable
tags: capl, signal, database, dbc, symbolic-access, canoe
---

## Signal Access via Database Symbols

Always access signals via database symbolic names, never via raw byte offsets. Symbolic access automatically applies scaling, offset, and byte order from the DBC definition.

**Incorrect (raw byte access with magic numbers):**

```capl
on message 0x100
{
    byte engineRpm_h = this.byte(0);
    byte engineRpm_l = this.byte(1);
    int rpm = (engineRpm_h << 8) | engineRpm_l;
}
```

**Correct (symbolic signal access via database):**

```capl
on message EngineData
{
    int rpm = this.EngineRPM.phys;
    write("Engine RPM: %d", rpm);
}
```

Using `.phys` returns the physical value with the DBC-defined factor and offset already applied. This ensures values stay correct when the database layout changes. Use message names (not hex IDs) in `on message` handlers to tie them to the database definition.

Reference: Vector CANoe/CANalyzer CAPL Programming Guide — Signal Access
