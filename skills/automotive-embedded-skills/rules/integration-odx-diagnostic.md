---
title: ODX/PDX Diagnostic Descriptions
impact: MEDIUM
impactDescription: Ensures diagnostic tool compatibility via standardized descriptions
tags: integration, odx, pdx, diagnostics, ediabas, d-pdu, iso-22901
---

## ODX/PDX Diagnostic Descriptions

Structure ODX diagnostic data correctly for interoperability with diagnostic tools (EDIABAS, D-PDU API). ODX (Open Diagnostic Data Exchange) describes diagnostic services, DIDs, DTCs, and communication parameters in a tool-independent format.

**Incorrect (hardcoded diagnostic definitions):**

```c
#define DID_VIN          0xF190U
#define DID_VIN_LENGTH   17U
```

**Correct (ODX-aligned diagnostic structure):**

```c
typedef struct
{
    uint16_t    didId;
    uint8_t     dataLength;
    const char *shortName;
    const char *longName;
    Std_ReturnType (*readFunc)(uint8_t *data, uint16_t *len);
    Std_ReturnType (*writeFunc)(const uint8_t *data, uint16_t len);
} DiagDid_t;

static const DiagDid_t g_didTable[] =
{
    { 0xF190U, 17U, "VIN", "Vehicle Identification Number",
      DID_ReadVin, NULL },
    { 0xF187U, 16U, "SparePartNumber", "ECU Spare Part Number",
      DID_ReadSparePartNumber, NULL },
};
```

Keep ODX/PDX files synchronized with source code DID tables. Export DID lists from source to ODX format as part of the build process to prevent mismatch.

Reference: ASAM MCD-2D (ODX) — Open Diagnostic Data Exchange, ISO 22901
