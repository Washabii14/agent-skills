---
title: "ara::com — Service-Oriented Communication: Proxy/Skeleton, Events, Methods, Fields"
impact: HIGH
impactDescription: Incorrect ara::com usage causes service discovery failures, event data loss, or undefined behavior in service-oriented architectures
tags: autosar, adaptive, ara-com, proxy, skeleton, service-discovery, event, method, field, someip
---

## ara::com — Service-Oriented Communication

ara::com (AUTOSAR Adaptive R24-10) is the communication middleware for service-oriented architectures. It implements the proxy/skeleton pattern: service providers implement skeletons, service consumers use proxies. The binding layer (e.g., SOME/IP, DDS) is abstracted away. All examples use C++14 per Adaptive Platform specification.

### Service Discovery — FindService / OfferService

**Incorrect (hardcoding service endpoints instead of using discovery):**

```cpp
// Hardcoded IP:port — breaks when service moves, no failover
auto proxy = RadarService::Proxy("192.168.1.10", 30509);
```

**Correct (dynamic service discovery via ara::com):**

```cpp
#include "radar_service/radar_service_proxy.h"

using namespace ara::com;

void StartRadarClient()
{
    auto handles = RadarService::Proxy::FindService(
        InstanceIdentifier("RadarFrontLeft"));

    if (!handles.empty())
    {
        RadarService::Proxy proxy(handles[0]);
        // Service bound — ready for events/methods/fields
    }

    // Or asynchronous discovery with handler:
    auto findHandle = RadarService::Proxy::StartFindService(
        [](ServiceHandleContainer<RadarService::Proxy::HandleType> handles,
           FindServiceHandle handle)
        {
            for (auto& h : handles)
            {
                auto proxy = std::make_unique<RadarService::Proxy>(h);
                // Process new service instance
            }
        },
        InstanceIdentifier("RadarFrontLeft"));

    // Stop discovery when no longer needed
    // RadarService::Proxy::StopFindService(findHandle);
}
```

### Skeleton — Service Provider

**Incorrect (not offering the service after construction):**

```cpp
class RadarServiceImpl : public RadarService::Skeleton
{
public:
    RadarServiceImpl(InstanceIdentifier id)
        : RadarService::Skeleton(id, MethodCallProcessingMode::kEvent)
    {
        // Forgot to call OfferService() — clients can never find this
    }
};
```

**Correct (offer service, implement methods, fire events):**

```cpp
class RadarServiceImpl : public RadarService::Skeleton
{
public:
    RadarServiceImpl(InstanceIdentifier id)
        : RadarService::Skeleton(id, MethodCallProcessingMode::kEvent)
    {
        OfferService();
    }

    ~RadarServiceImpl()
    {
        StopOfferService();
    }

    ara::core::Future<CalibrateOutput> Calibrate(
        const CalibrationConfig& config) override
    {
        ara::core::Promise<CalibrateOutput> promise;
        CalibrateOutput result;

        if (PerformCalibration(config, result))
        {
            promise.set_value(result);
        }
        else
        {
            promise.SetError(
                ara::com::MakeErrorCode(ComErrc::kServiceNotAvailable));
        }
        return promise.get_future();
    }

    void SendRadarObjects(const RadarObjectList& objects)
    {
        radarObjects.Send(objects);
    }
};
```

### Event Subscription (Consumer Side)

**Incorrect (not checking subscription state before accessing data):**

```cpp
void ProcessEvents(RadarService::Proxy& proxy)
{
    // No subscription — GetNewSamples() will fail
    proxy.radarObjects.GetNewSamples([](auto sample) {
        ProcessObject(*sample);
    });
}
```

**Correct (subscribe, set receive handler, process samples):**

```cpp
void SetupEventSubscription(RadarService::Proxy& proxy)
{
    proxy.radarObjects.Subscribe(10);  // Cache size = 10 samples

    proxy.radarObjects.SetReceiveHandler([&proxy]() {
        proxy.radarObjects.GetNewSamples(
            [](ara::com::SamplePtr<const RadarObjectList> sample) {
                for (const auto& obj : sample->objects)
                {
                    ProcessRadarObject(obj);
                }
            },
            5);  // Max 5 samples per call
    });

    // Check subscription state
    if (proxy.radarObjects.GetSubscriptionState()
        != ara::com::SubscriptionState::kSubscribed)
    {
        // Handle subscription failure
    }
}
```

### Method Calls

```cpp
void CallRadarMethod(RadarService::Proxy& proxy)
{
    CalibrationConfig config{/* ... */};
    auto future = proxy.Calibrate(config);

    // Non-blocking: register callback
    future.then([](ara::core::Future<CalibrateOutput> f) {
        auto result = f.GetResult();
        if (result.HasValue())
        {
            ApplyCalibration(result.Value());
        }
        else
        {
            HandleError(result.Error());
        }
    });
}
```

### Fields (Getter / Setter / Notifier)

```cpp
// Skeleton side: field with getter, setter, and notifier
void RadarServiceImpl::InitFields()
{
    // Register field handlers
    radarMode.RegisterGetHandler([]() -> ara::core::Future<RadarMode> {
        ara::core::Promise<RadarMode> p;
        p.set_value(currentMode);
        return p.get_future();
    });

    radarMode.RegisterSetHandler(
        [](const RadarMode& mode) -> ara::core::Future<RadarMode> {
            ara::core::Promise<RadarMode> p;
            if (IsValidMode(mode))
            {
                currentMode = mode;
                p.set_value(mode);
            }
            else
            {
                p.SetError(MakeErrorCode(ComErrc::kInvalidArgument));
            }
            return p.get_future();
        });
}

// Proxy side: subscribe to field notifications
void MonitorRadarMode(RadarService::Proxy& proxy)
{
    proxy.radarMode.Subscribe(1);
    proxy.radarMode.SetReceiveHandler([&proxy]() {
        proxy.radarMode.GetNewSamples([](auto sample) {
            OnRadarModeChanged(*sample);
        });
    });

    // Explicit get
    auto future = proxy.radarMode.Get();
}
```

Reference: AUTOSAR Adaptive R24-10 SWS_CommunicationManagement (ara::com), SOME/IP Protocol Specification
