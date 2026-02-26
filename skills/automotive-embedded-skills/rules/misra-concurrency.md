---
title: MISRA Concurrency — Thread Safety, Shared Data, MISRA C:2012 Amendment 4
impact: HIGH
impactDescription: prevents data races, ensures atomic access to shared resources, enforces safe concurrency patterns
tags: misra, concurrency, thread-safety, data-race, mutex, atomic, rtos, shared-data, safety
---

## MISRA Concurrency — Thread Safety, Shared Data, MISRA C:2012 Amendment 4

MISRA C:2012 Amendment 4 (published 2023) adds rules for concurrent programming, addressing multi-core automotive ECUs and RTOS-based systems. Data races are undefined behavior and a major safety risk.

### Rules Covered (MISRA C:2012 Amendment 4)

| Rule | Category | Summary |
|------|----------|---------|
| 23.1 | Required | A thread shall not be created or terminated within a function that acquires a resource |
| 23.2 | Required | A thread shall not be detached |
| 23.3 | Required | A thread shall be joined or detached before its id goes out of scope |
| 23.4 | Required | A thread shall not be started with a pointer to an automatic variable |
| 23.5 | Required | A thread shall not lock a mutex it already holds (no self-deadlock) |
| 23.6 | Required | A mutex shall not be destroyed while locked |
| 23.7 | Required | A mutex shall be locked and unlocked in the same thread |
| 23.8 | Required | A mutex shall be locked at most once by a thread (non-recursive) |

Additional concurrency guidance:
- MISRA C++:2023 Rules 30.0.1 (data race prevention), 30.0.2 (synchronization)
- ISO 26262-6 clause 8 (freedom from interference for multi-core)

---

### Top Violations and Patterns

**Data Race on Shared Variable (most critical)**

Incorrect (unprotected shared state):

```c
static uint16_t engine_rpm;  /* accessed from Task_10ms and Task_100ms */

void Task_10ms(void) {
    engine_rpm = ReadRpmSensor();  /* write without lock */
}

void Task_100ms(void) {
    if (engine_rpm > RPM_LIMIT) {  /* read without lock — data race */
        ActivateLimiter();
    }
}
```

Correct (mutex-protected access):

```c
static uint16_t engine_rpm;
static osMutexId_t rpm_mutex;

void Task_10ms(void) {
    (void)osMutexAcquire(rpm_mutex, osWaitForever);
    engine_rpm = ReadRpmSensor();
    (void)osMutexRelease(rpm_mutex);
}

void Task_100ms(void) {
    (void)osMutexAcquire(rpm_mutex, osWaitForever);
    uint16_t local_rpm = engine_rpm;
    (void)osMutexRelease(rpm_mutex);

    if (local_rpm > RPM_LIMIT) {
        ActivateLimiter();
    }
}
```

Alternative (atomic for simple types — no mutex overhead):

```c
#include <stdatomic.h>

static _Atomic uint16_t engine_rpm;

void Task_10ms(void) {
    atomic_store(&engine_rpm, ReadRpmSensor());
}

void Task_100ms(void) {
    uint16_t local_rpm = atomic_load(&engine_rpm);
    if (local_rpm > RPM_LIMIT) {
        ActivateLimiter();
    }
}
```

C++ equivalent:

```cpp
#include <atomic>
#include <mutex>

static std::atomic<uint16_t> engine_rpm{0U};

void Task10ms() {
    engine_rpm.store(ReadRpmSensor(), std::memory_order_release);
}

void Task100ms() {
    auto local_rpm = engine_rpm.load(std::memory_order_acquire);
    if (local_rpm > kRpmLimit) {
        ActivateLimiter();
    }
}
```

---

**Rule 23.5 — No Self-Deadlock**

Incorrect (recursive lock attempt):

```c
void UpdateState(void) {
    (void)osMutexAcquire(state_mutex, osWaitForever);
    state.counter++;
    LogStateChange();  /* calls UpdateState internally — deadlock */
    (void)osMutexRelease(state_mutex);
}

void LogStateChange(void) {
    (void)osMutexAcquire(state_mutex, osWaitForever);  /* same mutex — deadlock */
    /* ... log ... */
    (void)osMutexRelease(state_mutex);
}
```

Correct (separate locked/unlocked internal API):

