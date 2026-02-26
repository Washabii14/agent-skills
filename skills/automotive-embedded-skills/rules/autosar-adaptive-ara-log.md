---
title: "ara::log — Logging Levels, Log Context, and Structured Logging Patterns"
impact: LOW-MEDIUM
impactDescription: Improper logging causes excessive overhead in production, lost diagnostic trace data, or non-compliant DLT output
tags: autosar, adaptive, ara-log, logging, log-level, log-context, logstream, dlt, structured-logging
---

## ara::log — Logging Levels, Log Context, and Structured Logging

ara::log (AUTOSAR Adaptive R24-10) provides the standard logging API for Adaptive Applications, compatible with GENIVI DLT (Diagnostic Log and Trace). It supports log levels, application/context IDs, and structured log streams. Correct usage ensures production builds have minimal overhead while retaining diagnostic capability.

### Logger Creation and Log Context

**Incorrect (using stdout for logging):**

```cpp
#include <iostream>

void ProcessSensor()
{
    std::cout << "Sensor value: " << value << std::endl;  // No level, no context
    // Production: floods output, no filtering, no DLT integration
}
```

**Correct (ara::log with proper context):**

```cpp
#include <ara/log/logging.h>

// Create logger per component — 4-char context ID for DLT
static ara::log::Logger& GetLogger()
{
    static ara::log::Logger& logger = ara::log::CreateLogger(
        "SENS",                          // Context ID (max 4 chars)
        "Sensor Processing Module",      // Context description
        ara::log::LogLevel::kInfo);      // Default threshold
    return logger;
}

void ProcessSensor()
{
    auto& log = GetLogger();

    log.LogInfo() << "Sensor initialized successfully";

    log.LogDebug() << "Raw sensor value: " << rawValue
                   << " filtered: " << filteredValue;

    if (sensorError)
    {
        log.LogError() << "Sensor read failure, error code: "
                       << static_cast<int>(errorCode);
    }
}
```

### Logging Levels

```cpp
// Log levels (ara::log::LogLevel), ordered by severity:
//
// kOff     — Logging disabled
// kFatal   — System-level failure, immediate attention required
// kError   — Functional error, operation failed
// kWarn    — Abnormal condition, operation continues
// kInfo    — Significant operational events (startup, shutdown, config)
// kDebug   — Developer-level detail for debugging
// kVerbose — Maximum detail, performance impact acceptable only in dev

// Set threshold at initialization (Machine Manifest or runtime)
ara::log::Logger& logger = ara::log::CreateLogger(
    "RDAR", "Radar Processing", ara::log::LogLevel::kWarn);

// Only kWarn, kError, kFatal messages pass — kInfo/kDebug/kVerbose suppressed
logger.LogDebug() << "This is suppressed at zero cost";  // No string formatting
logger.LogWarn()  << "Radar signal degraded: SNR=" << snr;  // This is emitted
```

### LogStream Usage and Structured Logging

**Incorrect (expensive computation inside log call regardless of level):**

```cpp
void DiagnosticDump()
{
    // FormatDiagData() runs even if log level suppresses output
    log.LogDebug() << FormatDiagData(allSensors, allActuators);
}
```

**Correct (guard expensive logging with level check):**

```cpp
void DiagnosticDump()
{
    if (log.IsEnabled(ara::log::LogLevel::kDebug))
    {
        log.LogDebug() << FormatDiagData(allSensors, allActuators);
    }
}

// LogStream supports multiple types natively:
void LogExample()
{
    auto& log = GetLogger();

    uint32_t cycleCount = 42;
    float temperature = 85.3f;
    ara::core::StringView component("RadarFrontLeft");

    log.LogInfo() << "Cycle: " << cycleCount
                  << " temp: " << temperature
                  << " component: " << component;

    // Binary data with hex formatting:
    std::array<uint8_t, 8> rawData = {0x01, 0x02, 0x03};
    log.LogDebug() << ara::log::LogHex8(rawData.data(), rawData.size());
}
```

### Compile-Time Log Level Filtering

```cpp
// For maximum performance in production builds, compile-time filtering
// eliminates log code entirely:

// CMakeLists.txt or build config:
// target_compile_definitions(MyApp PRIVATE ARA_LOG_COMPILE_LEVEL=3)
//
// Level mapping:
//   0 = kOff (all logging compiled out)
//   1 = kFatal only
//   2 = kFatal + kError
//   3 = kFatal + kError + kWarn
//   4 = + kInfo
//   5 = + kDebug
//   6 = + kVerbose (all)

// With ARA_LOG_COMPILE_LEVEL=3, this compiles to nothing:
log.LogDebug() << "This entire statement is optimized away";

// This remains:
log.LogWarn() << "Low voltage detected: " << voltage << "V";
```

### Application ID Registration

```cpp
// Register application with DLT at process start (before creating loggers)
int main()
{
    ara::log::InitLogging(
        "MYAP",                     // Application ID (4 chars)
        "My Adaptive Application",  // Application description
        ara::log::LogLevel::kInfo,  // Default log level
        ara::log::LogMode::kConsole | ara::log::LogMode::kRemote);

    // kConsole: output to stdout (development)
    // kRemote:  output to DLT daemon (production)
    // kFile:    output to file (testing)

    // Now create loggers...
}
```

Reference: AUTOSAR Adaptive R24-10 SWS_Logging (ara::log), GENIVI DLT Protocol
