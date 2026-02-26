---
title: Use [[nodiscard]] for Important Return Values
impact: MEDIUM
impactDescription: prevents ignoring error codes
tags: autosar, cpp, nodiscard, error-handling, return-value, safety
---

## Use [[nodiscard]] for Important Return Values

Apply `[[nodiscard]]` to functions whose return value must not be silently ignored, especially error codes and status returns.

**Incorrect (error code silently ignored):**

```cpp
Std_ReturnType WriteEeprom(uint16_t addr, const uint8_t* data, uint16_t len);

void SaveConfig()
{
    WriteEeprom(0x100, configData, sizeof(configData));  /* Return value ignored */
}
```

**Correct ([[nodiscard]] enforced):**

```cpp
[[nodiscard]] Std_ReturnType WriteEeprom(uint16_t addr, const uint8_t* data,
                                          uint16_t len);

void SaveConfig()
{
    Std_ReturnType ret = WriteEeprom(0x100, configData, sizeof(configData));
    if (ret != E_OK)
    {
        ReportError(ERR_EEPROM_WRITE);
    }
}
```

Reference: AUTOSAR C++14 Rule A0-1-2
