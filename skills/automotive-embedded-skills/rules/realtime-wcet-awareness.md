---
title: Design with WCET in Mind
impact: HIGH
impactDescription: enables schedulability analysis
tags: realtime, wcet, timing-analysis, schedulability, worst-case
---

## Design with WCET in Mind

All code paths must have analyzable worst-case execution time (WCET) for schedulability proofs. Avoid constructs that make WCET analysis intractable: unbounded loops, recursion, dynamic dispatch with unknown targets, and data-dependent branching with vastly different path lengths.

**Incorrect (WCET unanalyzable):**

```c
void ProcessData(uint8_t *data, uint16_t len)
{
    for (uint16_t i = 0U; i < len; i++)  /* len is unbounded */
    {
        if (data[i] == SPECIAL_MARKER)
        {
            ExpandMarker(data, &i, &len);  /* Changes loop bounds */
        }
    }
}
```

**Correct (WCET-friendly bounded processing):**

```c
#define MAX_DATA_LEN (256U)

void ProcessData(uint8_t *data, uint16_t len)
{
    if (len > MAX_DATA_LEN) { len = MAX_DATA_LEN; }

    for (uint16_t i = 0U; i < len; i++)
    {
        ProcessByte(data[i]);
    }
}
```

Design guidelines for WCET analyzability:
- Use compile-time-known loop bounds or clamp to a maximum
- Avoid function pointers where possible; use switch-based dispatch
- Minimize branching asymmetry (balance if/else path lengths)
- Annotate WCET budgets in code comments for reviewer visibility

Reference: ISO 26262 Part 6 — Timing analysis; AUTOSAR OS timing protection
