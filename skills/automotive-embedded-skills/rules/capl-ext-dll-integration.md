---
title: CAPL DLL Integration
impact: HIGH
impactDescription: Incorrect DLL integration causes measurement crashes, data corruption, and hard-to-diagnose runtime failures
tags: capl, dll, integration, c, cpp, api, thread-safety, external
---

## CAPL DLL Integration

CAPL DLLs extend CANoe/vTESTstudio with native C/C++ code for hardware access, complex algorithms, or protocol implementations that exceed CAPL's capabilities. Correct integration requires matching the CAPL DLL API contract, handling data exchange safely, respecting threading constraints, and managing bitness alignment.

---

### Standard CAPL DLL API

**Incorrect (missing DLL API contract, wrong calling convention):**

```c
/* Missing CAPL DLL exports — CANoe cannot bind these functions */
int myFunction(int a, int b)
{
    return a + b;
}
```

**Correct (proper CAPL DLL export with initialization and cleanup):**

The DLL must include `cdll.h` (shipped with CANoe) and export functions using the `CAPL_DLL_EXPORT` table.

Header — `my_capl_dll.h`:

```c
#ifndef MY_CAPL_DLL_H
#define MY_CAPL_DLL_H

#include "cdll.h"

#ifdef __cplusplus
extern "C" {
#endif

void CAPLEXPORT far CAPLPASCAL caplDllInit(void);
void CAPLEXPORT far CAPLPASCAL caplDllRelease(void);

long CAPLEXPORT far CAPLPASCAL dllAdd(long a, long b);
double CAPLEXPORT far CAPLPASCAL dllScale(double value, double factor);
void CAPLEXPORT far CAPLPASCAL dllFormatString(char* input, char* output);

#ifdef __cplusplus
}
#endif

#endif
```

Implementation — `my_capl_dll.c`:

```c
#include "my_capl_dll.h"
#include <string.h>
#include <stdio.h>

void CAPLEXPORT far CAPLPASCAL caplDllInit(void)
{
    /* Called once when CANoe loads the DLL (measurement start).
     * Initialize resources, open file handles, allocate memory here. */
}

void CAPLEXPORT far CAPLPASCAL caplDllRelease(void)
{
    /* Called when CANoe unloads the DLL (measurement stop).
     * Release all resources acquired in caplDllInit. */
}

long CAPLEXPORT far CAPLPASCAL dllAdd(long a, long b)
{
    return a + b;
}

double CAPLEXPORT far CAPLPASCAL dllScale(double value, double factor)
{
    return value * factor;
}

void CAPLEXPORT far CAPLPASCAL dllFormatString(char* input, char* output)
{
    /* CAPL passes char* for string parameters.
     * output buffer is pre-allocated by CANoe (typically 512 bytes). */
    snprintf(output, 512, "Processed: %s", input);
}
```

Export table — required for CANoe to discover functions:

```c
#include "cdll.h"

CAPL_DLL_EXPORT_TABLE_BEGIN
    CAPL_DLL_EXPORT(dllAdd,       "long", "long,long",
                    "Add two integers",     'L', 2, "LL", "a", "b")
    CAPL_DLL_EXPORT(dllScale,     "double", "double,double",
                    "Scale a value",        'D', 2, "DD", "value", "factor")
    CAPL_DLL_EXPORT(dllFormatString, "void", "char*,char*",
                    "Format a string",      'V', 2, "CC", "input", "output")
CAPL_DLL_EXPORT_TABLE_END
```

The type codes in the export table: `'L'` = long, `'D'` = double, `'V'` = void, `'F'` = float. Parameter type string uses `'L'` = long, `'D'` = double, `'C'` = char*.

CAPL side — using the DLL:

```capl
variables
{
    char resultBuf[512];
}

on start
{
    long sum;
    double scaled;

    sum = dllAdd(10, 20);
    write("Sum: %d", sum);

    scaled = dllScale(3.14, 2.0);
    write("Scaled: %.4f", scaled);

    dllFormatString("hello", resultBuf);
    write("Formatted: %s", resultBuf);
}
```

---

### Data Exchange: Arrays and Structs

**Incorrect (passing complex data without matching layout):**

```c
/* Struct with no packing — padding will differ between compilers */
struct SensorData {
    char id;
    double value;
    int timestamp;
};
```

**Correct (packed struct with explicit layout, array passing via pointer+length):**

DLL side:

