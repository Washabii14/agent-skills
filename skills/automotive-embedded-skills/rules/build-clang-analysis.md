---
title: Clang Static Analysis for Automotive
impact: HIGH
impactDescription: Clang-Tidy and Clang Static Analyzer detect complex defects including concurrency bugs, cert violations, and undefined behavior before code reaches hardware
tags: build, clang, static-analysis, clang-tidy, scan-build, misra, cert, automotive
---

## Clang Static Analysis for Automotive

Clang provides two complementary analysis tools: Clang-Tidy (linter with auto-fix) and Clang Static Analyzer (path-sensitive bug finder via `scan-build`). Both must be configured with automotive-relevant checker sets for embedded projects.

**Incorrect (no .clang-tidy config, running with defaults):**

```yaml
# .clang-tidy — empty or missing configuration
# Running: clang-tidy src/*.c
# Uses default checks only — misses CERT, bugprone, and readability issues
# No header filter — analyzes third-party/vendor headers wastefully
```

**Correct (.clang-tidy configured for automotive embedded):**

```yaml
# .clang-tidy — automotive embedded configuration
---
Checks: >
  -*,
  bugprone-*,
  cert-*,
  clang-analyzer-*,
  concurrency-*,
  misc-*,
  modernize-*,
  performance-*,
  readability-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers,
  -cert-dcl37-c,
  -cert-dcl51-cpp

# Only check project headers, not vendor/SDK
HeaderFilterRegex: '^(src|include|app|bsw)/.*'

WarningsAsErrors: >
  bugprone-use-after-move,
  bugprone-signed-char-misuse,
  bugprone-integer-division,
  bugprone-narrowing-conversions,
  cert-err33-c,
  cert-err34-c,
  cert-int31-c,
  cert-flp30-c,
  clang-analyzer-core.*,
  clang-analyzer-deadcode.*,
  clang-analyzer-security.*,
  concurrency-*

CheckOptions:
  - key: readability-identifier-naming.FunctionCase
    value: CamelCase
  - key: readability-identifier-naming.VariableCase
    value: lower_case
  - key: readability-identifier-naming.MacroDefinitionCase
    value: UPPER_CASE
  - key: readability-identifier-naming.TypedefCase
    value: CamelCase
  - key: readability-identifier-naming.TypedefSuffix
    value: '_t'
  - key: readability-function-size.LineThreshold
    value: '75'
  - key: readability-function-cognitive-complexity.Threshold
    value: '15'
  - key: bugprone-narrowing-conversions.WarnOnFloatingPointNarrowingConversion
    value: 'true'
  - key: misc-non-private-member-variables-in-classes.IgnoreClassesWithAllMemberVariablesBeingPublic
    value: 'true'
...
```

**Incorrect (no scan-build in CI):**

```bash
# Just compile, no path-sensitive analysis
arm-none-eabi-gcc -c src/app.c -o build/app.o
```

**Correct (scan-build integrated into build):**

```bash
# Run Clang Static Analyzer with automotive-relevant checkers
scan-build \
  --use-analyzer=/usr/bin/clang \
  -enable-checker alpha.core.BoolAssignment \
  -enable-checker alpha.core.CastSize \
  -enable-checker alpha.core.FixedAddr \
  -enable-checker alpha.security.ArrayBoundV2 \
  -enable-checker alpha.security.MallocOverflow \
  -enable-checker alpha.deadcode.UnreachableCode \
  -enable-checker core.NullDereference \
  -enable-checker core.DivideZero \
  -enable-checker core.UndefinedBinaryOperatorResult \
  -enable-checker core.uninitialized.Assign \
  -o build/scan-results \
  make all
```

**CMake integration:**

```cmake
# CMakeLists.txt — optional Clang-Tidy integration
option(ENABLE_CLANG_TIDY "Run clang-tidy during compilation" OFF)

if(ENABLE_CLANG_TIDY)
    find_program(CLANG_TIDY_EXE NAMES clang-tidy)
    if(CLANG_TIDY_EXE)
        set(CMAKE_C_CLANG_TIDY
            "${CLANG_TIDY_EXE}"
            "--config-file=${CMAKE_SOURCE_DIR}/.clang-tidy"
            "--extra-arg=-std=c11"
        )
    endif()
endif()
```

**Key automotive checker categories:**

| Category | Detects |
|----------|---------|
| `bugprone-*` | Common bug patterns (narrowing, signed char misuse) |
| `cert-*` | CERT C/C++ Secure Coding Standard violations |
| `clang-analyzer-core.*` | Null dereference, divide-by-zero, uninitialized reads |
| `clang-analyzer-security.*` | Buffer overflows, format string vulnerabilities |
| `concurrency-*` | Data races, lock order violations |
| `readability-function-size` | Enforces function length limits (MISRA advisory) |

Reference: MISRA C:2012 Directive 4.1 (tool-assisted verification), ISO 26262-6 §9 (software unit verification methods)
