---
title: cppcheck Configuration for Embedded Projects
impact: MEDIUM
impactDescription: cppcheck is a free, open-source analyzer that integrates easily into CI/CD and provides MISRA checking via its addon — ideal for supplementing commercial tools
tags: analysis, cppcheck, open-source, misra, ci-cd, embedded, suppressions, custom-rules
---

## cppcheck Configuration for Embedded Projects

cppcheck is an open-source static analyzer with a MISRA C:2012 addon. While not a replacement for qualified tools, it provides fast feedback in CI/CD pipelines and catches undefined behavior, buffer overflows, and style issues at zero cost.

**Incorrect (running cppcheck with no project configuration):**

```bash
# No platform, no MISRA, default checks only
cppcheck src/
# Produces minimal results, misses embedded-specific issues,
# false positives on compiler intrinsics
```

**Correct (embedded project configuration):**

```xml
<!-- cppcheck_project.cppcheck — project configuration -->
<project version="1">
    <paths>
        <dir name="src/"/>
        <dir name="app/"/>
        <dir name="bsw/"/>
    </paths>
    <includes>
        <include directory="include/"/>
        <include directory="mcal/include/"/>
        <include directory="rtos/include/"/>
    </includes>
    <defines>
        <define name="__arm__"/>
        <define name="__GNUC__=10"/>
        <define name="PLATFORM_ECU"/>
    </defines>
    <excludes>
        <exclude directory="vendor/"/>
        <exclude directory="generated/"/>
        <exclude directory="test/"/>
    </excludes>
    <platform>arm32-wchar_t2</platform>
    <analyze-all-vs-configs>false</analyze-all-vs-configs>
</project>
```

**Incorrect (no MISRA addon, no suppressions file):**

```bash
cppcheck --enable=all src/
# Floods output with style warnings and missing-include noise
# No MISRA checking performed
```

**Correct (MISRA addon with suppressions):**

```bash
#!/bin/bash
# run_cppcheck.sh — automotive CI configuration

cppcheck \
    --project=cppcheck_project.cppcheck \
    --enable=warning,performance,portability,unusedFunction \
    --std=c11 \
    --platform=arm32-wchar_t2 \
    --inline-suppr \
    --suppressions-list=cppcheck_suppressions.txt \
    --addon=misra.py \
    --addon=cert.py \
    --error-exitcode=1 \
    --xml \
    --output-file=build/cppcheck_report.xml \
    2>&1 | tee build/cppcheck_log.txt

# Convert XML report to HTML for review
cppcheck-htmlreport \
    --file=build/cppcheck_report.xml \
    --report-dir=build/cppcheck_html \
    --source-dir=.
```

**Suppressions file for embedded-specific false positives:**

```
// cppcheck_suppressions.txt

// Hardware register access patterns (memory-mapped I/O)
memleak:mcal/src/*.c
nullPointerRedundantCheck:bsw/src/os_*.c

// Vendor-supplied code (not under project control)
*:vendor/*

// Intentional: RTOS task entry points are never called directly
unusedFunction:app/src/task_*.c

// Inline assembly blocks
syntaxError:mcal/src/cpu_*.c
```

**Custom rule example (enforce naming convention):**

```xml
<!-- custom_rules.xml — project-specific rules -->
<rules>
    <rule>
        <tokenlist>normal</tokenlist>
        <pattern>
            <token>typedef</token>
            <token type="type"/>
            <not><token pattern="_t$"/></not>
        </pattern>
        <message>
            <severity>style</severity>
            <summary>Typedef name must end with _t suffix</summary>
        </message>
    </rule>
</rules>
```

**CI/CD integration (GitHub Actions):**

```yaml
# .github/workflows/static_analysis.yml
static-analysis:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Install cppcheck
      run: sudo apt-get install -y cppcheck
    - name: Run cppcheck with MISRA
      run: |
        cppcheck --project=cppcheck_project.cppcheck \
            --enable=warning,performance,portability \
            --std=c11 \
            --addon=misra.py \
            --error-exitcode=1 \
            --xml --output-file=cppcheck_report.xml
    - name: Upload report
      uses: actions/upload-artifact@v4
      with:
        name: cppcheck-report
        path: cppcheck_report.xml
      if: always()
```

Reference: MISRA C:2012 Appendix A (tool qualification guidance), ISO 26262-8 §11 (qualification of software tools — TQL-5 for non-safety-qualified tools)
