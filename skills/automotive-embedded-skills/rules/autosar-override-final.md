---
title: Always Use override and final
impact: MEDIUM
impactDescription: catches signature mismatches at compile time
tags: autosar, cpp, override, final, virtual, polymorphism, compile-time
---

## Always Use override and final

Mark all overriding virtual functions with `override` to catch signature mismatches at compile time. Use `final` to prevent further overriding when appropriate.

**Incorrect (silent signature mismatch):**

```cpp
class SensorBase
{
public:
    virtual void Init(uint8_t channel);
};

class TempSensor : public SensorBase
{
public:
    virtual void Init(int channel);  /* Different signature — new function, not override */
};
```

**Correct (compiler catches the error):**

```cpp
class TempSensor : public SensorBase
{
public:
    void Init(uint8_t channel) override;  /* Compiler verifies base signature */
};
```

Reference: AUTOSAR C++14 Rule A10-3-1
