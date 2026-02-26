---
title: Boundary Value Testing
impact: MEDIUM
impactDescription: Systematic defect detection at value boundaries
tags: test, boundary, values, systematic, verification
---

## Boundary Value Testing

Test at and around boundary values systematically: min, min-1, min+1, max, max-1, max+1, zero, and typical values. Most defects cluster at boundaries.

**Incorrect (only happy-path test):**

```c
void test_SpeedLimit(void)
{
    TEST_ASSERT_TRUE(IsSpeedValid(100U));
}
```

**Correct (boundary value analysis):**

```c
void test_SpeedLimit_BoundaryValues(void)
{
    /* At boundaries */
    TEST_ASSERT_TRUE(IsSpeedValid(0U));      /* min */
    TEST_ASSERT_TRUE(IsSpeedValid(250U));    /* max */

    /* Just inside boundaries */
    TEST_ASSERT_TRUE(IsSpeedValid(1U));      /* min + 1 */
    TEST_ASSERT_TRUE(IsSpeedValid(249U));    /* max - 1 */

    /* Just outside boundaries */
    TEST_ASSERT_FALSE(IsSpeedValid(251U));   /* max + 1 */

    /* Typical value */
    TEST_ASSERT_TRUE(IsSpeedValid(120U));

    /* Edge case: uint16 max */
    TEST_ASSERT_FALSE(IsSpeedValid(UINT16_MAX));
}
```

For ISO 26262 ASIL B and above, boundary value analysis is a required test method. Combine with equivalence partitioning for comprehensive input coverage.

Reference: ISO 26262 Part 6, Table 10 — Methods for software unit testing
