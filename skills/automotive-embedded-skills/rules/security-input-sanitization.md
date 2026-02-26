---
title: Input Sanitization from External Interfaces
impact: HIGH
impactDescription: Prevents injection and buffer attacks from external data
tags: security, input, sanitization, validation, buffer, iso-21434
---

## Input Sanitization from External Interfaces

Sanitize and validate all data received from external interfaces (CAN, Ethernet, USB, diagnostic). External data is untrusted by definition and must be validated before processing to prevent buffer overflows, injection attacks, and data corruption.

**Incorrect (no validation of external input):**

```c
void ProcessDiagPayload(const uint8_t *input, uint16_t inputLen)
{
    uint8_t buffer[256];
    memcpy(buffer, input, inputLen);
    ExecuteCommand(buffer);
}
```

**Correct (validated and sanitized input):**

```c
Std_ReturnType Sanitize_DiagPayload(const uint8_t *input, uint16_t inputLen,
                                      uint8_t *output, uint16_t outputSize,
                                      uint16_t *outputLen)
{
    if ((input == NULL) || (output == NULL) || (outputLen == NULL))
    {
        return E_NOT_OK;
    }

    if (inputLen > outputSize)
    {
        ReportDtc(DTC_INPUT_OVERFLOW_ATTEMPT);
        return E_NOT_OK;
    }

    if (inputLen > MAX_DIAG_PAYLOAD_SIZE)
    {
        return E_NOT_OK;
    }

    for (uint16_t i = 0U; i < inputLen; i++)
    {
        if (!IsValidByte(input[i]))
        {
            ReportDtc(DTC_INVALID_INPUT_DATA);
            return E_NOT_OK;
        }
    }

    (void)memcpy(output, input, inputLen);
    *outputLen = inputLen;
    return E_OK;
}
```

Every external interface is an attack surface. Apply defense-in-depth: validate length, validate content, validate source, and enforce rate limiting.

Reference: ISO/SAE 21434:2021 — Road vehicles — Cybersecurity engineering
