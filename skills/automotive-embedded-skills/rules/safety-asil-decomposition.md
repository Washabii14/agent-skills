---
title: ASIL Decomposition Patterns
impact: HIGH
impactDescription: enables mixed-criticality systems
tags: safety, iso-26262, asil, decomposition, memory-partitioning, freedom-from-interference
---

## ASIL Decomposition Patterns

When decomposing ASIL requirements across software partitions, ensure freedom from interference between partitions. Use memory protection (MPU), separate linker sections, and partition-aware OS mechanisms to prevent lower-ASIL or QM code from corrupting higher-ASIL data.

**Incorrect (mixed criticality without isolation):**

```c
static BrakeControl_t g_brakeData;        /* ASIL D */
static InfotainmentData_t g_infoData;     /* QM */
/* Both in same memory region — no isolation */
```

**Correct (memory-partitioned by ASIL level):**

```c
/* ASIL D partition — protected memory region */
#pragma section ".ASIL_D_DATA"
static BrakeControl_t g_brakeData;
#pragma section

/* QM partition — separate memory region */
#pragma section ".QM_DATA"
static InfotainmentData_t g_infoData;
#pragma section
```

Key practices for ASIL decomposition:
- Use MPU (Memory Protection Unit) to enforce partition boundaries
- Assign each partition its own linker section mapped to a protected memory region
- Use OS trusted functions for cross-partition communication
- Never share writable data directly between partitions of different ASIL levels
- Document the decomposition rationale in the safety plan

Reference: ISO 26262 Part 6, Table 5 — Methods for software architectural design
