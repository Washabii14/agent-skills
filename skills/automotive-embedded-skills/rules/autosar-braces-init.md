---
title: Prefer Braced Initialization
impact: MEDIUM
impactDescription: prevents narrowing conversions
tags: autosar, cpp, initialization, braces, narrowing, type-safety
---

## Prefer Braced Initialization

Braced initialization (uniform initialization) prevents implicit narrowing conversions that silently truncate values. The compiler will produce an error for narrowing conversions within braces.

**Incorrect (allows narrowing):**

```cpp
uint8_t channel = 300;  /* Silently truncated to 44 */
```

**Correct (braces catch narrowing at compile time):**

```cpp
uint8_t channel{300};  /* Compilation error: narrowing conversion */
```

Reference: AUTOSAR C++14 Rule A8-5-2
