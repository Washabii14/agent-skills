---
title: Smart Pointers for Ownership
impact: HIGH
impactDescription: prevents resource leaks
tags: autosar, cpp, smart-pointers, unique_ptr, ownership, resource-management
---

## Smart Pointers for Ownership

Use `std::unique_ptr` for exclusive ownership and `std::shared_ptr` only when shared ownership is genuinely needed. Never use raw owning pointers.

**Incorrect (raw owning pointer):**

```cpp
class DiagSession
{
public:
    void Start()
    {
        m_handler = new UdsHandler();
    }
    ~DiagSession() { delete m_handler; }  /* Easily forgotten or doubled */

private:
    UdsHandler* m_handler = nullptr;
};
```

**Correct (unique_ptr):**

```cpp
class DiagSession
{
public:
    void Start()
    {
        m_handler = std::make_unique<UdsHandler>();
    }

private:
    std::unique_ptr<UdsHandler> m_handler;
};
```

Reference: AUTOSAR C++14 Rule A18-1-1
