---
title: Explicit Boolean Comparisons
impact: LOW
impactDescription: improves readability and prevents implicit conversion bugs
tags: misra, boolean, comparison, readability, type-safety, explicit
---

## Explicit Boolean Comparisons

Use explicit boolean comparisons rather than relying on implicit truthiness. This eliminates ambiguity and prevents bugs from implicit integer-to-boolean conversions.

**Incorrect (implicit boolean):**

```c
if (ptr)           { /* ... */ }
if (!errorCount)   { /* ... */ }
if (flags & MASK)  { /* ... */ }
```

**Correct (explicit comparisons):**

```c
if (ptr != NULL)          { /* ... */ }
if (errorCount == 0U)     { /* ... */ }
if ((flags & MASK) != 0U) { /* ... */ }
```

Reference: MISRA C:2012 Rule 14.4 — The controlling expression of an if/while shall have essentially Boolean type.
