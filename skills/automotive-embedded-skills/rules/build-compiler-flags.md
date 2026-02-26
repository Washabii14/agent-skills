---
title: Safety-Optimized Compiler Flags
impact: MEDIUM
impactDescription: Balances safety and performance in compiler optimization
tags: build, compiler-flags, optimization, safety, iso-26262
---

## Safety-Optimized Compiler Flags

For safety-critical code, avoid aggressive optimization that may reorder or eliminate safety checks. The compiler's optimizer may remove null-pointer checks or reorder memory accesses in ways that break safety invariants.

**Incorrect (aggressive optimization on safety code):**

```makefile
CFLAGS += -O3 -ffast-math
```

**Correct (differentiated optimization by safety level):**

```makefile
# Safety-critical modules: limited optimization
CFLAGS_SAFETY += -O1 -fno-strict-aliasing -fno-delete-null-pointer-checks

# Non-safety modules: optimize for size
CFLAGS_NORMAL += -Os
```

`-fno-strict-aliasing` prevents optimizations based on type-based alias analysis which can break hardware register access patterns. `-fno-delete-null-pointer-checks` ensures defensive NULL checks are not optimized away.

Reference: ISO 26262 Part 6 — Software unit design and implementation
