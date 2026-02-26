---
title: "ara::diag — Diagnostic Service Implementation and DTC Management"
impact: MEDIUM
impactDescription: Incorrect ara::diag usage causes failed workshop diagnostics, non-compliant UDS responses, or missed fault events
tags: autosar, adaptive, ara-diag, diagnostics, uds, dtc, generic-uds-service, diagnostic-conversation
---

## ara::diag — Diagnostic Service Implementation and DTC Management

ara::diag (AUTOSAR Adaptive R24-10) provides the diagnostic runtime for Adaptive Applications. It implements UDS (ISO 14229) services via C++ interfaces, replacing the Classic Dcm/Dem pattern with an object-oriented API. Applications implement diagnostic services as classes and register them with the Diagnostic Manager.

### GenericUDSService — Custom Diagnostic Services

**Incorrect (implementing UDS handling in raw socket code):**

```cpp
void HandleDoIPRequest(const std::vector<uint8_t>& data)
{
    if (data[0] == 0x31)  // RoutineControl
    {
        // Manual UDS parsing — no session check, no security, no NRC
        ExecuteRoutine(data);
        SendResponse(positiveResponse);
    }
}
```

**Correct (implement GenericUDSService for custom services):**

```cpp
#include <ara/diag/generic_uds_service.h>

class SelfTestService : public ara::diag::GenericUDSService
{
public:
    SelfTestService(const ara::core::InstanceSpecifier& specifier)
        : ara::diag::GenericUDSService(specifier)
    {
        Offer();
    }

    ~SelfTestService() override
    {
        StopOffer();
    }

    ara::core::Future<OperationOutput> HandleMessage(
        const ara::diag::MetaInfo& metaInfo,
        ara::diag::CancellationHandler cancellationHandler,
        const std::vector<uint8_t>& requestData) override
    {
        ara::core::Promise<OperationOutput> promise;

        // requestData[0] = SID (0x31 RoutineControl)
        // requestData[1] = SubFunction
        // requestData[2..3] = RoutineId

        if (requestData.size() < 4)
        {
            OperationOutput nrc;
            nrc.responseData = {0x7F, requestData[0], 0x13};  // incorrectMessageLength
            promise.set_value(nrc);
            return promise.get_future();
        }

        // Register cancellation callback for long-running operations
        cancellationHandler.SetIsCancelledFunction([this]() {
            return cancelRequested_.load();
        });

        // Execute self-test
        auto testResult = RunSelfTest();

        OperationOutput output;
        output.responseData = {
            static_cast<uint8_t>(requestData[0] + 0x40),  // Positive response
            requestData[1],
            requestData[2], requestData[3],
            testResult.status,
            testResult.errorCode
        };
        promise.set_value(output);
        return promise.get_future();
    }

private:
    std::atomic<bool> cancelRequested_{false};
};
```

### DiagnosticConversation — Session and Security

```cpp
#include <ara/diag/diagnostic_conversation.h>

void CheckDiagnosticContext(const ara::diag::MetaInfo& metaInfo)
{
    auto conversation = ara::diag::DiagnosticConversation::GetConversation(
        metaInfo);

    // Query session level
    auto sessionResult = conversation.GetDiagnosticSession();
    if (sessionResult.HasValue())
    {
        auto session = sessionResult.Value();
        if (session == ara::diag::SessionType::kDefaultSession)
        {
            // Limited diagnostic access in default session
        }
    }

    // Query security level
    auto secResult = conversation.GetDiagnosticSecurityLevel();
    if (secResult.HasValue())
    {
        auto level = secResult.Value();
        // Enforce access control based on security level
    }

    // Activity status — for tester-present monitoring
    auto activityResult = conversation.GetActivityStatus();
}
```

### DTC Management via ara::diag

```cpp
#include <ara/diag/monitor.h>

// Fault monitoring — replaces Dem_ReportErrorStatus from Classic
class RadarFaultMonitor
{
public:
    RadarFaultMonitor(const ara::core::InstanceSpecifier& specifier)
        : monitor_(specifier,
                   [this](ara::diag::Monitor::InitMonitorAction action) {
                       OnInitMonitor(action);
                   })
    {
    }

    void CheckRadarHealth()
    {
        if (IsRadarFaulty())
        {
            monitor_.ReportMonitorAction(
                ara::diag::MonitorAction::kFailed);
            // DM handles debouncing, DTC status update, snapshot capture
        }
        else
        {
            monitor_.ReportMonitorAction(
                ara::diag::MonitorAction::kPassed);
        }
    }

private:
    void OnInitMonitor(ara::diag::Monitor::InitMonitorAction action)
    {
        switch (action)
        {
            case ara::diag::Monitor::InitMonitorAction::kReInit:
                ResetMonitoringState();
                break;
            case ara::diag::Monitor::InitMonitorAction::kClear:
                ClearFaultCounters();
                break;
        }
    }

    ara::diag::Monitor monitor_;
};
```

### Data Identifier (DID) Service

```cpp
#include <ara/diag/generic_data_identifier.h>

class VehicleSpeedDID : public ara::diag::GenericDataIdentifier
{
public:
    VehicleSpeedDID(const ara::core::InstanceSpecifier& spec)
        : ara::diag::GenericDataIdentifier(spec)
    {
        Offer();
    }

    ara::core::Future<OperationOutput> Read(
        const ara::diag::MetaInfo& metaInfo,
        ara::diag::CancellationHandler cancellationHandler) override
    {
        ara::core::Promise<OperationOutput> promise;

        uint16_t speed = GetCurrentVehicleSpeed();
        OperationOutput output;
        output.responseData = {
            static_cast<uint8_t>(speed >> 8),
            static_cast<uint8_t>(speed & 0xFF)
        };
        promise.set_value(output);
        return promise.get_future();
    }
};
```

Reference: AUTOSAR Adaptive R24-10 SWS_Diagnostics (ara::diag), ISO 14229 (UDS)
