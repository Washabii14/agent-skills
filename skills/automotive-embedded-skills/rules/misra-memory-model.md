---
title: MISRA Memory Model — Volatile, Atomic Access, Memory Barriers, Object Overlap
impact: HIGH
impactDescription: prevents data races, ensures correct hardware register access, and enforces memory ordering
tags: misra, volatile, atomic, memory-barrier, memory-model, register-access, embedded, safety
---

## MISRA Memory Model — Volatile, Atomic Access, Memory Barriers, Object Overlap

Correct use of volatile, atomic operations, and memory barriers is critical for reliable hardware interaction and interrupt-safe data sharing. These rules prevent the compiler from optimizing away essential accesses and ensure correct memory ordering.

### Rules Covered

| Rule | Category | Summary |
|------|----------|---------|
| 19.1 | Mandatory | Object shall not be assigned or copied to an overlapping object |
| 19.2 | Advisory | union keyword should not be used |
| 2.7  | Advisory | There should be no unused parameters (related: volatile read pattern) |

Additional volatile/atomic guidance drawn from:
- MISRA C:2012 Dir 4.6 — typedefs for basic numerical types (volatile-qualified types)
- MISRA C:2012 Rule 11.8 — shall not cast away volatile qualification
- MISRA C++:2023 Rules 6.7.1 (volatile semantics), 30.0.1–30.0.2 (concurrency)
- ISO 26262-6 Table 1 — measures for preventing interference between software elements

---

### Top Violations and Patterns

**Volatile for Hardware Registers (most critical)**

Incorrect (missing volatile — compiler may optimize out reads):

```c
#define STATUS_REG  (*(uint32_t *)0x40020010UL)

void WaitReady(void) {
    while ((STATUS_REG & READY_BIT) == 0U) {
        /* compiler may read STATUS_REG once and loop forever */
    }
}
```

Correct (volatile-qualified register pointer):

```c
#define STATUS_REG  (*(volatile uint32_t *)0x40020010UL)

void WaitReady(void) {
    while ((STATUS_REG & READY_BIT) == 0U) {
        /* volatile forces re-read every iteration */
    }
}
```

Recommended pattern (typed register struct):

```c
typedef struct {
    volatile uint32_t CR;
    volatile uint32_t SR;
    volatile uint32_t DR;
} USART_TypeDef;

#define USART1  ((USART_TypeDef *)0x40011000UL)

void WaitTxReady(void) {
    while ((USART1->SR & USART_SR_TXE) == 0U) {
        /* each SR read is volatile */
    }
}
```

---

**Volatile for ISR-Shared Variables**

Incorrect (non-volatile shared variable):

```c
static uint8_t isr_flag = 0U;

void ISR_Timer(void) {
    isr_flag = 1U;
}

void MainLoop(void) {
    while (isr_flag == 0U) {  /* compiler may hoist read out of loop */
    }
    isr_flag = 0U;
    ProcessTimerEvent();
}
```

Correct (volatile + critical section):

```c
static volatile uint8_t isr_flag = 0U;

void ISR_Timer(void) {
    isr_flag = 1U;
}

void MainLoop(void) {
    while (isr_flag == 0U) {
        /* volatile forces re-read */
    }
    DisableInterrupts();
    isr_flag = 0U;
    EnableInterrupts();
    ProcessTimerEvent();
}
```

C++ equivalent (std::atomic — preferred for multi-core):

```cpp
#include <atomic>

static std::atomic<uint8_t> isr_flag{0U};

void ISR_Timer() {
    isr_flag.store(1U, std::memory_order_release);
}

void MainLoop() {
    while (isr_flag.load(std::memory_order_acquire) == 0U) { }
    isr_flag.store(0U, std::memory_order_relaxed);
    ProcessTimerEvent();
}
```

---

**Rule 19.1 — No Overlapping Object Copy**

Incorrect (memcpy with overlapping buffers):

