---
title: Clock Gating and Peripheral Power-Down
impact: MEDIUM
impactDescription: Leaving unused clocks and peripherals enabled wastes power, increases thermal load, and shortens battery life in 12V/48V systems
tags: power, clock-gating, peripheral, pll, clock-tree, power-domain, idle, current-consumption
---

## Clock Gating and Peripheral Power-Down

Disabling clocks to unused peripherals reduces dynamic power consumption proportionally. Reducing clock frequency during idle periods, using peripheral power domains, and reconfiguring PLLs for lower frequencies are key techniques for automotive power budgets.

**Incorrect (all peripherals clocked at maximum frequency regardless of usage):**

```c
/* WRONG: Everything enabled at boot, never disabled */
void SystemClock_Init_Bad(void)
{
    /* Enable ALL peripheral clocks — even unused ones */
    RCC->AHB1ENR = 0xFFFFFFFFU;
    RCC->APB1ENR = 0xFFFFFFFFU;
    RCC->APB2ENR = 0xFFFFFFFFU;

    /* PLL at maximum frequency — even during idle */
    RCC_PLL_Config(PLL_FREQ_MAX_160MHZ);
}
```

**Correct (enable only needed peripheral clocks, reduce in idle):**

```c
/* clock_config.h — peripheral clock control macros */
#define PERIPH_CLK_ENABLE(periph)   (RCC->AHB1ENR |= (periph))
#define PERIPH_CLK_DISABLE(periph)  (RCC->AHB1ENR &= ~(periph))
#define APB1_CLK_ENABLE(periph)     (RCC->APB1ENR |= (periph))
#define APB1_CLK_DISABLE(periph)    (RCC->APB1ENR &= ~(periph))
```

```c
/* clock_config.c — enable only required peripherals */
void SystemClock_Init(void)
{
    /* Enable only the GPIO ports actually used */
    PERIPH_CLK_ENABLE(RCC_AHB1ENR_GPIOAEN |   /* Port A: CAN Tx/Rx */
                      RCC_AHB1ENR_GPIOBEN |    /* Port B: SPI, I2C */
                      RCC_AHB1ENR_GPIOCEN);    /* Port C: ADC inputs */
    /* Ports D-K: NOT enabled — saves ~1mA each at 100MHz */

    /* Enable communication peripherals */
    APB1_CLK_ENABLE(RCC_APB1ENR_CAN1EN);       /* CAN1 */
    APB1_CLK_ENABLE(RCC_APB1ENR_SPI2EN);       /* SPI2 for sensor */
    /* USART, I2C, TIM2-7, etc.: NOT enabled */

    /* DMA only for active channels */
    PERIPH_CLK_ENABLE(RCC_AHB1ENR_DMA1EN);     /* DMA1 for SPI */
    /* DMA2: NOT enabled */
}
```

**Dynamic clock frequency scaling:**

```c
/* Reduce system clock during idle/low-load periods */
typedef enum {
    CLK_PROFILE_RUN,      /* 160 MHz — full processing */
    CLK_PROFILE_REDUCED,  /*  80 MHz — light processing */
    CLK_PROFILE_IDLE,     /*  16 MHz — minimal, waiting for events */
} ClockProfile_t;

void Clock_SetProfile(ClockProfile_t profile)
{
    /* Adjust flash wait states BEFORE increasing frequency,
       AFTER decreasing frequency */
    switch (profile)
    {
        case CLK_PROFILE_RUN:
            FLASH->ACR = (FLASH->ACR & ~FLASH_ACR_LATENCY) | FLASH_ACR_LATENCY_5WS;
            RCC_PLL_Reconfigure(PLL_MUL_40, PLL_DIV_1);  /* 160 MHz */
            break;

        case CLK_PROFILE_REDUCED:
            RCC_PLL_Reconfigure(PLL_MUL_20, PLL_DIV_1);  /* 80 MHz */
            FLASH->ACR = (FLASH->ACR & ~FLASH_ACR_LATENCY) | FLASH_ACR_LATENCY_2WS;
            break;

        case CLK_PROFILE_IDLE:
            RCC_SwitchToHSI();  /* Bypass PLL, use 16 MHz internal RC */
            RCC_PLL_Disable();  /* PLL off saves ~2mA */
            FLASH->ACR = (FLASH->ACR & ~FLASH_ACR_LATENCY) | FLASH_ACR_LATENCY_0WS;
            break;
    }

    /* Update SystemCoreClock for HAL/delay functions */
    SystemCoreClockUpdate();

    /* Reconfigure tick timer for new frequency */
    SysTick_Config(SystemCoreClock / 1000U);
}
```

**Peripheral power domain control:**

```c
/* Some MCUs have independent power domains for peripheral groups */
void Power_ConfigureDomains(void)
{
    /* Disable ADC power domain when not sampling */
    PWR->CR2 &= ~PWR_CR2_ADCDC1EN;

    /* Disable USB power domain (not used in this ECU) */
    PWR->CR2 &= ~PWR_CR2_USBPDEN;

    /* Keep CAN transceiver power domain enabled */
    PWR->CR2 |= PWR_CR2_CANPDEN;
}

/* Runtime peripheral power management */
void ADC_AcquirePower(void)
{
    PWR->CR2 |= PWR_CR2_ADCDC1EN;
    APB2_CLK_ENABLE(RCC_APB2ENR_ADC1EN);
    /* Wait for voltage regulator stabilization */
    Delay_us(ADC_REGULATOR_STARTUP_US);
    ADC1->CR |= ADC_CR_ADEN;
}

void ADC_ReleasePower(void)
{
    ADC1->CR |= ADC_CR_ADDIS;
    while (ADC1->CR & ADC_CR_ADEN) { /* wait */ }
    APB2_CLK_DISABLE(RCC_APB2ENR_ADC1EN);
    PWR->CR2 &= ~PWR_CR2_ADCDC1EN;
}
```

**C++ RAII pattern for peripheral power management:**

```cpp
class PeripheralPowerGuard
{
public:
    explicit PeripheralPowerGuard(volatile uint32_t& clkReg, uint32_t clkBit)
        : clk_reg_(clkReg), clk_bit_(clkBit)
    {
        clk_reg_ |= clk_bit_;
        __DSB();  /* Ensure clock is enabled before peripheral access */
    }

    ~PeripheralPowerGuard()
    {
        clk_reg_ &= ~clk_bit_;
    }

    PeripheralPowerGuard(const PeripheralPowerGuard&) = delete;
    PeripheralPowerGuard& operator=(const PeripheralPowerGuard&) = delete;

private:
    volatile uint32_t& clk_reg_;
    uint32_t clk_bit_;
};

// Usage: SPI clock enabled only for the transfer duration
void Sensor_Read(uint8_t* buf, size_t len)
{
    PeripheralPowerGuard spi_clk(RCC->APB1ENR, RCC_APB1ENR_SPI2EN);
    SPI_Transfer(SPI2, buf, len);
}  // SPI2 clock automatically disabled here
```

After disabling a peripheral clock, insert a `__DSB()` barrier before accessing other peripherals — the bus may need a cycle to propagate the clock gate. When reducing PLL frequency, adjust flash wait states to avoid instruction fetch corruption.

Reference: MCU Reference Manual — RCC/Power chapters; ISO 26262 Part 5 (Hardware power management); OEM sleep current specifications
