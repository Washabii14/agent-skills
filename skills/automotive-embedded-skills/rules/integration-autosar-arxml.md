---
title: AUTOSAR ARXML Configuration
impact: HIGH
impactDescription: Ensures correct AUTOSAR toolchain integration via ARXML
tags: integration, autosar, arxml, bsw, swc, configuration, toolchain
---

## AUTOSAR ARXML Configuration

Generate and parse AUTOSAR ARXML configuration files correctly for BSW configuration, SWC descriptions, and ECU extracts. ARXML is the central exchange format for all AUTOSAR toolchains (Vector DaVinci, Elektrobit tresos, ETAS ISOLAR).

**Incorrect (manual BSW configuration without ARXML):**

```c
/* Hardcoded COM stack configuration */
#define COM_NUM_IPDU    (50U)
#define COM_TIMEOUT_MS  (100U)
```

**Correct (ARXML-driven configuration):**

```xml
<!-- ECU Extract — Com stack configuration -->
<ECUC-VALUE-COLLECTION>
  <ECUC-MODULE-CONFIGURATION-VALUES>
    <SHORT-NAME>Com</SHORT-NAME>
    <CONTAINERS>
      <ECUC-CONTAINER-VALUE>
        <SHORT-NAME>ComConfig</SHORT-NAME>
        <PARAMETER-VALUES>
          <ECUC-NUMERICAL-PARAM-VALUE>
            <DEFINITION-REF>/AUTOSAR/Com/ComConfig/ComNumIPdu</DEFINITION-REF>
            <VALUE>50</VALUE>
          </ECUC-NUMERICAL-PARAM-VALUE>
          <ECUC-NUMERICAL-PARAM-VALUE>
            <DEFINITION-REF>/AUTOSAR/Com/ComConfig/ComTimeoutMs</DEFINITION-REF>
            <VALUE>100</VALUE>
          </ECUC-NUMERICAL-PARAM-VALUE>
        </PARAMETER-VALUES>
      </ECUC-CONTAINER-VALUE>
    </CONTAINERS>
  </ECUC-MODULE-CONFIGURATION-VALUES>
</ECUC-VALUE-COLLECTION>
```

```c
/* Generated from ARXML — DO NOT EDIT MANUALLY */
#include "Com_Cfg.h"  /* Contains generated #defines from ARXML */
```

ARXML is the single source of truth for AUTOSAR configuration. Never manually edit generated configuration headers — always modify the ARXML and regenerate. Add CI validation that generated code matches committed code.

Reference: AUTOSAR Classic Platform Specification, AUTOSAR Adaptive Platform Specification
