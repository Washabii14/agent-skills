---
title: GreenHills MULTI Safety-Qualified Compiler
impact: HIGH
impactDescription: GreenHills is the only IEC 61508 SIL 4 / ISO 26262 ASIL D pre-qualified compiler — correct pragma and option usage is essential for safety-certified builds
tags: build, greenhills, multi, asil-d, safety, iso-26262, compiler, pragma, qualified
---

## GreenHills MULTI Safety-Qualified Compiler

GreenHills MULTI is a safety-pre-qualified compiler for ISO 26262 ASIL D applications. Its proprietary pragmas control memory section placement, endianness, optimization levels, and runtime error checking. Misconfiguration negates the safety qualification.

**Incorrect (using GreenHills without safety options):**

```c
/* Building without runtime checks or section control */
/* ghscc -c app.c -o app.o */

uint8_t sensor_data[256];

int32_t Process_Sensor(int32_t raw_value)
{
    /* No runtime overflow checking enabled */
    int32_t scaled = raw_value * SCALE_FACTOR;
    return scaled / OFFSET_DIVISOR;  /* Potential divide-by-zero undetected */
}
```

**Correct (safety-qualified options and pragmas):**

```c
/* Build command with safety qualification options:
 * ghscc -c app.c -o app.o
 *   --safety_qualification
 *   -check=all
 *   -Ogeneral            (deterministic optimization level)
 *   -misra_c2012         (enable MISRA checking)
 *   --no_commons
 *   -dual_debug
 */

/* Place safety-critical data in dedicated memory section */
#pragma ghs section bss=".safety_bss"
#pragma ghs section data=".safety_data"

static uint8_t sensor_data[256];
static int32_t calibration_offset = DEFAULT_OFFSET;

#pragma ghs section bss=default
#pragma ghs section data=default

/* Endianness control for cross-platform CAN data */
#pragma ghs endian big
typedef struct
{
    uint16_t message_id;
    uint8_t  dlc;
    uint8_t  payload[8];
} CanFrame_BigEndian_t;
#pragma ghs endian default

/* Optimization control for timing-critical code */
#pragma ghs optimize=none
void SafetyMonitor_Check(void)
{
    /* Optimization disabled — deterministic execution time required.
     * GreenHills -check=all enables:
     *   - Arithmetic overflow detection
     *   - Divide-by-zero trapping
     *   - Array bounds checking
     *   - Null pointer dereference detection
     */
    if (calibration_offset == 0)
    {
        SafeState_Enter(FAULT_DIV_ZERO);
        return;
    }
    int32_t result = g_raw_value * SCALE_FACTOR;
    g_processed = result / calibration_offset;
}
#pragma ghs optimize=default

/* ASIL D certification pragma — marks function for safety-qualified compilation */
#pragma ghs ZAP_object_qualification ASIL_D
void ASILD_CriticalFunction(void)
{
    /* Compiler tracks this function's object code for
     * ISO 26262 certification evidence */
}
#pragma ghs ZAP_object_qualification default
```

**GreenHills project file (.gpj) safety configuration:**

```
# project.gpj — safety-qualified build configuration
[Project]
    -safety_qualification
    -check=all
    -check=assignbound
    -check=nilderef
    -check=switch
    -Ogeneral
    -misra_c2012
    -no_commons
    -dual_debug
    --signed_chars
    -cpu=cortexr5f
    -fpu=vfpv3_d16
    --diag_error=implicit-function-declaration
    --diag_error=uninitialized-local-variable
    --no_exceptions
```

**Key compiler options for ISO 26262:**

| Option | Purpose |
|--------|---------|
| `--safety_qualification` | Activates IEC 61508 SIL 4 qualified mode |
| `-check=all` | Runtime checks: overflow, bounds, null, div-zero |
| `-Ogeneral` | Deterministic optimization (auditable transforms) |
| `-misra_c2012` | Built-in MISRA C:2012 rule checking |
| `-dual_debug` | Produces debug info for both source and assembly |
| `--no_commons` | Prevents uninitialized globals sharing (C standard pitfall) |
| `#pragma ghs section` | Memory section placement for MPU/safety partitioning |

Reference: GreenHills MULTI Safety Manual, ISO 26262-6 Table 3 (software unit design verification), IEC 61508-3 (pre-qualification of tools)
