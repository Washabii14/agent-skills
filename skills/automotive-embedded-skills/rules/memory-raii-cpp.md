---
title: RAII for Resource Management in C++
impact: HIGH
impactDescription: automatic cleanup, exception-safe
tags: memory, raii, cpp, resource-management, scoped, cleanup
---

## RAII for Resource Management in C++

In C++ embedded code, use RAII to ensure resources are released even when execution paths change. This prevents resource leaks from early returns or exceptions.

**Incorrect (manual resource management):**

```cpp
void TransmitFrame(const uint8_t* data, size_t len)
{
    auto* buffer = HwBufferPool::Acquire();
    if (buffer == nullptr) { return; }

    std::memcpy(buffer->data, data, len);
    HwDriver::Send(buffer);
    /* If Send() throws or an early return is added, buffer leaks */
    HwBufferPool::Release(buffer);
}
```

**Correct (RAII wrapper):**

```cpp
class ScopedHwBuffer
{
public:
    ScopedHwBuffer() : m_buffer(HwBufferPool::Acquire()) {}
    ~ScopedHwBuffer() { if (m_buffer) { HwBufferPool::Release(m_buffer); } }

    ScopedHwBuffer(const ScopedHwBuffer&) = delete;
    ScopedHwBuffer& operator=(const ScopedHwBuffer&) = delete;

    HwBuffer* Get() { return m_buffer; }
    bool IsValid() const { return m_buffer != nullptr; }

private:
    HwBuffer* m_buffer;
};

void TransmitFrame(const uint8_t* data, size_t len)
{
    ScopedHwBuffer buffer;
    if (!buffer.IsValid()) { return; }

    std::memcpy(buffer.Get()->data, data, len);
    HwDriver::Send(buffer.Get());
}
```

Reference: AUTOSAR C++14 Rule A18-1-1
