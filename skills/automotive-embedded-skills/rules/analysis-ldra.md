---
title: LDRA Static Analysis and Certification Evidence
impact: MEDIUM
impactDescription: LDRA provides integrated static analysis, code coverage, and requirements traceability — producing the complete evidence package required for ISO 26262 certification
tags: analysis, ldra, tbvision, coverage, traceability, iso-26262, certification, requirements, mc-dc
---

## LDRA Static Analysis and Certification Evidence

LDRA TBvision provides a unified environment for static analysis, dynamic testing, code coverage measurement, and requirements traceability. It generates the complete verification evidence package required for ISO 26262 Part 6 certification.

**Incorrect (using LDRA for static analysis only):**

```
# Only running static analysis — missing coverage and traceability
ldra_analyze -project ecu_app -rules misra_c2012
# Result: MISRA report generated, but no coverage data,
# no requirements mapping, incomplete certification evidence
```

**Correct (full LDRA verification pipeline):**

```
# ldra_project.cfg — complete verification configuration

[Project]
ProjectName = ECU_MotorControl
Language = C
Standard = C11
Compiler = ARM_GCC_10
Target = Cortex-M4

[StaticAnalysis]
RuleSet = MISRA_C_2012_Required_Mandatory
AdditionalRules = CERT_C, CWE_Top25
Complexity.MaxCyclomaticComplexity = 15
Complexity.MaxNesting = 4
Complexity.MaxParameters = 6
Metrics.MaxLinesPerFunction = 75
Metrics.MaxStatementsPerFunction = 50

[DynamicAnalysis]
CoverageLevel = MC/DC
MinStatementCoverage = 100
MinBranchCoverage = 100
MinMCDCCoverage = 100
InstrumentationMode = source

[Traceability]
RequirementsSource = DOORS
RequirementsProject = ECU_MotorCtrl_SWR
BidirectionalTrace = true
OrphanDetection = true
```

**Incorrect (code coverage without requirements link):**

```c
/* Test achieves 100% statement coverage but has no
 * traceability to software requirements */
void test_motor_speed(void)
{
    Motor_SetSpeed(1000);
    assert(Motor_GetSpeed() == 1000);
    Motor_SetSpeed(0);
    assert(Motor_GetSpeed() == 0);
    /* Which requirement does this test verify? Unknown. */
}
```

**Correct (test with requirements traceability annotations):**

```c
/* LDRA requirement tags link tests to DOORS requirements */

/* @LDRA_REQUIREMENT: SWR-MC-042 Motor speed control range
 * @LDRA_REQUIREMENT: SWR-MC-043 Motor speed saturation limits */
void test_MotorSpeed_NominalRange(void)
{
    Motor_Init();

    /* Verify nominal speed setting — SWR-MC-042 */
    Motor_SetSpeed(1000);
    TEST_ASSERT_EQUAL_INT32(1000, Motor_GetSpeed());

    /* Verify upper saturation — SWR-MC-043 */
    Motor_SetSpeed(MAX_SPEED + 100);
    TEST_ASSERT_EQUAL_INT32(MAX_SPEED, Motor_GetSpeed());

    /* Verify lower bound — SWR-MC-043 */
    Motor_SetSpeed(-100);
    TEST_ASSERT_EQUAL_INT32(0, Motor_GetSpeed());
}

/* @LDRA_REQUIREMENT: SWR-MC-044 Motor emergency stop */
void test_MotorSpeed_EmergencyStop(void)
{
    Motor_SetSpeed(5000);
    Motor_EmergencyStop();
    TEST_ASSERT_EQUAL_INT32(0, Motor_GetSpeed());
    TEST_ASSERT_TRUE(Motor_IsStopped());
}
```

**Coverage instrumentation and reporting:**

```c
/* LDRA instruments source for MC/DC coverage measurement */

/* Original source */
bool Motor_IsOvertemp(int32_t temp, bool sensor_valid)
{
    if ((temp > TEMP_THRESHOLD) && sensor_valid)
    {
        return true;
    }
    return false;
}

/* MC/DC requires test vectors for all condition combinations:
 *
 * | Test | temp > THRESH | sensor_valid | Result | Proves        |
 * |------|---------------|--------------|--------|---------------|
 * | TC1  | true          | true         | true   | Both true     |
 * | TC2  | false         | true         | false  | temp decides  |
 * | TC3  | true          | false        | false  | sensor decides|
 *
 * Three test cases achieve 100% MC/DC for this decision.
 * LDRA TBvision verifies and reports this automatically.
 */
```

**ISO 26262 certification evidence output:**

| LDRA Artifact | ISO 26262 Requirement |
|--------------|----------------------|
| Static Analysis Report | Part 6, §9.4.1 (static analysis) |
| MISRA Compliance Matrix | Part 6, §9.4.1 (coding guidelines) |
| MC/DC Coverage Report | Part 6, §9.4.5 (structural coverage) |
| Requirements Traceability Matrix | Part 8, §6.4.4 (bidirectional traceability) |
| Complexity Metrics Report | Part 6, §9.4.2 (complexity metrics) |
| Dead Code Report | Part 6, §9.4.1 (unreachable code detection) |

Reference: ISO 26262-6 §9 (software unit verification), ISO 26262-6 Table 9 (structural coverage metrics), ISO 26262-8 §6.4.4 (traceability)
