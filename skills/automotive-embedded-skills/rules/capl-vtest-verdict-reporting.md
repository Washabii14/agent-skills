---
title: Verdict and Reporting Best Practices in vTESTstudio
impact: HIGH
impactDescription: Structured verdict messages with testStepPass/testStepFail enable automated result analysis, failure triage, and integration with test management tools (DOORS, Polarion)
tags: capl, vtestudio, verdict, reporting, testStepPass, testStepFail, screenshot, doors, polarion, test-management
---

## Verdict and Reporting Best Practices in vTESTstudio

Every test step must produce a clear, structured verdict using `testStepPass` / `testStepFail`. Verdict messages should contain the expected value, actual value, and signal context to enable automated triage without re-running tests. Integrate with test management tools (DOORS, Polarion) for traceability.

**Incorrect (vague or missing verdict messages):**

```capl
testcase TC_TurnSignal()
{
    setSignal(TurnSignalLever, 1);
    testWaitForTimeout(500);

    /* Bad: binary pass/fail with no context */
    if (getSignal(LeftIndicator) == 1)
        testStepPass("");  /* No description — useless in reports */
    else
        testStepFail("fail");  /* No detail — which signal? what value? */

    /* Bad: using write() instead of testStep functions */
    write("Test passed maybe");

    /* Bad: no verdict at all for some conditions */
    if (getSignal(HazardWarning) == 0)
    {
        /* silently passes — missing result never appears in report */
    }
}
```

**Correct (structured verdict messages):**

```capl
/*@@testunit: TU_TurnSignals */

@testcase
void TC_TurnSignal_LeftActivation()
{
    testCaseTitle("TC-TURN-001", "Left turn signal activation");
    testCaseDescription("Verify left indicator activates when lever pulled");

    /* Step 1: Precondition check */
    testStep("Precondition", "Verify all indicators OFF");
    int left_init = getSignal(LeftIndicator);
    int right_init = getSignal(RightIndicator);
    if ((left_init == 0) && (right_init == 0))
    {
        testStepPass("Precondition",
            "All indicators OFF (Left=%d, Right=%d)", left_init, right_init);
    }
    else
    {
        testStepFail("Precondition",
            "Indicators not OFF (Left=%d, Right=%d) — aborting",
            left_init, right_init);
        return;
    }

    /* Step 2: Stimulus */
    testStep("Stimulus", "Activating left turn signal lever");
    setSignal(TurnSignalLever, LEVER_LEFT);

    /* Step 3: Response with tolerance */
    testStep("Response", "Waiting for left indicator activation");
    int wait_result = testWaitForSignalMatch(LeftIndicator, 1, 300);

    /* Step 4: Explicit verdict with all context */
    int left_actual = getSignal(LeftIndicator);
    int right_actual = getSignal(RightIndicator);

    if ((wait_result == 0) && (left_actual == 1) && (right_actual == 0))
    {
        testStepPass("Verdict",
            "Left indicator ON (=%d), Right indicator OFF (=%d), "
            "response time within 300ms",
            left_actual, right_actual);
    }
    else
    {
        testStepFail("Verdict",
            "Expected: Left=1, Right=0 within 300ms. "
            "Actual: Left=%d, Right=%d, waitResult=%d",
            left_actual, right_actual, wait_result);

        /* Capture screenshot on failure for visual evidence */
        testStepScreenshot("Failure_TC_TURN_001",
            "Signal state at failure point");
    }

    /* Step 5: Cleanup verdict */
    testStep("Cleanup", "Deactivating turn signal");
    setSignal(TurnSignalLever, LEVER_NEUTRAL);
    testWaitForTimeout(500);

    if (getSignal(LeftIndicator) == 0)
    {
        testStepPass("Cleanup", "Left indicator returned to OFF");
    }
    else
    {
        testStepFail("Cleanup",
            "Left indicator stuck ON after lever neutral (=%d)",
            getSignal(LeftIndicator));
    }
}
```

**Screenshot capture on failure:**

```capl
/* Capture CANoe panel screenshot when test fails */
void CaptureFailureEvidence(char testId[], char context[])
{
    char filename[256];
    snprintf(filename, sizeof(filename),
             "screenshots/%s_%s.png", testId, context);

    /* Capture measurement window screenshot */
    testStepScreenshot(filename,
        "Failure evidence for %s — %s", testId, context);

    /* Also log signal snapshot for offline analysis */
    testStep("Evidence", "Signal snapshot at failure:");
    testStep("Evidence", "  VehicleSpeed = %.1f kph", getSignal(VehicleSpeed));
    testStep("Evidence", "  EngineRPM = %.0f", getSignal(EngineRPM));
    testStep("Evidence", "  BrakePedal = %.1f%%", getSignal(BrakePedal));
    testStep("Evidence", "  SteeringAngle = %.1f deg",
             getSignal(SteeringAngle));
}
```

**Test report customization:**

```capl
/* Custom report metadata for test management integration */
@setup
void Setup_WithReportMetadata()
{
    /* Tag test execution with metadata for DOORS/Polarion */
    testReportAddInfo("Project", "ECU_BodyControl_v2.3");
    testReportAddInfo("SUT_Version", "SW_1.4.2_HW_3.0");
    testReportAddInfo("TestEnvironment", "HIL_Bench_04");
    testReportAddInfo("Tester", getenv("USERNAME"));
    testReportAddInfo("DOORS_Baseline", "BL_2024_Q3_Release");
}
```

**Integration with test management tools:**

```xml
<!-- Export format for DOORS/Polarion import -->
<!-- vTESTstudio generates this automatically when configured -->
<test_execution_results>
    <execution date="2024-06-15" config="HIL_Bench_04">
        <testcase id="TC-TURN-001"
                  requirement="SRS-BODY-112"
                  verdict="PASS"
                  duration_ms="1847">
            <steps>
                <step name="Precondition" verdict="PASS"/>
                <step name="Stimulus" verdict="INFO"/>
                <step name="Response" verdict="PASS"/>
                <step name="Verdict" verdict="PASS"/>
                <step name="Cleanup" verdict="PASS"/>
            </steps>
        </testcase>
        <testcase id="TC-TURN-002"
                  requirement="SRS-BODY-113"
                  verdict="FAIL"
                  duration_ms="2103">
            <steps>
                <step name="Verdict" verdict="FAIL"
                      message="Expected: Right=1 within 300ms. Actual: Right=0"/>
            </steps>
            <attachments>
                <screenshot path="screenshots/TC-TURN-002_failure.png"/>
            </attachments>
        </testcase>
    </execution>
</test_execution_results>
```

**Verdict message format standard:**

| Component | Example | Purpose |
|-----------|---------|---------|
| Step name | `"Verdict"` | Categorizes the step in reports |
| Expected | `"Expected: Left=1, Right=0"` | Documents pass criteria |
| Actual | `"Actual: Left=0, Right=0"` | Documents observed behavior |
| Timing | `"within 300ms"` | Documents timing constraint |
| Context | `"waitResult=-1"` | Aids failure root cause analysis |

Reference: Vector vTESTstudio User Manual — Test Reporting, CANoe Help — Test Management Tool Integration
