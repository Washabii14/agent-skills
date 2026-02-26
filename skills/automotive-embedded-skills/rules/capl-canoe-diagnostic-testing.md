---
title: Diagnostic Testing Patterns
impact: HIGH
impactDescription: Proper diagnostic test patterns ensure UDS service compliance and catch NRC handling bugs
tags: capl, diagnostics, uds, testing, diag-request, nrc, canoe
---

## Diagnostic Testing Patterns

Implement diagnostic test patterns for UDS services with proper request/response validation. Always check for positive response, negative response codes (NRCs), and timeout conditions.

**Incorrect (no response validation):**

```capl
testcase TC_ReadDID_Bad()
{
    diagRequest ECU1.VehicleIdentification_Read req;
    diagSendRequest(req);
    testWaitForTimeout(2000);
    /* No check if response was positive or negative */
}
```

**Correct (full request/response validation with NRC check):**

```capl
testcase TC_ReadDID_F190()
{
    diagRequest ECU1.VehicleIdentification_Read req;
    diagResponse ECU1.VehicleIdentification_Read resp;

    testStep("Request", "Send ReadDataByIdentifier for VIN");
    diagSendRequest(req);

    if (testWaitForDiagResponse(resp, 2000) != 0)
    {
        testStepFail("Response", "No response received within 2s");
        return;
    }

    if (diagGetLastResponseCode(resp) != 0)  /* 0 = positive response */
    {
        long nrc = diagGetLastResponseCode(resp);
        testStepFail("Response", "Negative response: NRC 0x%02X", nrc);
        return;
    }

    testStepPass("Response", "VIN read successfully");
}
```

Check `diagGetLastResponseCode()` to distinguish positive from negative responses. Log the NRC value on failure for diagnostics. Set timeouts based on the P2 server timing parameter from the ECU specification.

Reference: ISO 14229-1:2020 (UDS), Vector CANoe Diagnostic/ISO TP Functions
