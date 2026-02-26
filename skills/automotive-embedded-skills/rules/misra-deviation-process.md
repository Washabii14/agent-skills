---
title: MISRA Deviation Process — Documentation, Approval, Common Deviations
impact: MEDIUM
impactDescription: establishes auditable deviation workflow required for ISO 26262 compliance
tags: misra, deviation, compliance, iso-26262, audit, documentation, process, safety
---

## MISRA Deviation Process — Documentation, Approval, Common Deviations

MISRA compliance does not mean zero deviations — it means every deviation is documented, justified, risk-assessed, and approved. A well-managed deviation process is essential for ISO 26262 certification and auditor acceptance.

### MISRA Compliance Categories

| Category | Meaning | Deviation Permitted? |
|----------|---------|---------------------|
| Mandatory | Violation is always undefined/critical behavior | No — cannot be deviated |
| Required | Violation may cause serious issues | Yes — with formal deviation record |
| Advisory | Best practice recommendation | Yes — with project-level decision |

A project declares compliance at one of three levels:
- **Compliant**: All rules followed, no deviations
- **Deviations**: All rules followed except documented deviations
- **Advisory deviations**: Required rules followed; advisory rules may be deviated with project policy

---

### Deviation Record Format

Every deviation must be recorded in a structured format traceable by auditors.

```
╔══════════════════════════════════════════════════════════════╗
║ MISRA DEVIATION RECORD                                       ║
╠══════════════════════════════════════════════════════════════╣
║ Deviation ID:     DEV-<CATEGORY>-<NNN>                       ║
║ MISRA Rule:       MISRA C:2012 Rule X.Y / Dir X.Y            ║
║ Rule Category:    Required / Advisory                        ║
║ Rule Text:        [exact rule text from MISRA document]      ║
║                                                              ║
║ Scope:            [where deviation applies]                  ║
║                   e.g., "HAL driver layer" or "module X"     ║
║                                                              ║
║ Justification:    [technical reason why rule cannot be       ║
║                   followed in this context]                  ║
║                                                              ║
║ Risk Assessment:  [what risk does the deviation introduce    ║
║                   and how is it mitigated]                   ║
║                                                              ║
║ Mitigation:       [specific measures to ensure safety]       ║
║                   e.g., "Code review + unit test coverage"   ║
║                                                              ║
║ Conditions:       [constraints on when deviation is allowed] ║
║                   e.g., "Only in ISR handlers"               ║
║                                                              ║
║ ASIL Impact:      [which ASIL level is affected]             ║
║                   e.g., "ASIL B — mitigated to acceptable"   ║
║                                                              ║
║ Approved by:      [Safety Manager name + date]               ║
║ Review Date:      [next review date]                         ║
║ Ticket/CR:        [change request reference]                 ║
╚══════════════════════════════════════════════════════════════╝
```

---

### Approval Workflow

```
1. Developer identifies need for deviation
   └─► Documents technical justification

2. Developer creates Deviation Record
   └─► Fills template with all fields

3. Technical Lead reviews
   └─► Verifies justification is sound
   └─► Confirms scope is minimal
   └─► Checks mitigation is adequate

4. Safety Manager approves/rejects
   └─► Assesses against ISO 26262 requirements
   └─► Verifies ASIL impact analysis
   └─► Signs off or requests changes

5. Configuration Management
   └─► Deviation record stored in project deviation database
   └─► Static analysis tool configured to suppress with deviation ID
   └─► Traceability link to requirement/CR established

6. Periodic Review (every 6-12 months)
   └─► Reassess if deviation is still necessary
   └─► Check if language/tool evolution eliminates the need
   └─► Update or retire deviation as needed
```

---

### Static Analysis Tool Configuration

Deviations must be annotated in source code with tool-specific suppression markers:

```c
/* Polyspace suppression */
/* polyspace<MISRA-C:11.3:Not a defect:Justify with DEV-TYPE-001> Hardware register cast */

/* PC-lint/FlexeLint suppression */
/*lint -e{9078} DEV-TYPE-001: hardware register cast */

/* PRQA/Helix QAC suppression */
/* PRQA S 0303 ++ DEV-TYPE-001: hardware register cast */

/* Parasoft suppression */
/* parasoft-suppress MISRA2012-RULE-11.3 DEV-TYPE-001 */

/* Coverity suppression */
/* coverity[misra_c_2012_rule_11_3_violation] DEV-TYPE-001 */
```

