---
title: PC-lint / PC-lint Plus Configuration for Automotive
impact: HIGH
impactDescription: PC-lint is the most widely deployed MISRA checker in automotive — proper configuration eliminates false positives and integrates cleanly into CI/CD pipelines
tags: analysis, pclint, misra, automotive, lint, ci-cd, false-positive, safety
---

## PC-lint / PC-lint Plus Configuration for Automotive

PC-lint Plus (Gimpel) is the industry-standard MISRA compliance checker for automotive C/C++. Correct configuration requires a policy file, MISRA rule selection, compiler-specific headers, and disciplined false positive suppression.

**Incorrect (running PC-lint with no configuration):**

```bash
# No configuration — hundreds of false positives from system headers
pclp64 src/app.c
# Result: 2000+ messages, mostly noise from toolchain headers
# Developers ignore all output, defeating the purpose
```

**Correct (automotive lint policy file):**

```
// automotive.lnt — master lint configuration for automotive project
// Include compiler-specific configuration
co-gcc.lnt           // GCC built-in types and predefined macros
// or: co-armcc.lnt  // for ARM Compiler
// or: co-ghs.lnt    // for GreenHills

// MISRA C:2012 compliance
au-misra3.lnt        // MISRA C:2012 base rules
au-misra3-amd1.lnt   // Amendment 1 rules
au-misra3-amd2.lnt   // Amendment 2 rules

// Project include paths
-i"include/"
-i"bsw/include/"
-i"mcal/include/"
-i"rtos/include/"

// Suppress noise from vendor/SDK headers
-wlib(0)             // No warnings from library headers
+libdir("vendor/")
+libdir("sdk/")
+libdir("rtos/include/")

// Size and type model for target (e.g., 32-bit ARM)
-si4                  // sizeof(int) = 4
-sl4                  // sizeof(long) = 4
-sp4                  // sizeof(pointer) = 4
-ss2                  // sizeof(short) = 2

// Automotive-critical message escalation
+e900                 // Successful completion message
-w3                   // Warning level 3 (strict)
+elib(900)

// Enable specific high-value checks
+e438                 // Last value assigned to variable not used
+e529                 // Symbol not subsequently referenced
+e534                 // Ignoring return value of function
+e818                 // Pointer parameter could be const
+e835                 // Zero used as argument to operator
```

**Incorrect (suppressing lint warnings with bare comments):**

```c
int32_t Read_Sensor(void)
{
    int32_t value;
    (void)HAL_ADC_Read(&value);  /* suppress warning somehow */
    return value;
    /* No traceability for why warning was suppressed */
}
```

**Correct (structured suppression with justification):**

```c
int32_t Read_Sensor(void)
{
    int32_t value;
    /*lint -e{534} HAL_ADC_Read return value checked via value param;
     * error state handled by watchdog timeout.
     * Approved: Safety Review SR-2024-042 */
    (void)HAL_ADC_Read(&value);
    return value;
}

/* File-level suppression for hardware register access pattern */
/*lint -save */
/*lint -e923 Cast from pointer to unsigned int [MISRA 2012 Rule 11.6]
 * Required for memory-mapped register access per MCAL specification.
 * Approved: MISRA Deviation DR-0017 */
#define REG32(addr) (*(volatile uint32_t *)(uintptr_t)(addr))
/*lint -restore */
```

**CI/CD integration (GitLab CI example):**

```yaml
# .gitlab-ci.yml — PC-lint stage
lint:pclint:
  stage: static_analysis
  script:
    - pclp64 -b automotive.lnt
        -max_threads=4
        -summary
        -"format=%f(%l): %t %n: %m"
        -oo"build/pclint_report.txt"
        -os"build/pclint_summary.txt"
        src/*.c src/*.h
    - python scripts/lint_check.py build/pclint_report.txt --max-errors=0
  artifacts:
    paths:
      - build/pclint_report.txt
      - build/pclint_summary.txt
    when: always
  rules:
    - if: $CI_MERGE_REQUEST_ID
```

Reference: MISRA C:2012 (compliance framework, deviation records), ISO 26262-6 §9 Table 7 (static analysis for unit verification)
