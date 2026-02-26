---
title: MISRA Initialization — Variables, Arrays, Structs, Designated Initializers
impact: HIGH
impactDescription: prevents use of uninitialized data which causes unpredictable behavior and latent field defects
tags: misra, initialization, uninitialized, array-init, struct-init, designated-initializer, safety
---

## MISRA Initialization — Variables, Arrays, Structs, Designated Initializers

Use of uninitialized variables is undefined behavior in C and a leading cause of intermittent field failures. These rules ensure every object is initialized before use and that aggregate initializers are complete and unambiguous.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 9.1 | Mandatory | Value of object with automatic storage shall not be read before being set |
| 9.2 | Required | Initializer for aggregate or union shall be enclosed in braces |
| 9.3 | Required | Arrays shall not be partially initialized |
| 9.4 | Required | Element of object shall not be initialized more than once |
| 9.5 | Required | Where designated initializers are used, array size shall be specified explicitly |

MISRA C++:2023 overlap: Rules 11.6.1 (class member initialization), 11.6.2 (brace initialization), 6.0.2 (uninitialized read).

---

### Top Violations and Patterns

**Rule 9.1 — Uninitialized Variable Read (most critical — Mandatory)**

Incorrect (conditional initialization path):

```c
Std_ReturnType ReadSensor(uint8_t channel, uint16_t *value) {
    uint16_t raw;  /* not initialized */

    if (channel < NUM_CHANNELS) {
        raw = AdcHw_Read(channel);
    }
    /* if channel >= NUM_CHANNELS, raw is uninitialized */

    *value = raw;  /* potential read of uninitialized variable */
    return E_OK;
}
```

Correct (initialize at declaration):

```c
Std_ReturnType ReadSensor(uint8_t channel, uint16_t *value) {
    Std_ReturnType ret = E_NOT_OK;
    uint16_t raw = 0U;

    if (channel < NUM_CHANNELS) {
        raw = AdcHw_Read(channel);
        *value = raw;
        ret = E_OK;
    } else {
        *value = 0U;
        Dem_ReportError(DEM_INVALID_CHANNEL);
    }

    return ret;
}
```

C++ equivalent:

```cpp
std::optional<uint16_t> ReadSensor(uint8_t channel) {
    if (channel >= kNumChannels) {
        Dem_ReportError(DemEventId::kInvalidChannel);
        return std::nullopt;
    }
    return AdcHw_Read(channel);
}
```

---

**Rule 9.2 — Brace-Enclosed Aggregate Initializers**

Incorrect (flat initializer for nested struct):

```c
typedef struct {
    uint16_t x;
    uint16_t y;
} Point_t;

typedef struct {
    Point_t origin;
    Point_t size;
} Rect_t;

Rect_t r = { 0, 0, 100, 200 };  /* ambiguous: which values go where? */
```

Correct (nested braces matching structure):

```c
Rect_t r = {
    { .x = 0U, .y = 0U },      /* origin */
    { .x = 100U, .y = 200U }   /* size */
};
```

C++ equivalent:

```cpp
Rect r = {
    .origin = {.x = 0U, .y = 0U},
    .size = {.x = 100U, .y = 200U}
};
```

---

**Rule 9.3 — No Partial Array Initialization**

Incorrect (partial init leaves remainder zero-initialized implicitly):

```c
uint8_t lookup[256] = { 0, 1, 2, 3 };  /* elements [4]-[255] are implicitly 0 */
```

If partial initialization is intentional, make it explicit:

```c
/* Correct approach 1: initialize all elements */
static const uint8_t lookup[256] = {
    [0] = 0U, [1] = 1U, [2] = 2U, [3] = 3U,
    /* remaining elements intentionally zero — use memset at runtime or
       designated initializers for all non-zero elements */
};

/* Correct approach 2: zero-init then set specific values */
uint8_t lookup[256];
(void)memset(lookup, 0, sizeof(lookup));
lookup[0] = 0U;
lookup[1] = 1U;
lookup[2] = 2U;
lookup[3] = 3U;
```

---

**Rule 9.4 — No Duplicate Initialization**

Incorrect (same element set twice):

```c
uint16_t table[4] = {
    [0] = 100U,
    [1] = 200U,
    [0] = 150U,   /* element [0] initialized twice — which value wins? */
    [3] = 400U
};
```

Correct (each element initialized exactly once):

```c
uint16_t table[4] = {
    [0] = 150U,
    [1] = 200U,
    [2] = 0U,
    [3] = 400U
};
```

---

**Rule 9.5 — Explicit Array Size with Designated Initializers**

Incorrect (compiler-inferred size from initializer):

```c
uint8_t error_codes[] = {
    [ERR_TIMEOUT] = 1U,
    [ERR_CRC] = 2U,
    [ERR_OVERFLOW] = 3U
};
/* array size depends on enum values — fragile */
```

Correct (explicit size):

```c
#define NUM_ERROR_CODES  16U

uint8_t error_codes[NUM_ERROR_CODES] = {
    [ERR_TIMEOUT]  = 1U,
    [ERR_CRC]      = 2U,
    [ERR_OVERFLOW]  = 3U
};

_Static_assert(ERR_OVERFLOW < NUM_ERROR_CODES, "Error code exceeds array size");
```

---

### Initialization Best Practices for Embedded

**Zero-initialization pattern for static data:**

```c
/* Module-level: zero-initialized by C startup (crt0) */
static SensorState_t sensor_state;

/* Function-level: explicit zero-init */
void InitModule(void) {
    SensorConfig_t cfg = { 0 };       /* C: zero-init shorthand */
    (void)memset(&cfg, 0, sizeof(cfg)); /* alternative explicit zero */
}
```

C++ equivalent:

```cpp
SensorConfig cfg{};    // value-initialization — all members zeroed
```

**Designated initializers for configuration structs:**

```c
static const AdcConfig_t adc_cfg = {
    .channel     = ADC_CH_TEMP,
    .resolution  = ADC_RES_12BIT,
    .sample_time = ADC_SAMP_56_CYCLES,
    .alignment   = ADC_ALIGN_RIGHT,
    .callback    = NULL
};
```

---

### Common Deviations

**Deviation: Rule 9.3 — Partial Initialization of Large Lookup Tables**

```
Deviation ID:    DEV-INIT-001
Rule:            MISRA C:2012 Rule 9.3 (Required)
Category:        Required → Permitted with conditions
Scope:           Constant lookup tables with sparse non-zero entries
Justification:   Large lookup tables (e.g., 256-entry CRC tables, character
                 class tables) where only a subset of entries are non-zero.
                 Explicitly listing all 256 zero values harms readability
                 and maintainability. The C standard guarantees that
                 unspecified elements of partially initialized arrays
                 are zero-initialized.
Conditions:      - Array must be declared const (immutable)
                 - A comment must document that unspecified elements are
                   intentionally zero
                 - Static analysis tool suppressions must reference this deviation ID
                 - Unit test must verify the full table contents
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 9.1 — Compiler-Initialized Stack Variables**

```
Deviation ID:    DEV-INIT-002
Rule:            MISRA C:2012 Rule 9.1 (Mandatory)
Category:        Mandatory — NOT deviated
Note:            This is a Mandatory rule and cannot be deviated. All automatic
                 (stack) variables must be initialized before read. Enable
                 compiler flag -Wuninitialized and treat as error. Where
                 static analysis reports false positives, use tool-specific
                 suppression with justification comment.
```

Reference: MISRA C:2012 Rules 9.1–9.5; MISRA C++:2023 Rules 11.6.x, 6.0.2
