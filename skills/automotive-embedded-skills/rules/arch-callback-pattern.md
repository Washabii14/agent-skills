---
title: Callback Patterns for Layer Decoupling
impact: MEDIUM
impactDescription: Callbacks decouple upper layers from lower layers, enabling independent development and testing
tags: arch, callback, function-pointer, decoupling, layered-architecture, dependency-inversion
---

## Callback Patterns for Layer Decoupling

Use function pointers / callbacks to decouple upper layers from lower layers. Lower layers should never include headers from upper layers — instead, they notify upward through registered callbacks.

**Incorrect (lower layer directly calls upper layer):**

```c
/* can_driver.c — directly calls application layer */
#include "app_engine_ctrl.h"  /* Tight coupling to application */

void CAN_RxHandler(const CanFrame_t *frame)
{
    EngineCtrl_ProcessCanFrame(frame);  /* Driver depends on application */
}
```

**Correct (callback registration decouples layers):**

```c
/* can_driver.h — defines callback type */
typedef void (*CanRxCallback_t)(const CanFrame_t *frame);
void CAN_RegisterRxCallback(uint32_t msgId, CanRxCallback_t callback);

/* can_driver.c — calls registered callback */
static CanRxCallback_t g_rxCallbacks[MAX_CALLBACKS];

void CAN_RxHandler(const CanFrame_t *frame)
{
    if (g_rxCallbacks[frame->id] != NULL)
    {
        g_rxCallbacks[frame->id](frame);
    }
}

/* app_engine_ctrl.c — registers itself */
void EngineCtrl_Init(void)
{
    CAN_RegisterRxCallback(MSG_ID_ENGINE_STATUS, OnEngineStatusRx);
}
```

The driver layer has no compile-time dependency on the application. This enables independent unit testing, reuse across projects, and follows the dependency inversion principle.

Reference: AUTOSAR Layered Architecture — BSW/RTE/SWC boundaries
