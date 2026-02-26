---
title: Integration Testing Patterns
impact: MEDIUM
impactDescription: Validates inter-module behavior and communication paths
tags: test, integration, modules, communication, verification
---

## Integration Testing Patterns

Test module interactions and communication paths. Integration tests verify that connected modules exchange data correctly and handle error propagation across interfaces.

**Incorrect (no integration test — only isolated unit tests):**

```c
void test_SensorModule(void)
{
    TEST_ASSERT_TRUE(Sensor_Read() == E_OK);
}
void test_FilterModule(void)
{
    TEST_ASSERT_FLOAT_WITHIN(0.1f, 25.0f, Filter_Apply(25.0f));
}
```

**Correct (integration test across module boundary):**

```c
void test_SensorToFilterIntegration(void)
{
    /* Initialize both modules */
    Sensor_Init(&sensorConfig);
    Filter_Init(&filterConfig);

    /* Inject known sensor reading */
    MockHal_SetAdcValue(ADC_CH_TEMP, 2048U);

    /* Execute sensor acquisition */
    Sensor_MainFunction();

    /* Verify filter receives and processes sensor output */
    Filter_MainFunction();
    float result = Filter_GetOutput();

    TEST_ASSERT_FLOAT_WITHIN(0.5f, 25.0f, result);

    /* Test error propagation */
    MockHal_SetAdcValue(ADC_CH_TEMP, ADC_FAULT_VALUE);
    Sensor_MainFunction();
    Filter_MainFunction();
    TEST_ASSERT_EQUAL(SIGNAL_INVALID, Filter_GetQuality());
}
```

Integration tests should cover nominal data flow, error propagation, timing interactions, and initialization sequencing between modules.

Reference: ISO 26262 Part 6, Clause 9 — Software integration and integration testing
