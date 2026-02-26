---
title: Stimulus/Response Timing in vTESTstudio
impact: HIGH
impactDescription: Correct use of testWaitForSignalMatch vs testWaitForTimeout with timing tolerances is critical for reliable real-time test validation and avoiding false test failures
tags: capl, vtestudio, timing, stimulus-response, testWaitForSignalMatch, testWaitForTimeout, real-time, synchronization, non-deterministic
---

## Stimulus/Response Timing in vTESTstudio

Real-time embedded testing requires careful handling of timing between stimulus and response. Using `testWaitForSignalMatch` (event-driven) instead of `testWaitForTimeout` (fixed delay) makes tests faster and more reliable. Timing tolerances must account for ECU processing time, bus latency, and hardware response variation.

**Incorrect (fixed delays and tight timing assumptions):**

```capl
testcase TC_AirbagDeployment_BAD()
{
    /* Bad: arbitrary fixed delay instead of event-driven wait */
    setSignal(CrashSensor_G, 25.0);
    testWaitForTimeout(50);  /* Assumes exactly 50ms response — brittle */

    /* Bad: checking signal at one exact moment */
    if (getSignal(AirbagDeploy) != 1)
    {
        testStepFail("Airbag not deployed");
    }
    /* Problem: if ECU responds at 51ms, test fails incorrectly.
     * If ECU responds at 30ms but signal toggles, we miss it. */

    /* Bad: no timing measurement — cannot verify real-time requirements */
}
```

**Correct (event-driven wait with timing measurement):**

```capl
/*@@testunit: TU_AirbagSystem */

variables
{
    const float CRASH_THRESHOLD_G = 20.0;
    const int DEPLOY_MAX_MS = 30;       /* ISO requirement: < 30ms */
    const int DEPLOY_TOLERANCE_MS = 5;  /* Measurement tolerance */
    const int RESPONSE_WINDOW_MS = DEPLOY_MAX_MS + DEPLOY_TOLERANCE_MS;
}

@testcase
void TC_AirbagDeployment_TimingVerification()
{
    testCaseTitle("TC-AIR-001", "Airbag deployment timing verification");
    testCaseDescription("Verify airbag deploys within 30ms of crash threshold");

    /* Record timestamp before stimulus */
    dword t_stimulus = timeNow();

    testStep("Stimulus", "Applying crash signal: %.1f G", CRASH_THRESHOLD_G);
    setSignal(CrashSensor_G, CRASH_THRESHOLD_G);

    /* Event-driven wait — returns immediately when signal matches */
    testStep("Wait", "Waiting for airbag deploy signal (max %d ms)",
             RESPONSE_WINDOW_MS);
    int result = testWaitForSignalMatch(AirbagDeploy, 1, RESPONSE_WINDOW_MS);

    /* Measure actual response time */
    dword t_response = timeNow();
    float response_time_ms = (float)(t_response - t_stimulus) / 1000.0;

    if (result == 0)
    {
        /* Signal matched within window — now verify timing requirement */
        if (response_time_ms <= (float)DEPLOY_MAX_MS)
        {
            testStepPass("Timing",
                "Airbag deployed in %.2f ms (requirement: < %d ms)",
                response_time_ms, DEPLOY_MAX_MS);
        }
        else
        {
            testStepFail("Timing",
                "Airbag deployed in %.2f ms — EXCEEDS requirement of %d ms",
                response_time_ms, DEPLOY_MAX_MS);
        }
    }
    else
    {
        testStepFail("Timing",
            "Airbag did NOT deploy within %d ms window "
            "(elapsed: %.2f ms, signal=%d)",
            RESPONSE_WINDOW_MS, response_time_ms, getSignal(AirbagDeploy));

        CaptureFailureEvidence("TC-AIR-001", "no_deployment");
    }
}
```

**Correct (testWaitForSignalMatch vs testWaitForTimeout decision):**

