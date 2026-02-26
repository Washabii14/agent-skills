---
title: Static Analysis Integration
impact: HIGH
impactDescription: Finds bugs without execution via automated static analysis
tags: build, static-analysis, misra, ci, quality, polyspace, coverity
---

## Static Analysis Integration

Integrate static analysis tools into CI/CD: PC-lint, Polyspace, Coverity, cppcheck. Static analysis detects defects (null dereference, buffer overflow, dead code, MISRA violations) without executing the code.

**Incorrect (no static analysis in pipeline):**

```makefile
build:
	$(CC) $(CFLAGS) -o $(TARGET) $(SRCS)
```

**Correct (static analysis gate before build):**

```makefile
analyze:
	cppcheck --enable=all --error-exitcode=1 --suppress=missingInclude $(SRCS)
	$(PCLINT) $(LINT_OPTS) $(SRCS)

build: analyze
	$(CC) $(CFLAGS) -o $(TARGET) $(SRCS)
```

For ISO 26262 compliance, static analysis is required at ASIL A and above. Use MISRA-capable tools (PC-lint Plus, Polyspace Bug Finder, Coverity) to check rule compliance and report deviations.

Reference: ISO 26262 Part 6, Table 7 — Methods for software unit verification
