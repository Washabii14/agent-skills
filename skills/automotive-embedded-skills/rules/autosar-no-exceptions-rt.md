---
title: No Exceptions in Real-Time Contexts
impact: HIGH
impactDescription: deterministic error handling
tags: autosar, cpp, exceptions, real-time, error-handling, determinism
---

## No Exceptions in Real-Time Contexts

C++ exceptions have non-deterministic cost due to stack unwinding. In hard real-time contexts, use return codes or `ara::core::Result<T,E>`.

**Incorrect (throwing in cyclic task):**

```cpp
void Cyclic10ms()
{
    auto data = sensorDriver.Read();
    if (!data.isValid)
    {
        throw std::runtime_error("Sensor failure");  /* Non-deterministic */
    }
}
```

**Correct (Result type):**

```cpp
ara::core::Result<SensorData> Cyclic10ms()
{
    auto result = sensorDriver.Read();
    if (!result.HasValue())
    {
        return ara::core::Result<SensorData>::FromError(
            SensorErrc::kReadFailure);
    }
    return result;
}
```

Reference: AUTOSAR C++14 Rule A15-0-1