```c
#pragma pack(push, 1)
typedef struct {
    long  id;
    double value;
    long  timestamp;
} SensorData;
#pragma pack(pop)

/* Array exchange: CAPL passes arrays as pointer + length */
double CAPLEXPORT far CAPLPASCAL dllArrayAverage(double values[], long count)
{
    double sum = 0.0;
    long i;
    if (count <= 0) return 0.0;

    for (i = 0; i < count; i++)
        sum += values[i];

    return sum / count;
}

/* Struct exchange: pass as byte array, reinterpret in DLL */
long CAPLEXPORT far CAPLPASCAL dllProcessSensor(unsigned char data[], long dataLen)
{
    SensorData* sensor;

    if (dataLen < (long)sizeof(SensorData))
        return -1;

    sensor = (SensorData*)data;

    if (sensor->value > 100.0)
        return 1;

    return 0;
}
```

CAPL side:

```capl
variables
{
    double sensorReadings[10];
    byte   sensorPacket[16];
}

on start
{
    double avg;
    long   result;
    int    i;

    for (i = 0; i < 10; i++)
        sensorReadings[i] = i * 1.1;

    avg = dllArrayAverage(sensorReadings, 10);
    write("Average: %.2f", avg);

    /* Pack struct into byte array matching DLL's packed layout:
     *   bytes 0-3:  id (long)
     *   bytes 4-11: value (double)
     *   bytes 12-15: timestamp (long) */
    sensorPacket[0] = 1;  /* id = 1 (little-endian long) */
    /* In practice, use memcpy or memcpy_off for multi-byte fields */

    result = dllProcessSensor(sensorPacket, 16);
    write("Sensor result: %d", result);
}
```

---

### Callbacks from DLL to CAPL

**Incorrect (calling CAPL functions directly from DLL — not supported):**

```c
/* DLL cannot directly call CAPL functions by name at runtime */
void dllDoWork(void)
{
    caplWriteMessage("Done");  /* does not exist in DLL context */
}
```

**Correct (use the CYCAPL callback mechanism via function table):**

The CAPL DLL API provides a callback mechanism through `caplCallbackFunction`. Register callbacks in the export table, then call them from C when needed.

DLL side:

```c
#include "cdll.h"

static CYCAPL_CALLBACK_FUNC gNotifyCallback = NULL;

void CAPLEXPORT far CAPLPASCAL dllRegisterCallback(
    CYCAPL_CALLBACK_FUNC callback)
{
    gNotifyCallback = callback;
}

void CAPLEXPORT far CAPLPASCAL dllProcessAsync(long input)
{
    long result = input * 2;

    /* Invoke the CAPL callback if registered */
    if (gNotifyCallback != NULL)
    {
        /* Parameters must match the CAPL function signature */
        caplCallbackFunction(gNotifyCallback, &result);
    }
}
```

CAPL side:

```capl
void OnDllNotify(long result)
{
    write("DLL callback result: %d", result);
}

on start
{
    dllRegisterCallback("OnDllNotify");
    dllProcessAsync(42);
}
```

Note: the exact callback API depends on the CANoe version. Consult the `cdll.h` header shipped with your installation for the precise signatures. Older versions may use a different registration pattern.

---

### Thread Safety

**Incorrect (blocking call inside DLL function):**

```c
/* CAPL DLL functions run on CANoe's measurement thread.
 * Blocking here freezes the entire measurement. */
long CAPLEXPORT far CAPLPASCAL dllReadSensor(void)
{
    Sleep(5000);          /* blocks measurement for 5 seconds */
    return readHardware(); /* may also block on I/O */
}
```

**Correct (non-blocking DLL with mutex-protected shared state):**

```c
#include "cdll.h"
#include <windows.h>

static CRITICAL_SECTION gDataLock;
static double gLatestValue = 0.0;
static volatile int gWorkerRunning = 0;
static HANDLE gWorkerThread = NULL;

/* Background thread performs slow I/O independently */
static DWORD WINAPI workerThread(LPVOID param)
{
    while (gWorkerRunning)
    {
        double val = readHardwareSensor(); /* slow I/O here is safe */

        EnterCriticalSection(&gDataLock);
        gLatestValue = val;
        LeaveCriticalSection(&gDataLock);

        Sleep(100);
    }
    return 0;
}

void CAPLEXPORT far CAPLPASCAL caplDllInit(void)
{
    InitializeCriticalSection(&gDataLock);
    gWorkerRunning = 1;
    gWorkerThread = CreateThread(NULL, 0, workerThread, NULL, 0, NULL);
}

void CAPLEXPORT far CAPLPASCAL caplDllRelease(void)
{
    gWorkerRunning = 0;
    if (gWorkerThread)
    {
        WaitForSingleObject(gWorkerThread, 2000);
        CloseHandle(gWorkerThread);
        gWorkerThread = NULL;
    }
    DeleteCriticalSection(&gDataLock);
}

/* Called from CAPL — returns immediately with latest value */
double CAPLEXPORT far CAPLPASCAL dllGetSensorValue(void)
{
    double val;
    EnterCriticalSection(&gDataLock);
    val = gLatestValue;
    LeaveCriticalSection(&gDataLock);
    return val;
}
```

