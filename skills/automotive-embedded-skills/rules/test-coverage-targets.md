---
title: Coverage Targets per ASIL Level
impact: HIGH
impactDescription: ISO 26262 mandated coverage requirements by ASIL level
tags: test, coverage, asil, iso-26262, mc-dc, verification
---

## Coverage Targets per ASIL Level

ISO 26262 defines coverage targets that vary by ASIL level. Higher ASIL levels require more rigorous structural coverage metrics.

| ASIL Level | Statement Coverage | Branch Coverage | MC/DC |
|------------|-------------------|-----------------|-------|
| QM         | Recommended       | —               | —     |
| ASIL A     | Required          | Recommended     | —     |
| ASIL B     | Required          | Required        | —     |
| ASIL C     | Required          | Required        | Recommended |
| ASIL D     | Required          | Required        | Required    |

**Incorrect (no coverage measurement):**

```makefile
test:
	$(CC) -o test_runner $(TEST_SRCS) $(SRCS)
	./test_runner
```

**Correct (coverage instrumentation and enforcement):**

```makefile
test:
	$(CC) --coverage -o test_runner $(TEST_SRCS) $(SRCS)
	./test_runner
	gcov $(SRCS)
	lcov --capture --directory . --output-file coverage.info
	genhtml coverage.info --output-directory coverage_report
```

MC/DC (Modified Condition/Decision Coverage) requires that each condition in a decision independently affects the decision outcome. This is the most stringent coverage metric and is mandatory for ASIL D.

Reference: ISO 26262 Part 6, Table 12 — Structural coverage metrics at the software unit level
