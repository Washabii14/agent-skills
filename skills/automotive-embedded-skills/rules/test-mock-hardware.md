---
title: Mock Hardware Dependencies
impact: HIGH
impactDescription: Enables host-based testing by abstracting hardware access
tags: test, mock, hardware, hal, host-testing
---

## Mock Hardware Dependencies

Use function pointers or link-time substitution to mock hardware for host-based testing. Tests that require real hardware are slow, expensive, and hard to automate.

**Incorrect (direct hardware access, untestable):**

```c
uint16_t ReadTemperature(void)
{
    return *(volatile uint16_t *)0x40012400U;
}
```

**Correct (mockable via HAL interface):**

```c
/* HAL interface — platform independent */
typedef struct
{
    Std_ReturnType (*Init)(const GpioConfig_t *config);
    Std_ReturnType (*WritePin)(uint8_t port, uint8_t pin, uint8_t level);
    uint8_t        (*ReadPin)(uint8_t port, uint8_t pin);
} Gpio_DriverApi_t;

/* Production: real hardware */
extern const Gpio_DriverApi_t Gpio_Driver;

/* Test: mock implementation */
static uint8_t mockPinState[MAX_PORTS][MAX_PINS];
uint8_t Mock_ReadPin(uint8_t port, uint8_t pin)
{
    return mockPinState[port][pin];
}
```

Use CMock, FFF (Fake Function Framework), or manual link-time substitution. The HAL pattern from Section 9.1 naturally supports this.
