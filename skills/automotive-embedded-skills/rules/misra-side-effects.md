---
title: No Side Effects in Conditional Expressions
impact: MEDIUM
impactDescription: prevents order-dependent bugs
tags: misra, side-effects, conditional, evaluation-order, safety, logic
---

## No Side Effects in Conditional Expressions

Conditional expressions shall not contain side effects. The evaluation order of operands is often implementation-defined, leading to unpredictable behavior.

**Incorrect (side effect in condition):**

```c
if ((ReadSensor(&value) == E_OK) && (value > threshold++))
{
    TriggerAlarm();
}
```

**Correct (separate side effects from conditions):**

```c
Std_ReturnType readResult = ReadSensor(&value);
if (readResult == E_OK)
{
    if (value > threshold)
    {
        threshold++;
        TriggerAlarm();
    }
}
```

Reference: MISRA C:2012 Rule 13.5 — The right hand operand of a logical && or || shall not contain persistent side effects.
