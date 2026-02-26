---
title: Link-Time Optimization
impact: LOW
impactDescription: Cross-module optimization to reduce code size
tags: build, lto, linker, optimization, code-size
---

## Link-Time Optimization

Use LTO for non-safety-critical modules to reduce code size. LTO enables the linker to perform cross-module inlining, dead code elimination, and constant propagation that the compiler cannot do at single-translation-unit scope.

**Incorrect (LTO on safety-critical code):**

```makefile
CFLAGS += -flto
LDFLAGS += -flto
```

**Correct (LTO only for non-safety modules):**

```makefile
# Non-safety modules: enable LTO
CFLAGS_QM  += -flto
LDFLAGS_QM += -flto

# Safety modules: no LTO (preserves analyzable object code)
CFLAGS_ASIL  += -O1
LDFLAGS_ASIL +=
```

LTO changes the object code in ways that make it harder to trace back to source for ISO 26262 verification. Use it only where traceability requirements are relaxed (QM-level code, infotainment, logging).
