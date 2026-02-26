---
title: const-Correctness Throughout Interfaces
impact: MEDIUM
impactDescription: enables compiler optimization, documents intent
tags: autosar, cpp, const, interface, optimization, immutability
---

## const-Correctness Throughout Interfaces

Apply `const` to all parameters, member functions, and return values that should not be modified. This enables compiler optimizations and clearly documents intent.

**Incorrect (missing const):**

```cpp
class SensorManager
{
public:
    float GetTemperature()     { return m_temp; }
    void ProcessData(float* data, int count);
};
```

**Correct (const-correct):**

```cpp
class SensorManager
{
public:
    float GetTemperature() const { return m_temp; }
    void ProcessData(const float* data, std::size_t count);
};
```

Reference: AUTOSAR C++14 Rule A7-1-1
