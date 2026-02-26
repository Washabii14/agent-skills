---
title: Parasoft C/C++test for Automotive Compliance
impact: MEDIUM
impactDescription: Parasoft provides integrated MISRA/AUTOSAR rule checking, unit test generation, and runtime analysis with pre-packaged ISO 26262 and DO-178C qualification kits
tags: analysis, parasoft, misra, autosar, unit-test, runtime-analysis, iso-26262, do-178c, qualification
---

## Parasoft C/C++test for Automotive Compliance

Parasoft C/C++test is an integrated static analysis, unit testing, and runtime verification tool. It ships with pre-configured rule packs for MISRA C:2012, AUTOSAR C++14, and CERT, plus ISO 26262 and DO-178C qualification kits.

**Incorrect (using Parasoft with default rule set):**

```
# Parasoft C/C++test — default configuration
# Test Configuration: "Recommended Rules"
# Result: generic findings not aligned with automotive standards
# No MISRA compliance report, no unit test generation
# Qualification kit not activated — tool evidence not valid for ISO 26262
```

**Correct (automotive test configuration):**

```xml
<!-- parasoft_automotive.properties — Parasoft test configuration -->
<TestConfig name="Automotive_ASIL_B">
    <!-- Static Analysis Rule Packs -->
    <RulePacks>
        <RulePack>MISRA C:2012 (Mandatory)</RulePack>
        <RulePack>MISRA C:2012 (Required)</RulePack>
        <RulePack>MISRA C:2012 Amendment 1</RulePack>
        <RulePack>AUTOSAR C++14 Guidelines</RulePack>
        <RulePack>CERT C Secure Coding</RulePack>
        <RulePack>CWE Top 25 2023</RulePack>
    </RulePacks>

    <!-- Severity mapping for automotive -->
    <SeverityMapping>
        <Map from="MISRA-Mandatory" to="Highest"/>
        <Map from="MISRA-Required" to="High"/>
        <Map from="MISRA-Advisory" to="Medium"/>
        <Map from="AUTOSAR-Required" to="High"/>
        <Map from="AUTOSAR-Advisory" to="Medium"/>
    </SeverityMapping>

    <!-- Compiler configuration -->
    <Compiler>
        <Family>GCC</Family>
        <Version>10</Version>
        <Target>arm-none-eabi</Target>
        <CStandard>C11</CStandard>
        <CppStandard>C++14</CppStandard>
    </Compiler>

    <!-- Unit Test Generation Settings -->
    <UnitTesting>
        <GenerateStubs>true</GenerateStubs>
        <StubGeneration>safe-defaults</StubGeneration>
        <CoverageTarget>MC/DC</CoverageTarget>
        <CoverageThreshold>100</CoverageThreshold>
    </UnitTesting>

    <!-- Runtime Analysis -->
    <RuntimeAnalysis>
        <MemoryMonitoring>true</MemoryMonitoring>
        <StackOverflowDetection>true</StackOverflowDetection>
        <ArrayBoundsChecking>true</ArrayBoundsChecking>
        <UninitializedMemory>true</UninitializedMemory>
    </RuntimeAnalysis>
</TestConfig>
```

**Incorrect (no unit test stubs for hardware dependencies):**

```c
/* Unit test attempts to call real hardware — fails on host */
void test_ReadSensor(void)
{
    int32_t value = HAL_ADC_Read(ADC_CHANNEL_0);  /* Crashes: no HW */
    ASSERT_IN_RANGE(value, 0, 4095);
}
```

**Correct (Parasoft-generated stubs with safe defaults):**

```c
/* Parasoft auto-generated stub for HAL_ADC_Read */
/* File: stubs/hal_adc_stub.c */
#include "hal_adc.h"
#include "cpptest.h"  /* Parasoft test framework */

/* Stub with configurable return values */
static int32_t HAL_ADC_Read_stub_return = 0;
static uint32_t HAL_ADC_Read_stub_call_count = 0U;

int32_t HAL_ADC_Read(uint8_t channel)
{
    HAL_ADC_Read_stub_call_count++;
    CPPTEST_REGISTER_STUB_CALL("HAL_ADC_Read");
    CPPTEST_REGISTER_STUB_PARAM("channel", channel);
    return HAL_ADC_Read_stub_return;
}

/* Test case using stub */
void test_ReadSensor_NominalRange(void)
{
    HAL_ADC_Read_stub_return = 2048;  /* Mid-range ADC value */
    int32_t result = Sensor_GetTemperature();
    CPPTEST_ASSERT_INTEGER_BETWEEN(20, 30, result);  /* 20-30°C expected */
    CPPTEST_ASSERT_EQUAL(1U, HAL_ADC_Read_stub_call_count);
}

void test_ReadSensor_Saturation(void)
{
    HAL_ADC_Read_stub_return = 4095;  /* Maximum ADC value */
    int32_t result = Sensor_GetTemperature();
    CPPTEST_ASSERT_INTEGER_EQUAL(MAX_TEMP_LIMIT, result);
}
```

**CI/CD integration with qualification evidence:**

```bash
#!/bin/bash
# parasoft_ci.sh — generate compliance report for ISO 26262 evidence

cpptestcli \
    -config "Automotive_ASIL_B" \
    -compiler gcc_10_arm \
    -input project.bdf \
    -report build/parasoft_report \
    -publish \
    -property report.format=html,xml,sarif \
    -property report.compliance.misra_c_2012=true \
    -property report.compliance.autosar_cpp14=true \
    -property tool.qualification.iso26262=true \
    -property tool.qualification.asil=B
```

Reference: ISO 26262-6 §9 (unit verification methods), ISO 26262-8 §11 (tool qualification), MISRA C:2012 Appendix A (tool chain qualification)
