---
title: Layered Architecture Pattern
impact: HIGH
impactDescription: Layered architecture enforces separation of concerns and is required for AUTOSAR compliance
tags: arch, layered-architecture, autosar, mcal, abstraction, maintainability
---

## Layered Architecture Pattern

Follow AUTOSAR layered architecture: MCAL → ECU Abstraction → Service Layer → RTE → Application SWC. Each layer may only call the layer directly below it. Never bypass layers.

**Incorrect (application directly accesses hardware):**

```c
/* Application layer directly accesses MCAL — skips ECU abstraction and services */
void AppTask(void)
{
    uint32_t adcRaw = *(volatile uint32_t *)ADC1_DR_ADDR;
    float voltage = (float)adcRaw * 3.3f / 4096.0f;
    ProcessVoltage(voltage);
}
```

**Correct (each layer calls only the layer below):**

```c
/* MCAL Layer — hardware register access */
uint32_t Adc_ReadChannel(uint8_t channel);

/* ECU Abstraction Layer — abstracts MCU-specific ADC */
float EcuAbstr_ReadVoltage(uint8_t channel);

/* Service Layer — provides calibrated sensor value */
float SensorSvc_GetTemperature(void);

/* Application SWC — uses service interface via RTE */
void AppTask(void)
{
    float temperature = Rte_Read_SensorSvc_Temperature();
    ProcessTemperature(temperature);
}
```

Layer boundaries enforce that hardware changes only affect MCAL, MCU migration only affects ECU Abstraction, and application logic remains portable. This is the foundation of AUTOSAR Classic and Adaptive platform architecture.

Reference: AUTOSAR Layered Software Architecture, ISO 26262 Part 6 — Software architectural design
