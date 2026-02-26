---
title: SOME/IP Service Discovery
impact: HIGH
impactDescription: dynamic service availability management
tags: comm, someip-sd, service-discovery, ethernet, multicast, autosar-adaptive
---

## SOME/IP Service Discovery

SOME/IP-SD enables dynamic service offer/find/subscribe over UDP multicast (default 224.224.224.245:30490). Critical for Adaptive AUTOSAR service-oriented architecture. Handle both OfferService (service becomes available) and StopOfferService (service goes down) correctly.

**Incorrect (no service lifecycle tracking):**

```c
void OnServiceFound(uint16_t serviceId)
{
    ConnectToService(serviceId);  /* No availability tracking */
}
```

**Correct (full service lifecycle management):**

```c
#define SOMEIP_SD_PORT           (30490U)
#define SOMEIP_SD_MULTICAST_ADDR "224.224.224.245"

typedef struct
{
    uint16_t serviceId;
    uint16_t instanceId;
    uint8_t  majorVersion;
    uint32_t minorVersion;
    uint32_t ttlSeconds;
    boolean  isAvailable;
    uint32_t lastOfferTimestamp;
} SomeIpSd_ServiceEntry_t;

void SomeIpSd_HandleOfferService(const SomeIpSd_ServiceEntry_t *entry)
{
    SomeIpSd_ServiceEntry_t *local = SomeIpSd_FindService(
        entry->serviceId, entry->instanceId);

    if (local == NULL)
    {
        SomeIpSd_RegisterService(entry);
    }
    else
    {
        local->isAvailable = TRUE;
        local->ttlSeconds  = entry->ttlSeconds;
        local->lastOfferTimestamp = Timer_GetCurrentMs();
    }

    SomeIpSd_NotifyConsumers(entry->serviceId, entry->instanceId,
                              SERVICE_AVAILABLE);
}

void SomeIpSd_HandleStopOfferService(uint16_t serviceId,
                                       uint16_t instanceId)
{
    SomeIpSd_ServiceEntry_t *local = SomeIpSd_FindService(
        serviceId, instanceId);

    if (local != NULL)
    {
        local->isAvailable = FALSE;
        SomeIpSd_NotifyConsumers(serviceId, instanceId,
                                  SERVICE_UNAVAILABLE);
    }
}
```

Key SD considerations:
- Monitor TTL expiration — if no re-offer before TTL expires, treat service as unavailable
- Use initial/repetition/main phases for offer timing
- Subscribe to event groups after service is offered
- Handle version mismatch (major version must match exactly)

Reference: AUTOSAR PRS_SOMEIPSD (SOME/IP Service Discovery Protocol Specification)
