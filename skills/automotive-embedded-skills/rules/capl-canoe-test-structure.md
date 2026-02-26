---
title: Test Case Structure
impact: HIGH
impactDescription: Well-structured test cases are reliable, reproducible, and produce clear pass/fail verdicts
tags: capl, testing, test-case, canoe, verification, test-structure
---

## Test Case Structure

Structure test cases with clear setup, stimulus, wait, and verification phases. Use `testStep` for traceability and `testWaitForSignalMatch` for deterministic signal verification.

**Incorrect (unstructured test with no phases or verdict):**

```capl
testcase TC_EngineStart_Bad()
{
    setSignal(IgnitionSwitch, 1);
    testWaitForTimeout(3000);
    /* No verification, no clear pass/fail */
}
```

**Correct (structured test with setup, stimulus, wait, and verification):**

```capl
testcase TC_EngineStart()
{
    /* Setup */
    testStep("Preconditions", "Set initial conditions");
    setSignal(IgnitionSwitch, 0);
    testWaitForTimeout(500);

    /* Stimulus */
    testStep("Action", "Turn ignition ON");
    setSignal(IgnitionSwitch, 1);

    /* Wait for response */
    testStep("Wait", "Wait for engine response");
    if (testWaitForSignalMatch(EngineRunning, 1, 2000) != 0)
    {
        testStepFail("Verification", "Engine did not start within 2s");
        return;
    }

    /* Verification */
    testStepPass("Verification", "Engine started successfully");
}
```

Each test phase should be documented with `testStep`. Timeouts must be chosen to match the system's specified response time. Always produce an explicit pass or fail verdict.

Reference: Vector CANoe Test Feature Set — CAPL Test Functions
