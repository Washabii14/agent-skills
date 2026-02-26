---
title: Memory Pool Pattern for Dynamic-Like Allocation
impact: HIGH
impactDescription: deterministic dynamic allocation
tags: memory, pool, allocation, pattern, determinism, fixed-size
---

## Memory Pool Pattern for Dynamic-Like Allocation

When you need allocation/deallocation flexibility, use a statically-backed memory pool with fixed-size blocks. This provides dynamic-like behavior with deterministic timing and no fragmentation.

**Correct (memory pool implementation):**

```c
#define POOL_BLOCK_SIZE  (64U)
#define POOL_BLOCK_COUNT (32U)

typedef struct
{
    uint8_t data[POOL_BLOCK_SIZE];
    boolean inUse;
} PoolBlock_t;

static PoolBlock_t g_memPool[POOL_BLOCK_COUNT];

void *MemPool_Alloc(void)
{
    uint16_t idx;
    for (idx = 0U; idx < POOL_BLOCK_COUNT; idx++)
    {
        if (g_memPool[idx].inUse == FALSE)
        {
            g_memPool[idx].inUse = TRUE;
            return (void *)g_memPool[idx].data;
        }
    }
    return NULL;
}

void MemPool_Free(void *ptr)
{
    uint16_t idx;
    for (idx = 0U; idx < POOL_BLOCK_COUNT; idx++)
    {
        if ((void *)g_memPool[idx].data == ptr)
        {
            g_memPool[idx].inUse = FALSE;
            return;
        }
    }
}
```

This pattern is especially useful for message buffers, CAN frame queues, and diagnostic session objects where allocation/deallocation is needed but heap is prohibited.

Reference: MISRA C:2012 Rule 21.3
