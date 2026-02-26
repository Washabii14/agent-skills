---
title: "ara::exec — Execution Manager, Process Lifecycle, and Function Groups"
impact: HIGH
impactDescription: Failure to report execution state or mishandling function groups causes process termination, failed startup, or undefined machine state
tags: autosar, adaptive, ara-exec, execution-manager, process-lifecycle, function-group, machine-state, deterministic-execution
---

## ara::exec — Execution Manager, Process Lifecycle, and Function Groups

ara::exec (AUTOSAR Adaptive R24-10) manages the lifecycle of Adaptive Applications. The Execution Manager (EM) starts, monitors, and terminates processes according to the Machine Manifest. Applications must report their execution state; failure to do so results in process termination.

### Process Lifecycle — ReportExecutionState

**Incorrect (application runs without reporting state to EM):**

```cpp
int main()
{
    InitializeApp();
    RunMainLoop();  // EM has no visibility — may terminate the process
    return 0;
}
```

**Correct (report kRunning, handle termination signals):**

```cpp
#include <ara/exec/execution_client.h>
#include <csignal>

volatile std::sig_atomic_t g_terminationRequested = 0;

void SignalHandler(int signal)
{
    g_terminationRequested = 1;
}

int main()
{
    std::signal(SIGTERM, SignalHandler);

    ara::exec::ExecutionClient execClient;

    // Report to EM that initialization is complete and app is running
    auto result = execClient.ReportExecutionState(
        ara::exec::ExecutionState::kRunning);

    if (!result.HasValue())
    {
        return 1;  // Cannot communicate with EM
    }

    InitializeApp();

    while (!g_terminationRequested)
    {
        RunCycle();
    }

    Cleanup();

    // kTerminating tells EM the process will exit gracefully
    execClient.ReportExecutionState(
        ara::exec::ExecutionState::kTerminating);

    return 0;
}
```

### Function Groups and Machine States

**Incorrect (starting services without checking function group state):**

```cpp
void StartAllServices()
{
    // Starting everything unconditionally — may violate machine state constraints
    StartRadarService();
    StartCameraService();
    StartParkingService();
}
```

**Correct (function group state drives which processes are active):**

```cpp
#include <ara/exec/function_group.h>
#include <ara/exec/function_group_state.h>

// Machine Manifest defines function groups and their states:
//
// FunctionGroup: FG_DrivingMode
//   States: Off, Standby, Driving, Parking
//
//   Driving -> processes: RadarApp, CameraApp, ADASApp
//   Parking -> processes: ParkingApp, CameraApp, UltrasonicApp
//   Standby -> processes: DiagApp (only)
//
// EM starts/stops processes automatically based on function group transitions

void RequestDrivingMode()
{
    ara::exec::FunctionGroup fg("FG_DrivingMode");
    ara::exec::FunctionGroupState drivingState(fg, "Driving");

    auto result = ara::exec::ExecutionClient::SetState(drivingState);

    if (!result.HasValue())
    {
        ara::log::LogError() << "Failed to set Driving state: "
                             << result.Error().Message();
    }
    // EM transitions: stops Parking-only processes, starts Driving processes
}

// Machine states (top-level function group):
//   Startup -> Verify -> Run -> Shutdown -> Off
//   EM manages these transitions based on Machine Manifest
```

### Deterministic Execution

```cpp
#include <ara/exec/deterministic_client.h>

// For ASIL-rated applications requiring deterministic timing
int main()
{
    ara::exec::DeterministicClient detClient;
    ara::exec::ExecutionClient execClient;

    execClient.ReportExecutionState(ara::exec::ExecutionState::kRunning);

    // EM provides deterministic activation — no application timer needed
    while (true)
    {
        auto activationResult = detClient.WaitForActivation();

        if (activationResult == ara::exec::ActivationReturnType::kRun)
        {
            // Cyclic execution — deterministic timing guaranteed by EM
            ReadSensors();
            ComputeControl();
            WriteActuators();
        }
        else if (activationResult ==
                 ara::exec::ActivationReturnType::kTerminate)
        {
            break;
        }
        else if (activationResult ==
                 ara::exec::ActivationReturnType::kRegisterServices)
        {
            // First activation: register ara::com services
            OfferServices();
        }
    }

    execClient.ReportExecutionState(ara::exec::ExecutionState::kTerminating);
    return 0;
}
```

### Process Manifest Configuration

```cpp
// process_manifest.json (simplified):
// {
//   "process": "RadarApp",
//   "functionGroup": "FG_DrivingMode",
//   "functionGroupState": "Driving",
//   "startupConfig": {
//     "schedulingPolicy": "FIFO",
//     "schedulingPriority": 80,
//     "numberOfRestartAttempts": 3,
//     "resourceGroup": "RG_HighPriority"
//   },
//   "executionDependencies": [
//     { "process": "PlatformServices", "state": "kRunning" }
//   ]
// }

// EM enforces:
//   - Process only runs in declared function group state
//   - Restart policy on unexpected termination
//   - Resource limits (CPU, memory) via cgroups
//   - Startup ordering via execution dependencies
```

Reference: AUTOSAR Adaptive R24-10 SWS_ExecutionManagement (ara::exec), AP_SWS_ExecutionManagement
