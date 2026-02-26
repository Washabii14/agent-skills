---
title: Compiler Warnings as Errors
impact: HIGH
impactDescription: Catches bugs at compile time by treating warnings as errors
tags: build, warnings, compiler, quality, ci
---

## Compiler Warnings as Errors

Enable comprehensive warnings and promote them to errors. A warning-free build is a baseline quality gate for safety-critical automotive code.

**Incorrect (minimal warnings, warnings ignored):**

```makefile
CFLAGS += -Wall
```

**Correct (comprehensive warnings as errors):**

```makefile
CFLAGS += -Wall -Wextra -Werror -Wpedantic
CFLAGS += -Wconversion -Wsign-conversion
CFLAGS += -Wcast-align -Wcast-qual
CFLAGS += -Wdouble-promotion -Wfloat-equal
CFLAGS += -Wshadow -Wstrict-prototypes
```

`-Wconversion` and `-Wsign-conversion` are particularly important for MISRA compliance as they catch implicit narrowing and signed/unsigned mixing at compile time.
