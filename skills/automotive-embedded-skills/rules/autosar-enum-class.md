---
title: Use enum class Over Plain enum
impact: MEDIUM
impactDescription: prevents implicit conversion, scoped names
tags: autosar, cpp, enum, scoped-enum, type-safety, naming
---

## Use enum class Over Plain enum

Scoped enumerations (`enum class`) prevent implicit conversions to integers and avoid name collisions across enumerator sets.

**Incorrect (plain enum pollutes scope):**

```c
enum Color { RED, GREEN, BLUE };
enum TrafficLight { RED, YELLOW, GREEN };  /* Compilation error: RED/GREEN conflict */
```

**Correct (scoped enum):**

```cpp
enum class Color : uint8_t { Red, Green, Blue };
enum class TrafficLight : uint8_t { Red, Yellow, Green };

auto c = Color::Red;  /* No conflict, no implicit conversion */
```

Reference: AUTOSAR C++14 Rule A7-2-2
