---
title: Single Responsibility Task Design
impact: MEDIUM
impactDescription: Each RTOS task should have one clear responsibility for maintainability and analyzability
tags: rtos, task-design, concurrency, maintainability, single-responsibility
---

## Single Responsibility Task Design

Each RTOS task should have a single responsibility with a clear, analyzable execution path. Mixing unrelated functionality in one task makes worst-case execution time analysis difficult and creates unnecessary coupling between subsystems.

**Incorrect (multiple responsibilities in one task):**

```c
void Task_Mixed(void)
{
    ReadSensors();
    ProcessDiagnostics();
    UpdateDisplay();
    HandleCommunication();
    LogToFlash();
}
```

**Correct (single responsibility per task):**

```c
void Task_SensorProcessing(void)
{
    ReadSensors();
    FilterSensorData();
    ValidatePlausibility();
}

void Task_Communication(void)
{
    TransmitCanMessages();
    ReceiveCanMessages();
}

void Task_Diagnostics(void)
{
    ProcessDiagRequests();
    UpdateDtcStatus();
}
```

Single-responsibility tasks enable independent WCET analysis, clearer priority assignment, and easier unit testing. When a task grows beyond its scope, split it into separate tasks with well-defined inter-task communication (message queues, events).

Reference: ISO 26262 Part 6 — Software architectural design principles
