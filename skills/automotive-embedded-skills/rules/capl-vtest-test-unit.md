---
title: vTESTstudio Test Unit and Fixture Structure
impact: HIGH
impactDescription: Proper test unit organization with @testcase, @setup, @teardown ensures repeatable and maintainable test execution in vTESTstudio projects
tags: capl, vtestudio, test-unit, test-group, fixture, setup, teardown, testcase, test-structure
---

## vTESTstudio Test Unit and Fixture Structure

vTESTstudio organizes tests into Test Units containing Test Groups with shared setup/teardown fixtures. Using `@testcase`, `@setup`, and `@teardown` decorators correctly ensures deterministic test execution and proper resource cleanup between tests.

**Incorrect (flat test structure with no fixtures):**

```capl
/* All tests in one block, no setup/teardown, duplicated initialization */

testcase TC_EngineStart()
{
    /* Manual setup repeated in every test */
    setBusContext(CAN1);
    setSignal(IgnitionSwitch, 1);
    testWaitForTimeout(500);
    setSignal(StartButton, 1);
    testWaitForTimeout(1000);

    if (getSignal(EngineRunning) != 1)
    {
        testStepFail("Engine did not start");
    }
    /* No cleanup — signal states leak into next test */
}

testcase TC_EngineStop()
{
    /* Same setup duplicated again */
    setBusContext(CAN1);
    setSignal(IgnitionSwitch, 1);
    testWaitForTimeout(500);
    /* Test depends on leaked state from TC_EngineStart */
    setSignal(IgnitionSwitch, 0);
    testWaitForTimeout(2000);

    if (getSignal(EngineRunning) != 0)
    {
        testStepFail("Engine did not stop");
    }
}
```

**Correct (structured test unit with fixtures):**

```capl
/*@@testunit: TU_EngineControl */
/*@@description: Engine start/stop sequence validation */
/*@@version: 1.2.0 */

/*@@testgroup: TG_EngineStartStop */
/*@@description: Tests for ignition and engine start/stop sequences */

/* Shared state for the test group */
variables
{
    const int IGNITION_ON = 1;
    const int IGNITION_OFF = 0;
    const int ENGINE_RUNNING = 1;
    const int ENGINE_STOPPED = 0;
    const int STARTUP_TIMEOUT_MS = 2000;
}

/* Runs before EACH test case in this group */
@setup
void Setup_EngineTests()
{
    testStep("Setup", "Initializing test environment");

    /* Reset all signals to known state */
    setBusContext(CAN1);
    setSignal(IgnitionSwitch, IGNITION_OFF);
    setSignal(StartButton, 0);
    setSignal(BrakePedal, 0);

    /* Wait for ECU to process reset */
    testWaitForTimeout(200);

    /* Verify clean starting state */
    testStep("Setup", "Verifying initial conditions");
    if (getSignal(EngineRunning) != ENGINE_STOPPED)
    {
        testStepFail("Setup", "Engine not in stopped state before test");
        testAbort("Precondition failed");
    }
}

/* Runs after EACH test case in this group */
@teardown
void Teardown_EngineTests()
{
    testStep("Teardown", "Cleaning up test environment");

    /* Force engine stop to prevent state leakage */
    setSignal(IgnitionSwitch, IGNITION_OFF);
    setSignal(StartButton, 0);
    testWaitForTimeout(500);

    /* Log final signal states for debugging */
    testStep("Teardown", "Engine state: %d", getSignal(EngineRunning));
}

@testcase
void TC_EngineStart_NormalSequence()
{
    testCaseTitle("TC-ENG-001", "Normal engine start sequence");
    testCaseDescription("Verify engine starts with ignition ON + start button");

    testStep("Action", "Turn ignition ON");
    setSignal(IgnitionSwitch, IGNITION_ON);
    testWaitForTimeout(500);

    testStep("Action", "Press start button with brake applied");
    setSignal(BrakePedal, 1);
    setSignal(StartButton, 1);

    testStep("Verify", "Wait for engine running indication");
    if (testWaitForSignalMatch(EngineRunning, ENGINE_RUNNING,
                                STARTUP_TIMEOUT_MS) == 0)
    {
        testStepPass("Verify", "Engine started within timeout");
    }
    else
    {
        testStepFail("Verify", "Engine did not start within %d ms",
                     STARTUP_TIMEOUT_MS);
    }
}

@testcase
void TC_EngineStart_NoBrake()
{
    testCaseTitle("TC-ENG-002", "Engine start rejected without brake");
    testCaseDescription("Verify engine does not start without brake pedal");

    testStep("Action", "Turn ignition ON");
    setSignal(IgnitionSwitch, IGNITION_ON);
    testWaitForTimeout(500);

    testStep("Action", "Press start button WITHOUT brake");
    setSignal(BrakePedal, 0);
    setSignal(StartButton, 1);
    testWaitForTimeout(STARTUP_TIMEOUT_MS);

    testStep("Verify", "Engine must remain stopped");
    if (getSignal(EngineRunning) == ENGINE_STOPPED)
    {
        testStepPass("Verify", "Engine correctly rejected start without brake");
    }
    else
    {
        testStepFail("Verify", "Engine started without brake — safety violation");
    }
}
```

**Test execution order control:**

```capl
/*@@testgroup: TG_OrderedTests */
/*@@execution_order: sequential */

/* Tests execute in declaration order when sequential is specified.
 * Use 'parallel' for independent tests that can run in any order.
 * Use 'random' for robustness testing against order dependencies. */

@testcase @order(1)
void TC_Precondition_Check() { /* runs first */ }

@testcase @order(2)
void TC_Main_Scenario() { /* runs second */ }

@testcase @order(3)
void TC_Postcondition_Verify() { /* runs last */ }
```

Reference: Vector vTESTstudio User Manual — Test Architecture, CANoe Test Feature Set documentation
