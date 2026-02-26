---
title: CRC Validation for Critical Data
impact: HIGH
impactDescription: detects data corruption in RAM or transit
tags: safety, iso-26262, crc, data-integrity, corruption-detection
---

## CRC Validation for Critical Data

Protect critical data stored in RAM or transmitted over communication with CRC checksums. Compute CRC over the data portion and store/send it alongside the payload. Re-validate before use to detect silent corruption.

**Incorrect (unprotected safety-relevant data in RAM):**

```c
typedef struct
{
    float   safetyRelevantParam;
    uint8_t controlFlags;
} UnprotectedData_t;
```

**Correct (CRC-protected critical data):**

```c
typedef struct
{
    float   safetyRelevantParam;
    uint8_t controlFlags;
    uint32_t crc;
} ProtectedData_t;

Std_ReturnType WriteProtectedData(ProtectedData_t *data, float param,
                                   uint8_t flags)
{
    data->safetyRelevantParam = param;
    data->controlFlags = flags;
    data->crc = Crc_CalculateCRC32(
        (const uint8_t *)data,
        offsetof(ProtectedData_t, crc),
        CRC_INITIAL_VALUE);
    return E_OK;
}

boolean ValidateProtectedData(const ProtectedData_t *data)
{
    uint32_t computed = Crc_CalculateCRC32(
        (const uint8_t *)data,
        offsetof(ProtectedData_t, crc),
        CRC_INITIAL_VALUE);
    return (computed == data->crc);
}
```

Always validate CRC before reading safety-relevant parameters. Use AUTOSAR CRC library (Crc_CalculateCRC32) for standardized computation.

Reference: ISO 26262 Part 6 — Data integrity measures; AUTOSAR SWS CRC Library
