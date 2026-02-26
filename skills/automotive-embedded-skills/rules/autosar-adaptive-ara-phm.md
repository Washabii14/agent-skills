---
title: "ara::phm — Platform Health Management and Supervision Modes"
impact: HIGH
impactDescription: Missing or incorrect health monitoring causes undetected process hangs, missed deadlines, or failure to reach safe state
tags: autosar, adaptive, ara-phm, health-management, alive-supervision, deadline-supervision, logical-supervision, watchdog
---

## ara::phm — Platform Health Management and Supervision Modes

ara::phm (AUTOSAR Adaptive R24-10) provides health monitoring for Adaptive Applications through three supervision mechanisms: alive supervision (periodic heartbeat), deadline supervision (execution time bounds), and logical supervision (correct execution order). PHM triggers recovery actions when supervision conditions are violated.

### Alive Supervision

**Incorrect (no periodic health reporting — hang goes undetected):**

```cpp
void MainLoop()
{
    while (running)
    {
        ProcessCycle();  // If this blocks forever, nobody notices
    }
}
```

**Correct (report alive checkpoint periodically):**

```cpp
#include <ara/phm/supervised_entity.h>

class RadarApp
{
public:
    RadarApp(const ara::core::InstanceSpecifier& seSpec)
        : supervisedEntity_(seSpec)
    {
    }

    void MainLoop()
    {
        while (running_)
        {
            // Report alive checkpoint — PHM expects this within configured window
            supervisedEntity_.ReportCheckpoint(kAliveCheckpointId);

            ProcessRadarCycle();
        }
    }

private:
    // Alive supervision configuration (Machine Manifest):
    //   ExpectedAliveIndications: 1
    //   MinMargin: 0         — no missed reports tolerated
    //   MaxMargin: 0         — no extra reports tolerated
    //   AliveReferenceCycle: 100ms  — expected reporting period
    //
    // If ReportCheckpoint not called within 100ms ± margins,
    // PHM reports supervision violation to EM

    static constexpr uint32_t kAliveCheckpointId = 0;
    ara::phm::SupervisedEntity supervisedEntity_;
    bool running_ = true;
};
```

### Deadline Supervision

**Incorrect (no time bounds on critical computation):**

```cpp
void ComputeTrajectory(const SensorData& data)
{
    // Complex computation — may exceed real-time deadline
    auto trajectory = PlanPath(data);  // No monitoring of execution time
    ActuatorControl(trajectory);
}
```

**Correct (checkpoint pair for deadline monitoring):**

```cpp
void ComputeTrajectory(const SensorData& data,
                       ara::phm::SupervisedEntity& se)
{
    // Start checkpoint — begins deadline timer
    se.ReportCheckpoint(kDeadlineStartCheckpoint);

    auto trajectory = PlanPath(data);
    ActuatorControl(trajectory);

    // End checkpoint — stops deadline timer
    se.ReportCheckpoint(kDeadlineEndCheckpoint);

    // Deadline supervision configuration (Machine Manifest):
    //   DeadlineSupervision:
    //     SourceCheckpoint: kDeadlineStartCheckpoint
    //     TargetCheckpoint: kDeadlineEndCheckpoint
    //     MinExecutionTime: 1ms     — detect too-fast execution (skipped logic)
    //     MaxExecutionTime: 15ms    — detect overruns
    //
    // Violation if execution time < 1ms or > 15ms
}

static constexpr uint32_t kDeadlineStartCheckpoint = 1;
static constexpr uint32_t kDeadlineEndCheckpoint = 2;
```

### Logical Supervision

**Incorrect (no verification that processing steps execute in correct order):**

```cpp
void ProcessSafetyCritical()
{
    // Steps may execute out of order due to bugs — no detection
    ValidateInput();
    ComputeOutput();
    VerifyOutput();
    ApplyOutput();
}
```

**Correct (logical supervision graph enforces execution order):**

```cpp
void ProcessSafetyCritical(ara::phm::SupervisedEntity& se)
{
    // Checkpoint graph defined in Machine Manifest:
    //   LogicalSupervision:
    //     Graph: CP_Start -> CP_Validate -> CP_Compute -> CP_Verify -> CP_Apply
    //     InitialCheckpoint: CP_Start
    //     FinalCheckpoint: CP_Apply
    //
    // Any deviation from the defined graph triggers a violation

    se.ReportCheckpoint(kCpStart);

    if (!ValidateInput())
    {
        se.ReportCheckpoint(kCpError);  // Error path also defined in graph
        return;
    }
    se.ReportCheckpoint(kCpValidate);

    ComputeOutput();
    se.ReportCheckpoint(kCpCompute);

    if (!VerifyOutput())
    {
        se.ReportCheckpoint(kCpError);
        return;
    }
    se.ReportCheckpoint(kCpVerify);

    ApplyOutput();
    se.ReportCheckpoint(kCpApply);  // Final checkpoint — supervision pass
}

static constexpr uint32_t kCpStart    = 10;
static constexpr uint32_t kCpValidate = 11;
static constexpr uint32_t kCpCompute  = 12;
static constexpr uint32_t kCpVerify   = 13;
static constexpr uint32_t kCpApply    = 14;
static constexpr uint32_t kCpError    = 15;
```

### Supervision Mode Transitions and Recovery

```cpp
// PHM supervision modes (configured in Machine Manifest):
//
// SupervisionMode:
//   Mode: Normal
//     AliveSupervision: enabled (100ms cycle)
//     DeadlineSupervision: enabled (15ms max)
//     LogicalSupervision: enabled
//
//   Mode: Degraded
//     AliveSupervision: enabled (200ms cycle — relaxed)
//     DeadlineSupervision: disabled
//     LogicalSupervision: enabled
//
//   Mode: SafeState
//     AliveSupervision: enabled (500ms cycle)
//     DeadlineSupervision: disabled
//     LogicalSupervision: disabled
//
// Recovery actions on supervision failure:
//   FailedSupervisionCycles: 3    — tolerate 3 consecutive failures
//   RecoveryAction:
//     1. Log error via ara::log
//     2. Notify EM to restart process
//     3. If restart fails: transition function group to safe state
//     4. If safe state fails: machine reset

// Application-side health status query:
void CheckHealth(ara::phm::SupervisedEntity& se)
{
    auto status = se.GetLocalSupervisionStatus();
    if (status.HasValue())
    {
        switch (status.Value())
        {
            case ara::phm::SupervisionStatus::kOk:
                break;
            case ara::phm::SupervisionStatus::kFailed:
                EnterDegradedMode();
                break;
            case ara::phm::SupervisionStatus::kExpired:
                RequestSafeState();
                break;
        }
    }
}
```

Reference: AUTOSAR Adaptive R24-10 SWS_PlatformHealthManagement (ara::phm), ISO 26262 Part 6
