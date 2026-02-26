---
title: Ensure Proper Data Structure Alignment
impact: MEDIUM
impactDescription: prevents hard faults, optimizes access speed
tags: memory, alignment, struct, packed, arm, hard-fault, padding
---

## Ensure Proper Data Structure Alignment

Misaligned access causes hard faults on ARM Cortex-M and performance degradation on other architectures. Order struct members from largest to smallest to minimize padding.

**Incorrect (packed struct with misaligned fields):**

```c
typedef struct __attribute__((packed))
{
    uint8_t  id;
    uint32_t timestamp;  /* Misaligned at offset 1 */
    uint16_t value;
} SensorData_t;
```

**Correct (naturally aligned struct):**

```c
typedef struct
{
    uint32_t timestamp;  /* Offset 0: aligned */
    uint16_t value;      /* Offset 4: aligned */
    uint8_t  id;         /* Offset 6 */
    uint8_t  reserved;   /* Padding made explicit */
} SensorData_t;
```

Use `__attribute__((packed))` only for wire protocol structures, and access fields through `memcpy()` to avoid alignment faults.

Reference: ARM Architecture Reference Manual, AUTOSAR C++14 Rule A9-6-1
