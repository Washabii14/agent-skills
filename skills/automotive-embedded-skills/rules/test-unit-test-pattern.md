---
title: Unit Test Patterns for Embedded
impact: HIGH
impactDescription: Early defect detection through structured embedded unit tests
tags: test, unit-test, unity, embedded, verification
---

## Unit Test Patterns for Embedded

Structure unit tests for embedded C with proper hardware mocking. Use test frameworks like Unity or Google Test that support embedded targets with minimal runtime overhead.

**Correct (Unity test framework pattern):**

```c
/* Using Unity test framework */
void test_PlausibilityCheck_RejectsOutOfRange(void)
{
    PlausibilityCheck_t check = {
        .currentValue = 25.0f,
        .previousValue = 25.0f,
        .maxDeltaPerCycle = 5.0f,
        .minValid = -40.0f,
        .maxValid = 150.0f
    };

    TEST_ASSERT_FALSE(IsPlausible(&check, 200.0f));  /* Above max */
    TEST_ASSERT_FALSE(IsPlausible(&check, -50.0f));  /* Below min */
    TEST_ASSERT_TRUE(IsPlausible(&check, 26.0f));    /* Valid */
}
```

Each test should follow Arrange-Act-Assert: set up preconditions, invoke the function under test, and verify the result. Keep tests independent — no shared mutable state between test cases.

Reference: ISO 26262 Part 6, Table 7 — Methods for software unit verification