```c
uint8_t buffer[64];
/* shift buffer contents left by 4 bytes */
(void)memcpy(&buffer[0], &buffer[4], 60U);  /* UB: source and dest overlap */
```

Correct (memmove for overlapping regions):

```c
uint8_t buffer[64];
(void)memmove(&buffer[0], &buffer[4], 60U);  /* handles overlap correctly */
```

---

**Rule 19.2 — Union Avoidance (type punning)**

Incorrect (union for type punning — UB in strict C):

```c
typedef union {
    float f;
    uint32_t u;
} FloatBits_t;

uint32_t FloatToRaw(float val) {
    FloatBits_t fb;
    fb.f = val;
    return fb.u;  /* reading non-active member — UB in C */
}
```

Correct (memcpy for type punning):

```c
uint32_t FloatToRaw(float val) {
    uint32_t raw;
    (void)memcpy(&raw, &val, sizeof(raw));
    return raw;
}
```

C++ equivalent:

```cpp
#include <bit>
uint32_t FloatToRaw(float val) {
    return std::bit_cast<uint32_t>(val);  // C++20 — defined behavior
}
```

---

**Memory Barrier Pattern for Multi-Core**

```c
/* ARM Cortex-specific barriers */
#define DMB()  __asm volatile ("dmb" ::: "memory")
#define DSB()  __asm volatile ("dsb" ::: "memory")
#define ISB()  __asm volatile ("isb" ::: "memory")

void WriteSharedData(SharedData_t *shared, const uint8_t *data, uint16_t len) {
    (void)memcpy(shared->buffer, data, len);
    shared->length = len;
    DMB();                    /* ensure writes are visible before flag */
    shared->ready_flag = 1U;  /* must be volatile */
}

void ReadSharedData(const SharedData_t *shared, uint8_t *out, uint16_t *out_len) {
    while (shared->ready_flag == 0U) { }  /* must be volatile */
    DMB();                                 /* ensure flag read before data read */
    *out_len = shared->length;
    (void)memcpy(out, shared->buffer, shared->length);
}
```

---

**Volatile Read-Discard Pattern (clearing status registers)**

```c
/* Some hardware requires reading a register to clear a flag */
static inline void ClearStatusFlag(volatile uint32_t *reg) {
    (void)*reg;  /* volatile read — value intentionally discarded */
}
```

---

### Common Deviations

**Deviation: Rule 19.2 — Unions for Memory-Mapped Register Overlays**

```
Deviation ID:    DEV-MEM-001
Rule:            MISRA C:2012 Rule 19.2 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           HAL/MCAL register overlay definitions
Justification:   Hardware vendors define register banks where the same
                 physical register has different bit-field interpretations
                 depending on access mode (e.g., UART Control Register in
                 TX vs RX mode). A union of structs is the standard way
                 to model this in C without manual bit shifting.
Conditions:      - Only in vendor-supplied or HAL-layer register definitions
                 - Union members must be volatile-qualified
                 - Application code must never directly use the union — only
                   through typed HAL accessor functions
                 - Each access must use the correct union member for the current mode
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 19.2 — Unions for Communication Protocol Variant Messages**

```
Deviation ID:    DEV-MEM-002
Rule:            MISRA C:2012 Rule 19.2 (Advisory)
Category:        Advisory → Permitted with conditions
Scope:           CAN/LIN/Ethernet protocol message parsing
Justification:   Automotive communication protocols (CAN, UDS, SOME/IP) use
                 discriminated unions where a message type field determines
                 which payload variant is active. This is a natural union
                 use case with a discriminant ensuring only the active
                 member is accessed.
Conditions:      - Union must have an associated discriminant field (e.g., msg_type)
                 - Access functions must check discriminant before reading member
                 - All members must have the same size or be documented clearly
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Rules 19.1–19.2, 11.8, Dir 4.6; MISRA C++:2023 Rules 6.7.1, 30.0.x
