---
title: Network Management State Machine
impact: MEDIUM
impactDescription: ensures NM compliance
tags: comm, network-management, nm, autosar, bus-sleep, state-machine
---

## Network Management State Machine

Follow AUTOSAR NM state machine transitions correctly: Bus-Sleep <-> Prepare Bus-Sleep <-> Network Mode (with Repeat Message and Normal Operation substates). Incorrect transitions can cause ECUs to stay awake unnecessarily (draining battery) or go to sleep prematurely (losing communication).

**Incorrect (simplified NM without proper states):**

```c
void NM_HandleWakeup(void)
{
    g_nmState = NM_AWAKE;
    CAN_Enable();
}
```

**Correct (AUTOSAR-compliant NM state machine):**

```c
typedef enum
{
    NM_STATE_BUS_SLEEP,
    NM_STATE_PREPARE_BUS_SLEEP,
    NM_STATE_REPEAT_MESSAGE,
    NM_STATE_NORMAL_OPERATION,
    NM_STATE_READY_SLEEP
} NmState_t;

void NM_MainFunction(void)
{
    switch (g_nmState)
    {
        case NM_STATE_BUS_SLEEP:
            if (NM_WakeupReceived() || NM_InternalWakeup())
            {
                NM_TransitionTo(NM_STATE_REPEAT_MESSAGE);
                NM_StartRepeatMessageTimer();
            }
            break;

        case NM_STATE_REPEAT_MESSAGE:
            NM_SendNmMessage();
            if (NM_RepeatMessageTimerExpired())
            {
                NM_TransitionTo(NM_STATE_NORMAL_OPERATION);
            }
            break;

        case NM_STATE_NORMAL_OPERATION:
            if (!NM_HasNetworkRequest())
            {
                NM_TransitionTo(NM_STATE_READY_SLEEP);
            }
            break;

        case NM_STATE_READY_SLEEP:
            if (NM_NmTimeoutExpired())
            {
                NM_TransitionTo(NM_STATE_PREPARE_BUS_SLEEP);
            }
            break;

        case NM_STATE_PREPARE_BUS_SLEEP:
            if (NM_WaitBusSleepTimerExpired())
            {
                NM_TransitionTo(NM_STATE_BUS_SLEEP);
            }
            break;

        default:
            NM_TransitionTo(NM_STATE_BUS_SLEEP);
            break;
    }
}
```

Reference: AUTOSAR SWS Network Management (NM); AUTOSAR SWS CAN Network Management (CanNm)
