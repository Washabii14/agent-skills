---
title: Avoid Unions, Use std::variant
impact: MEDIUM
impactDescription: type-safe, no UB from active member confusion
tags: autosar, cpp, union, variant, type-safety, undefined-behavior
---

## Avoid Unions, Use std::variant

Unions are type-unsafe — reading from an inactive member is undefined behavior. Use `std::variant` for type-safe discriminated unions.

**Incorrect (union with type confusion risk):**

```cpp
union SensorValue
{
    float    temperature;
    uint32_t pressure;
    int16_t  rawAdc;
};
```

**Correct (type-safe variant):**

```cpp
using SensorValue = std::variant<float, uint32_t, int16_t>;

SensorValue val = 25.5f;
if (auto* temp = std::get_if<float>(&val))
{
    ProcessTemperature(*temp);
}
```

Reference: AUTOSAR C++14 Rule A9-5-1
