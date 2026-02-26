---
title: MISRA Control Flow — Switch, Goto, Unreachable Code, Single Exit
impact: HIGH
impactDescription: enforces deterministic control flow and prevents dead/unreachable code paths
tags: misra, control-flow, switch, goto, unreachable-code, single-exit, break, return, safety
---

## MISRA Control Flow — Switch, Goto, Unreachable Code, Single Exit

Predictable control flow is essential for testability and WCET analysis. These rules prevent spaghetti code, ensure all switch paths are handled, and guarantee every code path is reachable and tested.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 15.1 | Advisory | The goto statement should not be used |
| 15.2 | Required | goto label shall be declared in the same or enclosing block |
| 15.3 | Required | goto shall reference a label in an enclosing block (forward only) |
| 15.4 | Advisory | At most one break/goto to terminate iteration |
| 15.5 | Advisory | Function should have a single point of exit at the end |
| 15.6 | Required | Body of iteration/selection shall be a compound statement |
| 15.7 | Required | All if…else if constructs shall be terminated with else |
| 16.1 | Required | All switch statements shall be well-formed |
| 16.2 | Required | Switch label shall only appear at top level of compound statement |
| 16.3 | Required | Every switch clause shall be terminated (break/return/fallthrough comment) |
| 16.4 | Required | Every switch shall have a default label |
| 16.5 | Required | Default label shall appear as either first or last in switch |
| 16.6 | Required | Every switch shall have at least two clauses |
| 16.7 | Required | Switch expression shall not have boolean essential type |

MISRA C++:2023 overlap: Rules 9.4.1 (switch well-formed), 9.4.2 (non-boolean switch), 9.5.1 (if…else), 0.1.1–0.1.2 (unreachable code).

---

### Top Violations and Patterns

**Rule 16.3 — Unterminated Switch Clause (most common)**

Incorrect (fallthrough without annotation):

```c
switch (msg_type) {
    case MSG_REQUEST:
        PrepareResponse(msg);
    case MSG_NOTIFY:           /* implicit fallthrough — bug or intentional? */
        SendMessage(msg);
        break;
    default:
        break;
}
```

Correct (explicit termination on every clause):

```c
switch (msg_type) {
    case MSG_REQUEST:
        PrepareResponse(msg);
        SendMessage(msg);
        break;

    case MSG_NOTIFY:
        SendMessage(msg);
        break;

    default:
        Dem_ReportError(DEM_INVALID_MSG);
        break;
}
```

C++ with `[[fallthrough]]`:

```cpp
switch (msg_type) {
    case MsgType::kRequest:
        PrepareResponse(msg);
        [[fallthrough]];   // explicit annotation — MISRA C++:2023 permits this
    case MsgType::kNotify:
        SendMessage(msg);
        break;
    default:
        Dem_ReportError(DemEventId::kInvalidMsg);
        break;
}
```

---

**Rule 15.7 — Else Termination for if…else if Chains**

Incorrect (missing final else):

```c
if (speed > SPEED_HIGH) {
    SetMode(MODE_SPORT);
} else if (speed > SPEED_LOW) {
    SetMode(MODE_NORMAL);
}
/* what if speed <= SPEED_LOW? silent pass-through */
```

Correct (terminated with else):

```c
if (speed > SPEED_HIGH) {
    SetMode(MODE_SPORT);
} else if (speed > SPEED_LOW) {
    SetMode(MODE_NORMAL);
} else {
    SetMode(MODE_ECO);
}
```

---

**Rule 15.6 — Compound Statements Required**

Incorrect (bare statement after if):

```c
if (error_flag)
    ReportError(ERR_GENERAL);
```

Correct (always use braces):

```c
if (error_flag) {
    ReportError(ERR_GENERAL);
}
```

---

**Rule 15.5 — Single Point of Exit**

Incorrect (multiple returns):

```c
Std_ReturnType ValidateFrame(const Frame_t *frame) {
    if (frame == NULL) { return E_NOT_OK; }
    if (frame->len > MAX_LEN) { return E_NOT_OK; }
    if (CalcCrc(frame) != frame->crc) { return E_NOT_OK; }
    return E_OK;
}
```

Correct (single exit):

```c
Std_ReturnType ValidateFrame(const Frame_t *frame) {
    Std_ReturnType result = E_NOT_OK;

    if (frame != NULL) {
        if (frame->len <= MAX_LEN) {
            if (CalcCrc(frame) == frame->crc) {
                result = E_OK;
            }
        }
    }

    return result;
}
```

---

**Rule 15.1 — No goto (permitted cleanup pattern)**

Incorrect (backward jump):

```c
retry:
    status = TrySend(msg);
    if (status != E_OK) { goto retry; }  /* backward goto — loop in disguise */
```

Correct (forward-only cleanup — see also misra-no-goto.md):

```c
Std_ReturnType ProcessMsg(const uint8_t *data, uint16_t len) {
    Std_ReturnType ret = E_NOT_OK;
    Buffer_t *buf = BufPool_Alloc();
    if (buf == NULL) { goto cleanup; }

    if (Decode(buf, data, len) != E_OK) { goto cleanup; }
    ret = Route(buf);

cleanup:
    if (buf != NULL) { BufPool_Free(buf); }
    return ret;
}
```

---

### Common Deviations

**Deviation: Rule 15.5 — Multiple Returns in Non-Safety-Critical Code**

```
Deviation ID:    DEV-FLOW-001
Rule:            MISRA C:2012 Rule 15.5 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           ASIL QM (non-safety) modules and utility libraries
Justification:   In deeply nested validation functions, single-exit creates
                 excessive nesting (>4 levels) that harms readability and
                 increases cyclomatic complexity. Early return on null/error
                 is more maintainable for QM code. Safety-critical code
                 (ASIL B–D) must still follow single-exit.
Conditions:      - Only for ASIL QM rated modules
                 - Maximum of 3 early returns per function, all for error paths
                 - All early returns must be within the first 10 lines (guard clause style)
                 - Function complexity shall not exceed 10 (McCabe)
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 15.1 — Goto for Error Cleanup in C**

```
Deviation ID:    DEV-FLOW-002
Rule:            MISRA C:2012 Rule 15.1 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           C modules only (C++ must use RAII)
Justification:   Forward-only goto to a single cleanup label prevents resource
                 leaks in functions acquiring multiple resources.
Conditions:      - Single cleanup label per function, at end of function body
                 - Forward jumps only
                 - Not permitted in C++ (use RAII/scope guards instead)
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Rules 15.1–15.7, 16.1–16.7; MISRA C++:2023 Rules 9.4.x, 9.5.x