```c
static void UpdateState_Locked(void) {
    state.counter++;
    LogStateChange_Locked();
}

void UpdateState(void) {
    (void)osMutexAcquire(state_mutex, osWaitForever);
    UpdateState_Locked();
    (void)osMutexRelease(state_mutex);
}

static void LogStateChange_Locked(void) {
    /* already holds state_mutex — no re-acquisition */
}
```

---

**Rule 23.4 — No Pointer to Automatic Variable in Thread**

Incorrect (stack variable outlived by thread):

```c
void StartProcessing(void) {
    uint8_t config_data[32];
    LoadConfig(config_data);
    osThreadNew(ProcessingThread, config_data, NULL);
    /* config_data goes out of scope — thread holds dangling pointer */
}
```

Correct (static or heap-free persistent storage):

```c
static uint8_t processing_config[32];

void StartProcessing(void) {
    LoadConfig(processing_config);
    osThreadNew(ProcessingThread, processing_config, NULL);
}
```

---

**Consistent Lock Ordering to Prevent Deadlock**

Incorrect (inconsistent acquisition order):

```c
/* Task A */
osMutexAcquire(mutex_sensor, osWaitForever);
osMutexAcquire(mutex_actuator, osWaitForever);  /* order: sensor → actuator */

/* Task B */
osMutexAcquire(mutex_actuator, osWaitForever);
osMutexAcquire(mutex_sensor, osWaitForever);    /* order: actuator → sensor — DEADLOCK */
```

Correct (global lock hierarchy):

```c
/*
 * Lock hierarchy (always acquire in this order):
 *   Level 1: mutex_sensor
 *   Level 2: mutex_actuator
 *   Level 3: mutex_comm
 */

/* Task A */
osMutexAcquire(mutex_sensor, osWaitForever);
osMutexAcquire(mutex_actuator, osWaitForever);

/* Task B — same order */
osMutexAcquire(mutex_sensor, osWaitForever);
osMutexAcquire(mutex_actuator, osWaitForever);
```

C++ RAII pattern:

```cpp
void CriticalSection() {
    std::scoped_lock lock(mutex_sensor, mutex_actuator);  // deadlock-free
    // ... access shared data ...
}
```

---

**ISR-to-Task Communication (interrupt-safe pattern)**

```c
static volatile uint8_t isr_event_flag = 0U;
static osSemaphoreId_t event_sem;

void ISR_ExternalEvent(void) {
    isr_event_flag = 1U;
    osSemaphoreRelease(event_sem);  /* ISR-safe RTOS call */
}

void Task_EventHandler(void) {
    for (;;) {
        (void)osSemaphoreAcquire(event_sem, osWaitForever);
        if (isr_event_flag != 0U) {
            DisableInterrupts();
            isr_event_flag = 0U;
            EnableInterrupts();
            HandleExternalEvent();
        }
    }
}
```

---

### Common Deviations

**Deviation: Rule 23.8 — Recursive Mutex for Legacy Module Integration**

```
Deviation ID:    DEV-CONC-001
Rule:            MISRA C:2012 Amd4 Rule 23.8 (Required)
Category:        Required → Permitted with conditions
Scope:           Integration shim for legacy third-party library
Justification:   A third-party library module (e.g., TCP/IP stack) uses
                 internal recursive locking patterns that cannot be
                 refactored. The integration shim wraps the library's
                 mutex with a recursive mutex to prevent deadlock.
Conditions:      - Only for documented third-party code that cannot be modified
                 - Recursive mutex must be explicitly created with osMutexRecursive
                 - Maximum recursion depth must be bounded and documented
                 - Must have a plan to replace with non-recursive design in next version
Approved by:     Safety Manager, [date]
```

**Deviation: Rule 23.2 — Detached Thread for Background Diagnostics**

```
Deviation ID:    DEV-CONC-002
Rule:            MISRA C:2012 Amd4 Rule 23.2 (Required)
Category:        Required → Permitted with conditions
Scope:           ASIL QM background diagnostic task only
Justification:   The background diagnostic thread runs for the entire ECU
                 lifetime and is never intended to be joined. Detaching
                 it prevents resource accumulation from unjoinable threads.
Conditions:      - Only for QM-rated lifetime threads (not safety-critical)
                 - Thread must have its own error handling (no unhandled failures)
                 - Thread stack must be statically allocated
Approved by:     Safety Manager, [date]
```

Reference: MISRA C:2012 Amendment 4 Rules 23.1–23.8; MISRA C++:2023 Rules 30.0.x; ISO 26262-6 clause 8