```capl
/* USE testWaitForSignalMatch WHEN:
 * - You know the expected signal value
 * - You need to measure response time
 * - The response should be as fast as possible */

@testcase
void TC_BrakeLight_ResponseTime()
{
    testStep("Stimulus", "Applying brake pedal");
    dword t_start = timeNow();
    setSignal(BrakePedal, 80);

    /* Correct: event-driven, returns as soon as signal matches */
    int result = testWaitForSignalMatch(BrakeLight, 1, 200);
    float response_ms = (float)(timeNow() - t_start) / 1000.0;

    if (result == 0)
    {
        testStepPass("Response", "Brake light ON in %.1f ms", response_ms);
    }
    else
    {
        testStepFail("Response", "Brake light not ON after 200ms");
    }
}

/* USE testWaitForTimeout WHEN:
 * - You need a fixed settling time (e.g., signal debounce)
 * - You're verifying a signal does NOT change
 * - You're synchronizing with a periodic process */

@testcase
void TC_BrakeLight_NoFalseActivation()
{
    testStep("Stimulus", "Light brake pedal touch (below threshold)");
    setSignal(BrakePedal, 3);  /* Below 5% activation threshold */

    /* Correct: fixed wait to verify signal remains OFF */
    testStep("Observe", "Monitoring for 2 seconds");
    testWaitForTimeout(2000);

    if (getSignal(BrakeLight) == 0)
    {
        testStepPass("Verify", "Brake light correctly stayed OFF for 2s");
    }
    else
    {
        testStepFail("Verify", "Brake light falsely activated at %d%% pedal",
                     getSignal(BrakePedal));
    }
}
```

**Handling non-deterministic timing (signal debounce, network jitter):**

```capl
@testcase
void TC_EngineStart_WithDebounce()
{
    testCaseTitle("TC-ENG-010", "Engine start with debounce handling");

    testStep("Stimulus", "Pressing start button");
    setSignal(StartButton, 1);

    /* Non-deterministic: ECU debounces button for 50-100ms,
     * then cranking takes 500-2000ms depending on temperature.
     * Use multi-stage waiting with intermediate checks. */

    /* Stage 1: Wait for cranking state (debounce + processing) */
    testStep("Phase1", "Waiting for cranking phase");
    int crank_result = testWaitForSignalMatch(
        EngineState, STATE_CRANKING, 500);

    if (crank_result != 0)
    {
        testStepFail("Phase1",
            "Engine did not enter cranking within 500ms (state=%d)",
            getSignal(EngineState));
        return;
    }
    testStepPass("Phase1", "Cranking started");

    /* Stage 2: Wait for running state (variable cranking duration) */
    testStep("Phase2", "Waiting for engine running (max 3s)");
    int run_result = testWaitForSignalMatch(
        EngineState, STATE_RUNNING, 3000);

    if (run_result == 0)
    {
        testStepPass("Phase2", "Engine running");
    }
    else
    {
        /* Not necessarily a failure — check if still cranking */
        int current_state = getSignal(EngineState);
        if (current_state == STATE_CRANKING)
        {
            testStepFail("Phase2",
                "Engine still cranking after 3s — possible starter fault");
        }
        else
        {
            testStepFail("Phase2",
                "Unexpected engine state: %d (expected RUNNING=%d)",
                current_state, STATE_RUNNING);
        }
    }
}
```

**Synchronization with hardware signals (HIL-specific):**

```capl
/* Synchronizing with hardware PWM signals on HIL bench */
@testcase
void TC_FanPWM_DutyCycle()
{
    testCaseTitle("TC-FAN-001", "Fan PWM duty cycle verification");

    setSignal(CoolantTemp, 95.0);  /* Above fan activation threshold */
    testWaitForTimeout(1000);       /* Allow controller to stabilize */

    /* Measure PWM over multiple cycles for accuracy */
    testStep("Measure", "Sampling PWM duty cycle over 500ms window");
    float duty_measured = testMeasurePWMDutyCycle(FanPWM_Pin, 500);
    float duty_expected = 75.0;  /* 75% expected at 95°C */
    float duty_tolerance = 5.0;  /* ±5% tolerance */

    if (fabs(duty_measured - duty_expected) <= duty_tolerance)
    {
        testStepPass("DutyCycle",
            "PWM duty=%.1f%% (expected %.1f%% ±%.1f%%)",
            duty_measured, duty_expected, duty_tolerance);
    }
    else
    {
        testStepFail("DutyCycle",
            "PWM duty=%.1f%% OUT OF RANGE (expected %.1f%% ±%.1f%%)",
            duty_measured, duty_expected, duty_tolerance);
    }
}
```

**Timing strategy selection guide:**

| Scenario | Function | Rationale |
|----------|----------|-----------|
| Signal should become X | `testWaitForSignalMatch(sig, X, timeout)` | Event-driven, measures actual response time |
| Signal must NOT change | `testWaitForTimeout(duration)` + check | Proves stability over observation window |
| Multi-phase response | Chained `testWaitForSignalMatch` calls | Validates each phase independently |
| Periodic signal (PWM) | `testWaitForTimeout` + measure over window | Needs multiple cycles for accuracy |
| Hardware sync (HIL) | `testWaitForSignalMatch` + hardware trigger | Synchronizes with physical events |

Reference: Vector vTESTstudio User Manual — Timing Functions, CANoe Help — Real-Time Test Execution, ISO 26262-6 §10 (integration testing timing requirements)
