---
title: Reproducible Builds
impact: MEDIUM
impactDescription: Ensures identical binary output for ISO 26262 traceability
tags: build, reproducible, traceability, iso-26262, audit
---

## Reproducible Builds

Ensure identical binary output from identical source for traceability and audit. ISO 26262 requires the ability to reproduce any released software version exactly.

**Incorrect (non-reproducible build):**

```makefile
CFLAGS += -DBUILD_DATE=\"$(shell date)\"
CFLAGS += -DBUILD_USER=\"$(USER)\"
```

**Correct (reproducible build with pinned metadata):**

```makefile
CFLAGS += -DBUILD_VERSION=\"$(GIT_HASH)\"
CFLAGS += -frandom-seed=$(GIT_HASH)
CFLAGS += -ffile-prefix-map=$(PWD)=.

export SOURCE_DATE_EPOCH=$(shell git log -1 --format=%ct)
```

Key practices for reproducible automotive builds:
- Pin all tool versions (compiler, linker, assembler) in version control
- Use `SOURCE_DATE_EPOCH` to eliminate timestamp variation
- Use `-ffile-prefix-map` to remove absolute paths from debug info
- Store the exact toolchain version in build artifacts for audit

Reference: ISO 26262 Part 8 — Configuration management
