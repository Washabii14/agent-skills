---
title: Bare-Metal Boot Sequence
impact: HIGH
impactDescription: Incorrect startup ordering causes undefined behavior, stack corruption, or silent peripheral failures before any diagnostic output is possible
tags: boot, bare-metal, startup, vector-table, BSS, stack-pointer, linker-script, c-runtime
---

## Bare-Metal Boot Sequence

The bare-metal boot sequence must follow a strict order: reset vector → stack pointer setup → vector table relocation → C runtime init (BSS clear, data copy) → `main()` → peripheral init → scheduler start. Deviating from this order causes hard-to-diagnose failures because no debug infrastructure is available yet.

**Incorrect (using global initialized data before C runtime init):**

```c
/* startup.c — WRONG: assumes .data and .bss are ready */
volatile uint32_t g_bootCount = 0;  /* .data section — not yet copied from flash */
static uint8_t g_buffer[256];       /* .bss section — not yet zeroed */

void Reset_Handler(void)
{
    g_bootCount++;              /* Reading garbage — .data not copied yet */
    g_buffer[0] = 0xAA;        /* May work by accident, but .bss state is undefined */
    main();
}
```

**Correct (proper startup assembly → C runtime init → main):**

```c
/* startup.s or startup.c — minimal reset handler */
extern uint32_t _estack;
extern uint32_t _sidata, _sdata, _edata;
extern uint32_t _sbss, _ebss;
extern void main(void);

__attribute__((naked, noreturn))
void Reset_Handler(void)
{
    /* 1. Stack pointer is loaded from vector table entry 0 by hardware,
          but set explicitly for safety */
    __asm volatile("ldr sp, =_estack");

    /* 2. Relocate vector table if not at default address */
    SCB->VTOR = (uint32_t)&g_vectorTable;

    /* 3. Copy .data from flash to RAM */
    uint32_t *src = &_sidata;
    uint32_t *dst = &_sdata;
    while (dst < &_edata)
    {
        *dst++ = *src++;
    }

    /* 4. Zero .bss */
    dst = &_sbss;
    while (dst < &_ebss)
    {
        *dst++ = 0U;
    }

    /* 5. Call main — global/static variables are now safe to use */
    main();

    /* 6. Should never return */
    for (;;) {}
}
```

```c
/* main.c — ordered initialization after C runtime is ready */
int main(void)
{
    /* Phase 1: Clock tree — must be first, all peripherals depend on it */
    SystemClock_Config();

    /* Phase 2: Critical peripherals (watchdog, GPIO for safe states) */
    WDG_Init();
    GPIO_Init();

    /* Phase 3: Communication peripherals */
    CAN_Init();
    SPI_Init();

    /* Phase 4: Application init */
    App_Init();

    /* Phase 5: Start scheduler (if using RTOS) — never returns */
    RTOS_StartScheduler();

    return 0;  /* Never reached */
}
```

**C++ additional consideration — static constructors:**

```cpp
// startup.cpp — call C++ static constructors before main
extern "C" {
    typedef void (*InitFunc)(void);
    extern InitFunc __init_array_start[];
    extern InitFunc __init_array_end[];
}

static void call_static_constructors(void)
{
    for (InitFunc *func = __init_array_start; func < __init_array_end; ++func)
    {
        (*func)();
    }
}

// Called between BSS/data init and main()
// Reset_Handler: ... bss_clear → data_copy → call_static_constructors() → main()
```

The linker script must define `_estack`, `_sidata`, `_sdata`, `_edata`, `_sbss`, `_ebss` symbols matching the memory layout. Verify with `arm-none-eabi-objdump -h` that sections are placed correctly.

Reference: ARM Cortex-M Architecture Reference Manual — Reset behavior; ISO 26262 Part 6 Table 1 (Initialization of variables)
