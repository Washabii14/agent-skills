---
title: MISRA Pointer Safety — Arithmetic, Null Checks, Conversions, Restrict
impact: HIGH
impactDescription: prevents out-of-bounds access, null dereference, and undefined behavior from pointer misuse
tags: misra, pointer, arithmetic, null-check, restrict, array, bounds, safety
---

## MISRA Pointer Safety — Arithmetic, Null Checks, Conversions, Restrict

Pointer misuse is the leading cause of memory corruption in embedded C. These rules restrict pointer arithmetic to array contexts, enforce null checking, and prevent unsafe pointer conversions.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 18.1 | Required | Pointer arithmetic shall only produce a result that addresses an element of the same array |
| 18.2 | Required | Subtraction between pointers shall address elements of the same array |
| 18.3 | Required | Relational operators shall not be applied to unrelated pointers |
| 18.4 | Advisory | +, -, += and -= shall not be applied to pointers (prefer array indexing) |
| 18.5 | Advisory | No more than 2 levels of pointer indirection |
| 18.6 | Required | Address of automatic variable shall not be assigned to outliving object |
| 18.7 | Required | Flexible array members shall not be declared |
| 18.8 | Required | Variable-length arrays shall not be used |
| 11.1 | Required | No conversion between function pointer and other type |
| 11.2 | Required | No conversion between pointer to incomplete type and other type |
| 11.3 | Required | No cast between object pointer types (see also misra-type-system.md) |
| 11.4 | Advisory | No conversion between pointer to object and integer type |
| 11.5 | Advisory | No implicit conversion from void* to object pointer |

MISRA C++:2023 overlap: Rules 8.2.4 (reinterpret_cast), 8.7.1–8.7.2 (pointer arithmetic), 21.2.1 (null pointer dereference), 8.0.1 (pointer conversions).

---

### Top Violations and Patterns

**Rule 18.1 — Out-of-Bounds Pointer Arithmetic**

Incorrect (pointer walks past array end):

```c
void ClearBuffer(uint8_t *buf, uint16_t size) {
    uint8_t *end = buf + size;
    uint8_t *p = buf;
    while (p <= end) {   /* off-by-one: dereferences one past end */
        *p = 0U;
        p++;
    }
}
```

Correct (strict bounds):

```c
void ClearBuffer(uint8_t *buf, uint16_t size) {
    for (uint16_t i = 0U; i < size; i++) {
        buf[i] = 0U;
    }
}
```

C++ equivalent:

```cpp
void ClearBuffer(std::span<uint8_t> buf) {
    std::fill(buf.begin(), buf.end(), 0U);
}
```

---

**Rule 18.4 — Prefer Array Indexing Over Pointer Arithmetic**

Incorrect (pointer arithmetic):

```c
uint16_t ReadSignal(const uint8_t *frame) {
    return (uint16_t)(*(frame + 4) << 8U) | (uint16_t)(*(frame + 5));
}
```

Correct (array indexing):

```c
uint16_t ReadSignal(const uint8_t *frame) {
    return ((uint16_t)frame[4] << 8U) | (uint16_t)frame[5];
}
```

---

**Rule 18.6 — Dangling Pointer to Local Variable**

Incorrect (returning pointer to local):

```c
const char *GetModeName(uint8_t mode) {
    char name[32];
    (void)snprintf(name, sizeof(name), "MODE_%u", mode);
    return name;  /* dangling pointer — name is on stack */
}
```

Correct (caller-provided buffer):

```c
void GetModeName(uint8_t mode, char *buf, size_t buf_size) {
    (void)snprintf(buf, buf_size, "MODE_%u", mode);
}
```

Correct (static lookup for known set):

```c
const char *GetModeName(uint8_t mode) {
    static const char *const names[] = {
        "MODE_OFF", "MODE_INIT", "MODE_RUN", "MODE_SHUTDOWN"
    };
    if (mode < (uint8_t)(sizeof(names) / sizeof(names[0]))) {
        return names[mode];
    }
    return "MODE_UNKNOWN";
}
```

---

**Rule 18.5 — Limit Pointer Indirection Depth**

Incorrect (triple pointer):

```c
void GetConfig(Config_t ***cfg_list) {
    /* three levels of indirection — unreadable and error-prone */
}
```

Correct (flatten with struct or limit to 2 levels):

```c
typedef struct {
    Config_t *entries;
    uint16_t count;
} ConfigList_t;

void GetConfig(ConfigList_t *cfg_list) {
    /* single level of indirection into clear struct */
}
```

---

**Rule 18.8 — No Variable-Length Arrays**

Incorrect (VLA on stack):

```c
void ProcessData(uint16_t n) {
    uint8_t buffer[n];  /* stack size unknown at compile time — may overflow */
}
```

Correct (fixed-size buffer with bounds check):

```c
#define PROCESS_BUF_MAX  256U

void ProcessData(uint16_t n) {
    uint8_t buffer[PROCESS_BUF_MAX];
    if (n > PROCESS_BUF_MAX) {
        Dem_ReportError(DEM_BUF_OVERFLOW);
        return;
    }
    /* use buffer[0..n-1] */
}
```

---

**Null Pointer Checking Pattern**

Incorrect (no null check before dereference):

```c
void UpdateSensor(SensorData_t *data) {
    data->timestamp = GetTick();  /* crash if data == NULL */
}
```

Correct (null guard with defensive return):

```c
void UpdateSensor(SensorData_t *data) {
    if (data == NULL) {
        Dem_ReportError(DEM_NULL_PTR);
        return;
    }
    data->timestamp = GetTick();
}
```

C++ equivalent:

```cpp
void UpdateSensor(SensorData_t& data) {  // reference cannot be null
    data.timestamp = GetTick();
}
// If pointer is unavoidable:
void UpdateSensor(SensorData_t* data) {
    if (data == nullptr) {
        Dem_ReportError(DemEventId::kNullPtr);
        return;
    }
    data->timestamp = GetTick();
}
```

---

### Common Deviations

**Deviation: Rule 18.4 — Pointer Arithmetic in DMA Scatter-Gather Lists**

```
Deviation ID:    DEV-PTR-001
Rule:            MISRA C:2012 Rule 18.4 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           DMA driver and ring-buffer implementations only
Justification:   DMA scatter-gather descriptor lists require pointer increment
                 to walk contiguous descriptor arrays. Array indexing alternative
                 would require maintaining a separate index and offers no safety
                 benefit since the array bounds are hardware-fixed.
Conditions:      - Pointer arithmetic only on arrays with compile-time-known size
                 - Static assertion that descriptor count matches array size
                 - Bounded loop with explicit upper limit
Approved by:     Safety Manager, [date]
```

**Deviation: Array-to-Pointer Decay in API Boundaries**

```
Deviation ID:    DEV-PTR-002
Rule:            MISRA C:2012 Rule 18.1 (Related)
Category:        Required — Mitigated
Scope:           AUTOSAR COM/PDU interfaces
Justification:   AUTOSAR SWS APIs accept (uint8*, length) pairs. Array-to-pointer
                 decay is unavoidable at these boundaries. Safety is maintained by
                 always passing the length alongside the pointer and validating
                 length < buffer capacity at the receiving end.
Conditions:      - Every pointer parameter must have an associated length parameter
                 - Receiving function must validate length before access
                 - Static analysis tool configured to accept this pattern at API boundaries
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Rules 18.1–18.8, 11.1–11.5; MISRA C++:2023 Rules 8.7.x, 21.2.1
