---
title: Polyspace Bug Finder and Code Prover for Automotive
impact: HIGH
impactDescription: Polyspace Code Prover formally proves absence of runtime errors, providing ISO 26262 certification evidence that no other tool can match
tags: analysis, polyspace, mathworks, formal-verification, iso-26262, runtime-errors, code-prover, bug-finder
---

## Polyspace Bug Finder and Code Prover for Automotive

Polyspace (MathWorks) provides two tools: Bug Finder (fast defect detection) and Code Prover (formal verification proving absence of runtime errors). Code Prover produces green/orange/red classifications that serve as ISO 26262 certification evidence.

**Incorrect (ignoring orange results, no project configuration):**

```xml
<!-- polyspace_project.psprj — misconfigured -->
<!-- Missing target settings, no MISRA checking, wrong language standard -->
<project>
  <source>src/*.c</source>
  <!-- No target CPU specified — assumes host semantics -->
  <!-- No MISRA rules enabled -->
  <!-- Orange results ignored in CI — defeats formal verification value -->
</project>
```

**Correct (automotive Polyspace project options):**

```matlab
% polyspace_config.m — Polyspace configuration for automotive ECU
% Run: polyspace-configure -config polyspace_config.m

% Target configuration matching actual hardware
opts = polyspace.Options;
opts.TargetProcessor = 'cortex-m4';
opts.TargetOperatingSystem = 'no-os';
opts.TargetWordSize = 32;
opts.TargetSignedIntSize = 32;
opts.TargetCharSignedness = 'signed';
opts.TargetEndianness = 'little';

% Language and compilation
opts.CStandard = 'c11';
opts.Compiler = 'gnu4.9';  % Match cross-compiler semantics
opts.DefinedMacros = {'__arm__', 'PLATFORM_ECU', 'NDEBUG'};
opts.IncludeFolders = {'include/', 'bsw/include/', 'mcal/include/'};

% MISRA C:2012 checking
opts.MisraC2012 = 'required-mandatory';  % Check mandatory + required rules
opts.EnableMisraC2012Amendments = true;

% Code Prover verification depth
opts.VerificationDepth = 'full';  % Exhaustive path analysis
opts.Timeout = 3600;              % Per-function timeout in seconds

% Runtime checks to prove
opts.CheckOverflow = true;
opts.CheckDivisionByZero = true;
opts.CheckArrayBounds = true;
opts.CheckNullPointer = true;
opts.CheckUninitializedVariables = true;
opts.CheckShiftOperations = true;
```

**Interpreting Code Prover results for ISO 26262:**

```c
/* Example function with Code Prover color annotations */
int32_t Compute_Torque(int32_t rpm, int32_t load)
{
    /* GREEN — Polyspace PROVED no overflow is possible here
     * (all input ranges verified) */
    int32_t base_torque = rpm * TORQUE_CONSTANT;

    /* GREEN — division by zero proved impossible
     * (load range constrained by caller) */
    int32_t adjusted = base_torque / (load + 1);

    /* RED — Polyspace PROVED overflow WILL occur for some inputs.
     * THIS MUST BE FIXED before certification. */
    int32_t result = adjusted * AMPLIFICATION_FACTOR;  /* FIX REQUIRED */

    /* ORANGE — Polyspace CANNOT PROVE safety.
     * Requires either:
     *   (a) Tighten input ranges via precondition stubs, or
     *   (b) Add runtime defensive check, or
     *   (c) Justify as acceptable with documented rationale */
    int32_t limited = Saturate_I32(result, MIN_TORQUE, MAX_TORQUE);

    return limited;  /* GREEN after Saturate — range now bounded */
}
```

**Correct (handling orange results systematically):**

```c
/* BEFORE: orange result on division — Polyspace cannot prove
 * divisor is non-zero */
int32_t Calculate_Average(const int32_t *data, uint32_t count)
{
    int32_t sum = 0;
    for (uint32_t i = 0U; i < count; i++)
    {
        sum += data[i];
    }
    return sum / (int32_t)count;  /* ORANGE: potential division by zero */
}

/* AFTER: defensive check turns orange to green */
int32_t Calculate_Average(const int32_t *data, uint32_t count)
{
    if ((data == NULL) || (count == 0U))
    {
        return 0;  /* GREEN: null/zero case handled */
    }

    int32_t sum = 0;
    for (uint32_t i = 0U; i < count; i++)
    {
        sum += data[i];
    }
    return sum / (int32_t)count;  /* GREEN: count > 0 proved */
}
```

**Result classification for certification evidence:**

| Color | Meaning | ISO 26262 Action |
|-------|---------|------------------|
| **Green** | Proved safe — no runtime error possible | Certification evidence (no action needed) |
| **Red** | Proved defective — error will occur | Must fix before release |
| **Orange** | Unresolvable — safety not provable | Review and justify, add preconditions, or fix |
| **Gray** | Dead code — unreachable | Review: intentional or defect? |

Reference: ISO 26262-6 §9 Table 7 (formal verification for unit verification), ISO 26262-8 §11 (qualification of software tools)
