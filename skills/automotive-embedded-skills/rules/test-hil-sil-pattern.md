---
title: HIL/SIL Test Patterns
impact: HIGH
impactDescription: Validates real-time behavior on hardware and simulation
tags: test, hil, sil, real-time, verification, timing
---

## HIL/SIL Test Patterns

Structure Hardware-in-the-Loop (HIL) and Software-in-the-Loop (SIL) tests for timing and integration verification. HIL tests run on real ECU hardware with simulated plant models; SIL tests run compiled application code on a host with virtual ECU.

**Correct (SIL test pattern with timing verification):**

```c
void SilTest_ControlLoopTiming(void)
{
    VirtualEcu_Init();
    PlantModel_Init();

    /* Run 1000 cycles of the 10ms control loop */
    for (uint32_t cycle = 0U; cycle < 1000U; cycle++)
    {
        PlantModel_Step();
        VirtualEcu_RunTask_10ms();

        float wcet = VirtualEcu_GetLastTaskDuration_us();
        TEST_ASSERT_TRUE(wcet < 8000.0f);  /* Must complete within 8ms */

        float output = VirtualEcu_GetActuatorCommand();
        TEST_ASSERT_FLOAT_WITHIN(TOLERANCE, PlantModel_GetExpected(), output);
    }
}
```

**Correct (HIL test pattern with CAPL):**

```capl
testcase TC_HIL_BrakeResponse()
{
    testStep("Stimulus", "Apply brake pedal signal");
    setSignal(BrakePedalPosition, 80);

    testStep("Verify", "Check brake pressure response within 50ms");
    if (testWaitForSignalInRange(BrakePressure, 70, 90, 50) != 0)
    {
        testStepFail("Timing", "Brake response exceeded 50ms deadline");
        return;
    }
    testStepPass("Timing", "Brake response within deadline");
}
```

SIL tests catch logic defects early. HIL tests validate real-time behavior, electrical interfaces, and timing on target hardware.

Reference: ISO 26262 Part 6, Clause 10 — Software integration testing on target hardware