Pattern for source code annotation:

```c
/* DEV-TYPE-001: MISRA C:2012 Rule 11.3 — hardware register access requires
 * pointer cast. Register address from STM32F4xx reference manual RM0090 Table 1.
 * Alignment verified: GPIO registers are 32-bit aligned at 0x400200xx.
 */
#define GPIOA  ((volatile GPIO_TypeDef *)0x40020000UL)  /* NOLINT(misra-c2012-11.3) */
```

---

### Common Acceptable Deviations Across Automotive Projects

**1. Hardware Register Access (Rule 11.3, 11.4)**

```
Pattern:         Cast integer constant to volatile pointer for MMIO
Prevalence:      Universal — every embedded project needs this
Risk:            Low — addresses are compile-time constants from verified vendor headers
Typical scope:   HAL/MCAL layer only
```

**2. Goto for Error Cleanup (Rule 15.1)**

```
Pattern:         Forward-only goto to single cleanup label
Prevalence:      Very common in C projects
Risk:            Low — pattern is well-understood and prevents resource leaks
Typical scope:   Functions acquiring multiple resources in C (C++ uses RAII)
```

**3. Union for Protocol Messages (Rule 19.2)**

```
Pattern:         Discriminated union for CAN/UDS/SOME-IP message variants
Prevalence:      Common in communication stacks
Risk:            Medium — mitigated by always checking discriminant before access
Typical scope:   Communication middleware
```

**4. Multiple Returns in Validation (Rule 15.5)**

```
Pattern:         Early return on null/error in guard clauses
Prevalence:      Common in QM-rated utility code
Risk:            Low — limited to early guards, not scattered through function
Typical scope:   ASIL QM modules only, max 3 early returns
```

**5. Partial Array Initialization (Rule 9.3)**

```
Pattern:         Large lookup tables with sparse non-zero entries
Prevalence:      Common — CRC tables, character classification tables
Risk:            Low — C guarantees zero-initialization of unspecified elements
Typical scope:   const lookup tables with documented intent
```

**6. Banned Functions in Debug Builds (Rule 21.6)**

```
Pattern:         printf/sprintf for development diagnostics
Prevalence:      Universal during development
Risk:            None — removed in production via compile switch
Typical scope:   #ifdef DEBUG_BUILD guarded code
```

**7. Inline Assembly for ISR/Barrier (Rule — language extension)**

```
Pattern:         __asm volatile for DMB/DSB/ISB and interrupt enable/disable
Prevalence:      Universal on ARM Cortex targets
Risk:            Low — well-documented, compiler-specific, hardware-required
Typical scope:   RTOS port layer and HAL interrupt management
```

---

### Anti-Patterns in Deviation Management

**Do NOT:**

- Blanket-suppress entire rule categories across the project
- Use "performance" as sole justification without measurement data
- Approve deviations for Required rules without Safety Manager sign-off
- Allow deviation scope to be "entire project" — always scope to specific modules
- Forget to configure the static analysis tool — unannotated suppressions are invisible to auditors
- Copy deviation justifications from other projects without reviewing applicability

**DO:**

- Keep a centralized deviation database (spreadsheet, database, or docs-as-code)
- Cross-reference deviations in the safety case documentation
- Review all deviations at each safety assessment milestone
- Track deviation count per module as a quality metric (high count = design smell)
- Retire deviations when the root cause is resolved (e.g., migrating from union to memcpy)

---

### Deviation Count Targets (Guidelines)

| Module Type | Max Required Rule Deviations | Max Advisory Deviations |
|-------------|------------------------------|------------------------|
| ASIL D safety-critical | 0–2 per module | 5–10 per module |
| ASIL B functional | 2–5 per module | 10–20 per module |
| ASIL QM non-safety | 5–10 per module | No hard limit (track trend) |
| HAL/MCAL drivers | 5–15 per module (hardware access) | No hard limit |

These are guidelines — the actual limits should be agreed with the safety assessor.

Reference: MISRA C:2012 Appendix A (Deviation procedure); MISRA Compliance:2020; ISO 26262-6:2018 clause 8
