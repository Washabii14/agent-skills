---
title: Prefer Message Queues Over Shared Memory
impact: MEDIUM
impactDescription: Message queues provide inherent synchronization and decouple sender/receiver timing
tags: rtos, message-queue, inter-task-communication, decoupling, concurrency
---

## Prefer Message Queues Over Shared Memory

Message queues provide inherent synchronization and decouple sender from receiver timing. Unlike shared memory with manual locking, queues eliminate the risk of forgotten locks and data races.

**Incorrect (shared memory with manual synchronization):**

```c
static volatile SensorData_t g_sharedData;
static volatile boolean g_dataReady = FALSE;

void Task_Producer(void)
{
    SensorData_t data;
    ReadSensor(&data);
    OS_EnterCritical();
    g_sharedData = data;
    g_dataReady = TRUE;
    OS_ExitCritical();
}

void Task_Consumer(void)
{
    if (g_dataReady)
    {
        OS_EnterCritical();
        SensorData_t local = g_sharedData;
        g_dataReady = FALSE;
        OS_ExitCritical();
        ProcessData(&local);
    }
}
```

**Correct (message queue decouples tasks):**

```c
static OS_Queue_t sensorQueue;

void Task_Producer(void)
{
    SensorData_t data;
    ReadSensor(&data);
    OS_QueueSend(&sensorQueue, &data, OS_NO_WAIT);
}

void Task_Consumer(void)
{
    SensorData_t data;
    if (OS_QueueReceive(&sensorQueue, &data, QUEUE_TIMEOUT_MS) == OS_OK)
    {
        ProcessData(&data);
    }
}
```

Message queues handle synchronization internally, support multiple producers/consumers, and allow the RTOS to block waiting tasks efficiently. Use queues for inter-task data transfer and events for signaling.

Reference: AUTOSAR OS Specification — Message and event handling