Key threading rules:
- CAPL DLL functions execute on the **measurement thread** — never block.
- No Win32 UI calls (`MessageBox`, `CreateWindow`) from DLL functions.
- Use `CRITICAL_SECTION` for shared state between worker threads and CAPL-called functions.
- Start background threads in `caplDllInit`, stop them in `caplDllRelease`.

---

### 32-bit vs 64-bit Considerations

**Incorrect (bitness mismatch):**

```capl
/* CANoe is running in 64-bit mode but the DLL was compiled as 32-bit.
 * Result: DLL fails to load silently or crashes. */
on start
{
    dllInit();  /* call never executes — DLL not loaded */
}
```

**Correct (matched bitness with diagnostic check):**

Build the DLL to match CANoe's process architecture:
- **CANoe x64** (default on modern installations) → compile DLL as **x64**
- **CANoe x86** (legacy or explicitly selected) → compile DLL as **x86**

Check CANoe's bitness: Help → About → "64-bit" or "32-bit" in the version string.

Struct alignment must be explicit to ensure identical layout in CAPL and C:

```c
/* Always use pragma pack for CAPL-exchanged structs */
#pragma pack(push, 1)
typedef struct {
    long   msgId;
    long   dlc;
    unsigned char data[8];
    double timestamp;
} CanFrame;
#pragma pack(pop)

/* sizeof(CanFrame) = 4 + 4 + 8 + 8 = 24 bytes on both x86 and x64 */
```

Use fixed-width types (`long` = 4 bytes in CAPL) rather than platform-dependent types (`int`, `size_t`). The CAPL `long` is always 32-bit signed; the DLL must use `long` (not `int64_t`) for matching parameters.

---

### Error Handling: DLL Load Failures

**Incorrect (no detection of DLL failure):**

```capl
on start
{
    long val;
    val = dllCompute(100);  /* if DLL failed to load, this is undefined */
    write("Result: %d", val);
}
```

**Correct (detect DLL availability and degrade gracefully):**

```capl
variables
{
    int gDllAvailable = 0;
}

on preStart
{
    /* Probe DLL by calling a known safe function.
     * If the DLL did not load, CAPL will report a runtime error.
     * Wrap in a controlled check. */
    gDllAvailable = dllProbe();
    if (gDllAvailable != 1)
    {
        write("WARNING: CAPL DLL not loaded — running in degraded mode");
        gDllAvailable = 0;
    }
    else
    {
        write("CAPL DLL loaded successfully");
    }
}

on start
{
    if (gDllAvailable)
    {
        long val;
        val = dllCompute(100);
        write("DLL result: %d", val);
    }
    else
    {
        write("Skipping DLL computation — DLL unavailable");
    }
}
```

DLL side — provide a probe function:

```c
long CAPLEXPORT far CAPLPASCAL dllProbe(void)
{
    /* Return 1 to indicate DLL is loaded and operational */
    return 1;
}
```

For DLL-internal errors, return error codes rather than crashing:

```c
long CAPLEXPORT far CAPLPASCAL dllCompute(long input)
{
    if (input < 0)
        return -1;  /* error code — CAPL checks return value */

    return input * 42;
}
```

```capl
on timer computeTimer
{
    long result;

    if (!gDllAvailable) return;

    result = dllCompute(sensorValue);
    if (result < 0)
    {
        write("DLL error: dllCompute returned %d for input %d",
              result, sensorValue);
    }
    else
    {
        setSignal(ProcessedValue, result);
    }
}
```

---

### Summary of CAPL DLL API Type Mappings

| CAPL Type | DLL C Type | Export Code |
|-----------|-----------|-------------|
| `long`    | `long`    | `'L'`       |
| `dword`   | `unsigned long` | `'D'` (context-dependent) |
| `float`   | `float`   | `'F'`       |
| `double`  | `double`  | `'D'`       |
| `char[]`  | `char*`   | `'C'`       |
| `byte[]`  | `unsigned char*` | `'B'` |

Always consult the `cdll.h` header from your CANoe installation for the definitive macro signatures and type codes, as they may vary between Vector tool versions.

Reference: Vector CANoe Help — CAPL DLL Programming, cdll.h API Reference
