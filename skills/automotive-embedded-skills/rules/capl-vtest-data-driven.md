---
title: Data-Driven Testing in vTESTstudio
impact: MEDIUM
impactDescription: Parameterized test cases with external data sources eliminate test code duplication and enable systematic boundary value testing across hundreds of signal combinations
tags: capl, vtestudio, data-driven, parameterized, csv, excel, test-parameters, iteration
---

## Data-Driven Testing in vTESTstudio

Data-driven testing separates test logic from test data by binding test cases to external parameter files (CSV, Excel). Each row in the data file becomes a test iteration, enabling exhaustive boundary value and equivalence class testing without duplicating CAPL code.

**Incorrect (hardcoded test values, duplicated test cases):**

```capl
/* Duplicated test logic for each speed threshold — unmaintainable */
testcase TC_SpeedWarning_60kph()
{
    setSignal(VehicleSpeed, 60);
    testWaitForTimeout(500);
    if (getSignal(SpeedWarningLamp) != 0)
        testStepFail("Warning should be OFF at 60 kph");
}

testcase TC_SpeedWarning_119kph()
{
    setSignal(VehicleSpeed, 119);
    testWaitForTimeout(500);
    if (getSignal(SpeedWarningLamp) != 0)
        testStepFail("Warning should be OFF at 119 kph");
}

testcase TC_SpeedWarning_120kph()
{
    setSignal(VehicleSpeed, 120);
    testWaitForTimeout(500);
    if (getSignal(SpeedWarningLamp) != 1)
        testStepFail("Warning should be ON at 120 kph");
}

/* ... 20 more nearly identical test cases ... */
```

**Correct (parameterized test with external data):**

```capl
/*@@testgroup: TG_SpeedWarning_Parameterized */
/*@@description: Speed warning lamp activation thresholds */
/*@@parameter_file: test_data/speed_warning_params.csv */

/* CSV file: test_data/speed_warning_params.csv
 * ------------------------------------------
 * TestID,Speed_kph,Expected_Warning,Description
 * SPD-001,0,0,Standstill
 * SPD-002,30,0,City speed
 * SPD-003,60,0,Suburban speed
 * SPD-004,119,0,Just below threshold
 * SPD-005,120,1,At threshold - warning ON
 * SPD-006,121,1,Just above threshold
 * SPD-007,200,1,High speed
 * SPD-008,255,1,Maximum signal value
 */

@setup
void Setup_SpeedWarningTests()
{
    setSignal(VehicleSpeed, 0);
    setSignal(IgnitionSwitch, 1);
    testWaitForTimeout(200);
}

@teardown
void Teardown_SpeedWarningTests()
{
    setSignal(VehicleSpeed, 0);
    testWaitForTimeout(100);
}

@testcase
void TC_SpeedWarning_Threshold(
    char TestID[],
    int Speed_kph,
    int Expected_Warning,
    char Description[])
{
    testCaseTitle(TestID, "Speed warning at %d kph", Speed_kph);
    testCaseDescription(Description);

    testStep("Stimulate", "Setting vehicle speed to %d kph", Speed_kph);
    setSignal(VehicleSpeed, Speed_kph);

    testStep("Wait", "Allowing ECU processing time");
    testWaitForTimeout(500);

    testStep("Verify", "Checking warning lamp state");
    int actual_warning = getSignal(SpeedWarningLamp);
    if (actual_warning == Expected_Warning)
    {
        testStepPass("Result",
            "%s: Speed=%d kph, Warning=%d (expected %d)",
            TestID, Speed_kph, actual_warning, Expected_Warning);
    }
    else
    {
        testStepFail("Result",
            "%s: Speed=%d kph, Warning=%d but expected %d",
            TestID, Speed_kph, actual_warning, Expected_Warning);
    }
}
```

**Excel-bound parameterized test (complex multi-signal scenarios):**

```capl
/*@@testgroup: TG_ClimateControl_Parameterized */
/*@@parameter_file: test_data/climate_control.xlsx */
/*@@parameter_sheet: NominalScenarios */

/* Excel columns map directly to test case parameters.
 * Each row is one test iteration with its own verdict. */

@testcase
void TC_ClimateControl_AutoMode(
    char TestID[],
    float Ambient_Temp_C,
    float Cabin_Temp_C,
    float Target_Temp_C,
    int Expected_Compressor,
    int Expected_FanSpeed,
    char Description[])
{
    testCaseTitle(TestID, Description);

    /* Stimulate environment sensors */
    testStep("Setup", "Ambient=%.1f°C, Cabin=%.1f°C, Target=%.1f°C",
             Ambient_Temp_C, Cabin_Temp_C, Target_Temp_C);
    setSignal(AmbientTemp, Ambient_Temp_C);
    setSignal(CabinTemp, Cabin_Temp_C);
    setSignal(TargetTemp, Target_Temp_C);
    setSignal(ClimateMode, MODE_AUTO);

    testWaitForTimeout(2000);

    /* Verify actuator outputs */
    testStep("Verify", "Checking compressor and fan speed");
    int compressor = getSignal(AC_Compressor);
    int fan = getSignal(BlowerFanSpeed);

    if (compressor == Expected_Compressor)
    {
        testStepPass("Compressor", "State=%d (expected %d)",
                     compressor, Expected_Compressor);
    }
    else
    {
        testStepFail("Compressor", "State=%d (expected %d)",
                     compressor, Expected_Compressor);
    }

    if (fan == Expected_FanSpeed)
    {
        testStepPass("FanSpeed", "Level=%d (expected %d)",
                     fan, Expected_FanSpeed);
    }
    else
    {
        testStepFail("FanSpeed", "Level=%d (expected %d)",
                     fan, Expected_FanSpeed);
    }
}
```

**Result reporting per parameter combination:**

```capl
/* vTESTstudio automatically generates per-iteration verdicts:
 *
 * Test Report:
 *   TG_SpeedWarning_Parameterized
 *     TC_SpeedWarning_Threshold [SPD-001] — PASS (0 kph, warning OFF)
 *     TC_SpeedWarning_Threshold [SPD-002] — PASS (30 kph, warning OFF)
 *     TC_SpeedWarning_Threshold [SPD-003] — PASS (60 kph, warning OFF)
 *     TC_SpeedWarning_Threshold [SPD-004] — PASS (119 kph, warning OFF)
 *     TC_SpeedWarning_Threshold [SPD-005] — FAIL (120 kph, expected ON)
 *     ...
 *
 * Each iteration has its own verdict, timestamp, and signal trace.
 * Failed iterations can be re-run individually.
 */
```

Reference: Vector vTESTstudio User Manual — Test Parameter Tables, CANoe Help — Data-Driven Test Execution
