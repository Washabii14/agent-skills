---
title: Hardware Abstraction Layer
impact: HIGH
impactDescription: HAL enables portability across MCU families and mockability for unit testing
tags: arch, hal, abstraction, portability, testability, driver
---

## Hardware Abstraction Layer

Isolate hardware dependencies behind abstract interfaces for portability across MCU families and unit test mockability. Application logic should never access hardware registers directly.

**Incorrect (direct hardware access in application code):**

```c
void SetIndicatorLed(uint8_t state)
{
    *(volatile uint32_t *)0x40021018U = state ? 0x2000U : 0U;  /* GPIO register */
}
```

**Correct (HAL interface with platform-specific implementation):**

```c
/* HAL interface — platform independent */
typedef struct
{
    Std_ReturnType (*Init)(const GpioConfig_t *config);
    Std_ReturnType (*WritePin)(uint8_t port, uint8_t pin, uint8_t level);
    uint8_t        (*ReadPin)(uint8_t port, uint8_t pin);
} Gpio_DriverApi_t;

/* Platform-specific implementation registered at startup */
extern const Gpio_DriverApi_t Gpio_Driver;
```

The application calls `Gpio_Driver.WritePin(PORT_B, PIN_13, HIGH)` without knowing the MCU family. During unit testing, a mock implementation can be substituted at link time. This pattern follows the AUTOSAR MCAL (Microcontroller Abstraction Layer) architecture.

Reference: AUTOSAR Classic Platform — MCAL Specification, ISO 26262 Part 6
