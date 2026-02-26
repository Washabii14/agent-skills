---
title: Adaptive AUTOSAR Boot Sequence
impact: HIGH
impactDescription: Incorrect Adaptive Platform startup causes process launch failures, unmet dependencies between function groups, or stalled execution states
tags: boot, autosar, adaptive, execution-manager, function-group, machine-manifest, ara, process
---

## Adaptive AUTOSAR Boot Sequence

Adaptive AUTOSAR startup is orchestrated by the Execution Manager (EM): platform bootstrap → machine manifest parsing → function group state transitions → process startup → each process calls `ReportExecutionState(kRunning)`. Unlike Classic AUTOSAR, processes are OS-level (POSIX) and can start/stop independently.

**Incorrect (process assumes platform services are ready without checking, never reports state):**

```cpp
// WRONG: Application never reports execution state, no dependency checking
int main()
{
    // Immediately uses ara::com without waiting for Communication Management
    auto proxy = ara::com::FindService<RadarServiceProxy>();  // May fail — CM not ready
    proxy->Subscribe();

    while (true)
    {
        DoWork();
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    // Never reports kRunning — EM thinks process is stuck in startup
    return 0;
}
```

**Correct (proper Adaptive Platform startup with execution state reporting):**

```cpp
#include <ara/exec/execution_client.h>
#include <ara/log/logging.h>
#include <ara/com/runtime.h>

int main()
{
    // 1. Initialize logging first — needed for all subsequent diagnostics
    ara::log::InitLogging("RDAR", "Radar Application",
                          ara::log::LogLevel::kInfo,
                          ara::log::LogMode::kConsole);
    auto logger = ara::log::CreateLogger("MAIN", "Main context");

    // 2. Initialize ara::com runtime
    ara::com::Runtime::Initialize();

    // 3. Create execution client to communicate with Execution Manager
    ara::exec::ExecutionClient exec_client;

    // 4. Report kRunning — EM now knows this process has started successfully
    auto result = exec_client.ReportExecutionState(
        ara::exec::ExecutionState::kRunning);
    if (!result.HasValue())
    {
        logger.LogError() << "Failed to report kRunning: "
                          << result.Error().Message();
        return 1;
    }
    logger.LogInfo() << "Reported kRunning to Execution Manager";

    // 5. Now safe to find and use services (dependencies resolved by EM)
    auto find_result = ara::com::FindService<RadarServiceProxy>(
        ara::com::InstanceSpecifier("/radar/RadarService"));
    if (find_result.HasValue() && !find_result.Value().empty())
    {
        auto& handle = find_result.Value()[0];
        auto proxy = std::make_unique<RadarServiceProxy>(handle);
        proxy->Subscribe();
    }

    // 6. Application main loop
    while (!shutdown_requested)
    {
        DoWork();
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }

    // 7. Clean shutdown
    ara::com::Runtime::Deinitialize();
    return 0;
}
```

**Machine manifest and function group configuration (JSON/ARXML concept):**

```cpp
// Execution Manager processes the machine manifest at boot:
//
// Machine Manifest defines:
//   MachineFG (Machine Function Group):
//     States: Off, Startup, Running, Shutdown
//
//   ApplicationFG (Application Function Group):
//     States: Off, Running, Diagnostics
//     Dependencies: MachineFG must be in "Running"
//
// Process manifest for RadarApp:
//   FunctionGroup: ApplicationFG
//   StartupConfig:
//     State: Running           // Start when ApplicationFG enters "Running"
//     SchedulingPolicy: FIFO
//     SchedulingPriority: 50
//     Dependencies: [CommunicationManagement, PlatformHealthManagement]

// Function group state request from application
void RequestDiagnosticMode(ara::exec::ExecutionClient& exec_client)
{
    // Request function group state change through Execution Manager
    auto result = exec_client.SetState(
        ara::exec::FunctionGroup("ApplicationFG"),
        ara::exec::FunctionGroupState("Diagnostics"));

    if (!result.HasValue())
    {
        // EM may deny if dependencies are not met
        logger.LogError() << "State transition denied: "
                          << result.Error().Message();
    }
}
```

**Startup dependency ordering managed by EM:**

```cpp
// Platform startup order (managed by Execution Manager):
//
// 1. EM starts → reads machine manifest
// 2. MachineFG → "Startup"
//    - Start: LogAndTrace, PlatformHealthManagement
// 3. MachineFG → "Running"
//    - Start: CommunicationManagement, DiagnosticManagement,
//             UpdateAndConfigManagement, TimeSync
// 4. ApplicationFG → "Running"  (depends on MachineFG = "Running")
//    - Start: RadarApp, CameraApp, FusionApp
//    - Each calls ReportExecutionState(kRunning)
// 5. EM verifies all processes in "Running" state reported kRunning
//    - Timeout → EM triggers recovery (restart process or escalate)
```

Each process must call `ReportExecutionState(kRunning)` within the EM-configured timeout. Failure to report triggers recovery actions defined in the machine manifest (process restart, function group state change, or platform reset).

Reference: AUTOSAR Adaptive Platform SWS_ExecutionManagement; AUTOSAR AP Manifest Specification; ISO 26262 Part 6 (Startup/shutdown for ASIL applications)
