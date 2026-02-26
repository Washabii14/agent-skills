---
title: "ara::core — Result<T,E>, ErrorCode, Future/Promise, Span, and StringView"
impact: HIGH
impactDescription: Not using ara::core Result causes exception-based error handling, which is prohibited in safety-critical automotive software
tags: autosar, adaptive, ara-core, result, error-code, error-domain, future, promise, span, string-view, no-exceptions
---

## ara::core — Result, ErrorCode, Future/Promise, and Vocabulary Types

ara::core (AUTOSAR Adaptive R24-10) provides foundational vocabulary types for the Adaptive Platform. The most critical is `Result<T, E>` which enables exception-free error handling — essential because C++ exceptions are prohibited in ASIL-rated automotive software. All ara:: APIs return `Result` instead of throwing.

### Result<T, E> — Exception-Free Error Handling

**Incorrect (using C++ exceptions in automotive adaptive code):**

```cpp
RadarData ReadRadar()
{
    if (!radarHw.IsReady())
    {
        throw std::runtime_error("Radar not ready");  // Prohibited in ASIL code
    }
    return radarHw.GetData();
}

void ProcessRadar()
{
    try  // Exception handling has non-deterministic timing
    {
        auto data = ReadRadar();
        Process(data);
    }
    catch (const std::exception& e)
    {
        // ...
    }
}
```

**Correct (using ara::core::Result for deterministic error handling):**

```cpp
#include <ara/core/result.h>
#include <ara/core/error_code.h>

ara::core::Result<RadarData> ReadRadar()
{
    if (!radarHw.IsReady())
    {
        return ara::core::Result<RadarData>::FromError(
            MyErrors::MakeErrorCode(MyErrors::Errc::kHardwareNotReady));
    }
    return radarHw.GetData();
}

void ProcessRadar()
{
    auto result = ReadRadar();

    if (result.HasValue())
    {
        Process(result.Value());
    }
    else
    {
        HandleError(result.Error());
    }

    // Or using Value-or-default pattern:
    RadarData data = result.ValueOr(RadarData::SafeDefault());

    // Or using monadic chaining:
    result
        .Map([](const RadarData& d) { return Transform(d); })
        .InspectError([](const ara::core::ErrorCode& e) {
            LogError(e);
        });
}
```

### ErrorCode and ErrorDomain

**Incorrect (using magic numbers for errors):**

```cpp
int DoSomething()
{
    return -1;  // What does -1 mean? No domain, no message
}
```

**Correct (domain-specific error codes via ara::core::ErrorDomain):**

```cpp
#include <ara/core/error_domain.h>
#include <ara/core/error_code.h>

// Define application-specific error domain
enum class RadarErrc : ara::core::ErrorDomain::CodeType
{
    kHardwareNotReady = 1,
    kCalibrationFailed = 2,
    kSignalLost = 3
};

class RadarErrorDomain final : public ara::core::ErrorDomain
{
    static constexpr IdType kId = 0x8000'0001;

public:
    RadarErrorDomain() noexcept : ErrorDomain(kId) {}

    const char* Name() const noexcept override { return "Radar"; }

    const char* Message(CodeType errorCode) const noexcept override
    {
        switch (static_cast<RadarErrc>(errorCode))
        {
            case RadarErrc::kHardwareNotReady: return "Radar hardware not ready";
            case RadarErrc::kCalibrationFailed: return "Calibration failed";
            case RadarErrc::kSignalLost: return "Radar signal lost";
            default: return "Unknown radar error";
        }
    }
};

inline constexpr RadarErrorDomain g_radarErrorDomain;

inline ara::core::ErrorCode MakeErrorCode(RadarErrc code) noexcept
{
    return ara::core::ErrorCode(
        static_cast<ara::core::ErrorDomain::CodeType>(code),
        g_radarErrorDomain);
}
```

### Future / Promise Pattern

```cpp
#include <ara/core/future.h>
#include <ara/core/promise.h>

// ara::core::Future is similar to std::future but returns Result
ara::core::Future<SensorData> ReadSensorAsync()
{
    ara::core::Promise<SensorData> promise;
    auto future = promise.get_future();

    // Asynchronous operation on another thread/context
    scheduler.Post([p = std::move(promise)]() mutable {
        auto result = sensor.Read();
        if (result.HasValue())
        {
            p.set_value(result.Value());
        }
        else
        {
            p.SetError(result.Error());
        }
    });

    return future;
}

void ConsumeSensorData()
{
    auto future = ReadSensorAsync();

    // Non-blocking continuation
    future.then([](ara::core::Future<SensorData> f) {
        auto result = f.GetResult();  // Returns Result<SensorData, ErrorCode>
        if (result.HasValue())
        {
            ProcessSensor(result.Value());
        }
    });

    // Or blocking (use sparingly — only in non-real-time contexts)
    // auto result = future.GetResult();
}
```

### Span and StringView

```cpp
#include <ara/core/span.h>
#include <ara/core/string_view.h>

// ara::core::Span<T> — non-owning view over contiguous memory
// Avoids heap allocation for passing arrays/buffers
void ProcessBuffer(ara::core::Span<const uint8_t> data)
{
    for (auto byte : data)
    {
        Accumulate(byte);
    }
}

void Example()
{
    uint8_t buffer[256];
    FillBuffer(buffer);

    // Zero-copy — no allocation, no ownership transfer
    ProcessBuffer(ara::core::Span<const uint8_t>(buffer, 256));

    // From std::array
    std::array<uint8_t, 64> arr;
    ProcessBuffer(ara::core::Span<const uint8_t>(arr));
}

// ara::core::StringView — non-owning view over string data
void LogMessage(ara::core::StringView msg)
{
    // No heap allocation for string parameter
    ara::log::LogInfo() << msg;
}
```

Reference: AUTOSAR Adaptive R24-10 SWS_CoreTypes (ara::core), AP_SWS_CoreTypes
