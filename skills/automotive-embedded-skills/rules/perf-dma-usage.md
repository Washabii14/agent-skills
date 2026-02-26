---
title: DMA for Bulk Data Transfers
impact: MEDIUM
impactDescription: Offloads CPU by using DMA for large data transfers
tags: perf, dma, data-transfer, adc, spi, uart, optimization
---

## DMA for Bulk Data Transfers

Use DMA for ADC sampling, SPI/UART transfers, and memory-to-memory copies to offload the CPU. DMA transfers proceed in parallel with CPU execution, freeing cycle budget for application logic.

**Incorrect (CPU-bound bulk transfer):**

```c
void ReadAdcChannels(uint16_t *results, uint8_t count)
{
    for (uint8_t i = 0U; i < count; i++)
    {
        results[i] = ADC_ReadBlocking(i);
    }
}
```

**Correct (DMA-driven transfer):**

```c
void ReadAdcChannels_Dma(uint16_t *results, uint8_t count)
{
    DMA_ConfigTransfer(DMA_CH_ADC,
                       (uint32_t)&ADC->DATA,
                       (uint32_t)results,
                       count,
                       DMA_PERIPH_TO_MEM | DMA_WIDTH_16BIT);
    DMA_EnableTransferCompleteIrq(DMA_CH_ADC);
    DMA_Start(DMA_CH_ADC);
    ADC_StartSequence(count);
}

void DMA_ADC_CompleteISR(void)
{
    DMA_ClearFlag(DMA_CH_ADC);
    OS_SetEvent(TASK_SENSOR, EVT_ADC_COMPLETE);
}
```

Ensure DMA buffers are aligned to cache line boundaries and use cache maintenance operations (clean/invalidate) when CPU and DMA share memory on cached architectures.
