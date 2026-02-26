---
title: No Dynamic Memory Allocation
impact: CRITICAL
impactDescription: eliminates heap fragmentation, deterministic memory
tags: misra, dynamic-memory, heap, malloc, free, determinism, safety
---

## No Dynamic Memory Allocation

Dynamic memory allocation shall not be used in production embedded automotive code. The heap introduces non-determinism and fragmentation that can lead to unpredictable failures.

**Incorrect (dynamic allocation):**

```c
void InitModule(uint8_t count)
{
    Item_t *items = (Item_t *)malloc(count * sizeof(Item_t));
    if (items == NULL)
    {
        /* Too late — system may already be compromised */
        return;
    }
    /* ... */
    free(items);
}
```

**Correct (static allocation with maximum bound):**

```c
#define MAX_ITEMS (16U)

static Item_t g_items[MAX_ITEMS];
static uint8_t g_itemCount = 0U;

Std_ReturnType InitModule(uint8_t count)
{
    if (count > MAX_ITEMS)
    {
        return E_NOT_OK;
    }
    g_itemCount = count;
    (void)memset(g_items, 0, sizeof(g_items));
    return E_OK;
}
```

Use static allocation, stack allocation, or memory pool patterns instead of `malloc`/`free`.

Reference: MISRA C:2012 Rule 21.3 — The memory allocation and deallocation functions of `<stdlib.h>` shall not be used.
