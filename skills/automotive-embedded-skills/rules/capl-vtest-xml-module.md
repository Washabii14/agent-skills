---
title: XML Test Module Integration in vTESTstudio
impact: MEDIUM
impactDescription: XML test modules bridge CAPL test implementations to formal test specifications, enabling automated execution via command line and standardized HTML/PDF report generation
tags: capl, vtestudio, xml, test-module, automation, command-line, report, html, pdf
---

## XML Test Module Integration in vTESTstudio

XML test modules map CAPL test case implementations to formal test specifications defined in XML. This enables test management tools to drive test execution, supports command-line automation for CI/CD, and produces standardized reports (HTML/PDF) for certification evidence.

**Incorrect (CAPL tests with no XML specification mapping):**

```capl
/* Tests exist only as CAPL code — no formal specification link,
 * no automated execution possible, no standardized reports */
testcase TC_BrakeLightOn()
{
    setSignal(BrakePedal, 80);
    testWaitForTimeout(200);
    if (getSignal(BrakeLight) != 1)
        testStepFail("Brake light not ON");
}
/* Cannot run from command line, no report generation,
 * no traceability to test specification */
```

**Correct (XML test specification with CAPL mapping):**

```xml
<!-- test_specs/braking_system_tests.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<testmodule version="2.0"
            name="TM_BrakingSystem"
            description="Braking system functional tests per SRS-BRK">

    <preparation>
        <initialize signal="IgnitionSwitch" value="1"/>
        <initialize signal="VehicleSpeed" value="0"/>
        <wait duration="500"/>
    </preparation>

    <testgroup name="TG_BrakeLightControl"
               description="Brake light activation and deactivation">

        <testcase name="TC_BrakeLight_Activation"
                  id="TC-BRK-001"
                  requirement="SRS-BRK-042"
                  description="Brake light activates when pedal > 10%">
            <steps>
                <step action="Set BrakePedal to 15%"
                      expected="BrakeLight signal = ON within 100ms"/>
            </steps>
            <!-- Maps to CAPL implementation -->
            <implementation language="CAPL"
                           function="TC_BrakeLight_Activation"/>
        </testcase>

        <testcase name="TC_BrakeLight_Deactivation"
                  id="TC-BRK-002"
                  requirement="SRS-BRK-043"
                  description="Brake light deactivates when pedal released">
            <steps>
                <step action="Release BrakePedal to 0%"
                      expected="BrakeLight signal = OFF within 200ms"/>
            </steps>
            <implementation language="CAPL"
                           function="TC_BrakeLight_Deactivation"/>
        </testcase>

        <testcase name="TC_BrakeLight_EmergencyBraking"
                  id="TC-BRK-003"
                  requirement="SRS-BRK-044,SRS-BRK-045"
                  description="Brake light flashes during emergency braking">
            <parameters>
                <param name="deceleration_threshold" value="8.0"/>
                <param name="flash_frequency_hz" value="4.0"/>
            </parameters>
            <implementation language="CAPL"
                           function="TC_BrakeLight_EmergencyBraking"/>
        </testcase>
    </testgroup>

    <completion>
        <initialize signal="BrakePedal" value="0"/>
        <initialize signal="IgnitionSwitch" value="0"/>
    </completion>
</testmodule>
```

**CAPL implementation linked to XML spec:**

```capl
/*@@testunit: TU_BrakingSystem */
/*@@xml_testmodule: test_specs/braking_system_tests.xml */

@testcase
void TC_BrakeLight_Activation()
{
    testCaseTitle("TC-BRK-001", "Brake light activation at >10% pedal");

    testStep("Stimulate", "Applying 15%% brake pedal");
    setSignal(BrakePedal, 15);

    testStep("Verify", "Checking brake light within 100ms");
    if (testWaitForSignalMatch(BrakeLight, 1, 100) == 0)
    {
        testStepPass("Result", "Brake light activated within 100ms");
    }
    else
    {
        testStepFail("Result", "Brake light did not activate within 100ms");
    }
}

@testcase
void TC_BrakeLight_Deactivation()
{
    testCaseTitle("TC-BRK-002", "Brake light deactivation on pedal release");

    setSignal(BrakePedal, 50);
    testWaitForTimeout(200);

    testStep("Stimulate", "Releasing brake pedal");
    setSignal(BrakePedal, 0);

    testStep("Verify", "Checking brake light off within 200ms");
    if (testWaitForSignalMatch(BrakeLight, 0, 200) == 0)
    {
        testStepPass("Result", "Brake light deactivated within 200ms");
    }
    else
    {
        testStepFail("Result", "Brake light remained ON after pedal release");
    }
}
```

**Command-line automated execution:**

```bash
#!/bin/bash
# run_tests.sh — automated test execution via CANoe command line

# Execute XML test module with specified configuration
"C:/Vector/CANoe/Exec64/CANoe64.exe" \
    -config "configs/BrakingSystem.cfg" \
    -testmodule "test_specs/braking_system_tests.xml" \
    -autostart \
    -autoexit \
    -reportdir "reports/" \
    -reportformat html,pdf \
    -reportname "BrakingSystem_$(date +%Y%m%d_%H%M%S)" \
    -loglevel verbose

# Check exit code for CI/CD pass/fail
if [ $? -ne 0 ]; then
    echo "ERROR: Test execution failed"
    exit 1
fi

echo "Reports generated in reports/ directory"
```

**Report configuration for certification:**

```xml
<!-- report_template.xml — customize HTML/PDF report output -->
<reportconfig>
    <header>
        <project>ECU Braking System</project>
        <version>${PROJECT_VERSION}</version>
        <date>${EXECUTION_DATE}</date>
        <tester>${TESTER_NAME}</tester>
    </header>
    <content>
        <include section="summary"/>
        <include section="testcase_details"/>
        <include section="signal_traces"/>
        <include section="verdicts"/>
        <include section="requirements_coverage"/>
    </content>
    <format>
        <pdf enabled="true" template="iso26262_report.xsl"/>
        <html enabled="true" include_graphs="true"/>
    </format>
</reportconfig>
```

Reference: Vector vTESTstudio User Manual — XML Test Modules, CANoe Command Line Interface documentation
