---
title: Flash Wear Leveling Strategies
impact: MEDIUM
impactDescription: Without wear leveling, frequently written flash blocks exceed endurance limits, causing data loss within the 15+ year automotive lifetime
tags: nvm, wear-leveling, flash, endurance, block-rotation, lifecycle, eeprom-emulation, automotive-lifetime
---

## Flash Wear Leveling Strategies

Flash memory has finite write endurance (typically 10K–100K cycles for embedded NOR flash). Automotive ECUs must survive 15+ years of operation. Wear leveling distributes writes evenly across flash to avoid hot spots. Static wear leveling moves rarely-written data to heavily-used blocks; dynamic wear leveling rotates writes among available blocks.

**Incorrect (always writing to the same flash location):**

```c
/* WRONG: Fixed address — this location will wear out first */
#define CONFIG_ADDR  0x08060000U

void Config_Save(const Config_t *cfg)
{
    Flash_EraseSector(CONFIG_ADDR);
    Flash_Write(CONFIG_ADDR, (const uint8_t *)cfg, sizeof(Config_t));
    /* At 1 write/minute, this sector exhausts 100K cycles in ~69 days */
}
```

**Correct (dynamic wear leveling with block rotation):**

```c
/* wear_level.h — block rotation for EEPROM emulation */
#define WL_NUM_SECTORS       8U      /* Sectors available for rotation */
#define WL_SECTOR_SIZE       4096U
#define WL_BASE_ADDR         0x08060000U
#define WL_ENTRY_SIZE        (sizeof(WL_Header_t) + sizeof(Config_t))

typedef struct {
    uint32_t sequence;    /* Monotonically increasing write counter */
    uint16_t crc;         /* CRC-16 of payload */
    uint8_t  valid;       /* 0xA5 = valid, 0xFF = erased, 0x00 = invalidated */
    uint8_t  reserved;
} WL_Header_t;
```

```c
/* wear_level.c — dynamic wear leveling implementation */
static uint32_t g_currentSector = 0U;
static uint32_t g_currentOffset = 0U;
static uint32_t g_sequenceCounter = 0U;

/* Scan all sectors at startup to find latest valid entry */
void WL_Init(void)
{
    uint32_t highestSeq = 0U;

    for (uint32_t s = 0U; s < WL_NUM_SECTORS; s++)
    {
        uint32_t sectorBase = WL_BASE_ADDR + (s * WL_SECTOR_SIZE);
        uint32_t offset = 0U;

        while (offset + WL_ENTRY_SIZE <= WL_SECTOR_SIZE)
        {
            const WL_Header_t *hdr =
                (const WL_Header_t *)(sectorBase + offset);

            if (hdr->valid == 0xA5U && hdr->sequence >= highestSeq)
            {
                /* Verify CRC */
                const uint8_t *payload = (const uint8_t *)(hdr + 1);
                if (CRC16_Compute(payload, sizeof(Config_t)) == hdr->crc)
                {
                    highestSeq = hdr->sequence;
                    g_currentSector = s;
                    g_currentOffset = offset + WL_ENTRY_SIZE;
                }
            }
            else if (hdr->valid == 0xFFU)
            {
                break;  /* Reached erased area */
            }
            offset += WL_ENTRY_SIZE;
        }
    }
    g_sequenceCounter = highestSeq;
}

/* Append-style write with sector rotation */
WL_Status_t WL_Write(const Config_t *data)
{
    g_sequenceCounter++;

    /* Check if current sector has room */
    if (g_currentOffset + WL_ENTRY_SIZE > WL_SECTOR_SIZE)
    {
        /* Move to next sector */
        g_currentSector = (g_currentSector + 1U) % WL_NUM_SECTORS;
        g_currentOffset = 0U;

        /* Erase the target sector */
        uint32_t sectorAddr = WL_BASE_ADDR + (g_currentSector * WL_SECTOR_SIZE);
        FlashStatus_t fstat = Flash_EraseSector(sectorAddr);
        if (fstat != FLASH_OK) { return WL_ERR_ERASE; }
    }

    /* Prepare header */
    WL_Header_t hdr = {
        .sequence = g_sequenceCounter,
        .crc      = CRC16_Compute((const uint8_t *)data, sizeof(Config_t)),
        .valid    = 0xA5U,
        .reserved = 0xFFU
    };

    /* Write header + data */
    uint32_t writeAddr = WL_BASE_ADDR + (g_currentSector * WL_SECTOR_SIZE)
                         + g_currentOffset;

    uint8_t buffer[WL_ENTRY_SIZE];
    (void)memcpy(buffer, &hdr, sizeof(WL_Header_t));
    (void)memcpy(&buffer[sizeof(WL_Header_t)], data, sizeof(Config_t));

    FlashStatus_t fstat = Flash_WriteAligned(writeAddr, buffer, WL_ENTRY_SIZE);
    if (fstat != FLASH_OK) { return WL_ERR_WRITE; }

    g_currentOffset += WL_ENTRY_SIZE;
    return WL_OK;
}
```

**Write cycle tracking and endurance estimation:**

```c
/* wear_monitor.c — track erase cycles per sector */
typedef struct {
    uint32_t eraseCounts[WL_NUM_SECTORS];
    uint32_t totalWrites;
} WearStats_t;

static WearStats_t g_wearStats;

void WL_RecordErase(uint32_t sectorIndex)
{
    g_wearStats.eraseCounts[sectorIndex]++;
    g_wearStats.totalWrites++;
}

/* Endurance estimation for automotive lifetime */
/*
 * Given:
 *   - Flash endurance: 100,000 cycles per sector
 *   - Available sectors: 8
 *   - Total write capacity: 800,000 cycles
 *   - Write frequency: 1 write per 10 seconds (worst case)
 *   - Writes per year: 365 * 24 * 3600 / 10 = 3,153,600
 *   - Required lifetime: 15 years
 *   - Total writes needed: 47,304,000
 *
 * 800,000 << 47,304,000 — NOT ENOUGH!
 *
 * Mitigations:
 *   1. Increase sector count or use higher-endurance flash
 *   2. Reduce write frequency (buffer changes, write only on delta)
 *   3. Use external EEPROM (1M+ cycle endurance) for hot data
 *   4. Pack multiple entries per sector (append-only log)
 */

bool WL_CheckEnduranceMargin(uint32_t yearsRemaining)
{
    uint32_t maxErases = 0U;
    for (uint32_t s = 0U; s < WL_NUM_SECTORS; s++)
    {
        if (g_wearStats.eraseCounts[s] > maxErases)
        {
            maxErases = g_wearStats.eraseCounts[s];
        }
    }

    uint32_t remainingCycles = FLASH_ENDURANCE_CYCLES - maxErases;
    uint32_t projectedCycles = (g_wearStats.totalWrites / WL_NUM_SECTORS)
                               * yearsRemaining;

    return (remainingCycles > projectedCycles);
}
```

**Static wear leveling — moving cold data:**

```c
/* Static wear leveling moves rarely-written blocks to high-use sectors
 * so that all sectors age evenly. Check periodically. */
void WL_StaticRebalance(void)
{
    uint32_t minErases = UINT32_MAX;
    uint32_t maxErases = 0U;
    uint32_t coldSector = 0U;
    uint32_t hotSector = 0U;

    for (uint32_t s = 0U; s < WL_NUM_SECTORS; s++)
    {
        if (g_wearStats.eraseCounts[s] < minErases)
        {
            minErases = g_wearStats.eraseCounts[s];
            coldSector = s;
        }
        if (g_wearStats.eraseCounts[s] > maxErases)
        {
            maxErases = g_wearStats.eraseCounts[s];
            hotSector = s;
        }
    }

    /* Rebalance if wear difference exceeds threshold */
    if ((maxErases - minErases) > WL_REBALANCE_THRESHOLD)
    {
        WL_SwapSectorContents(coldSector, hotSector);
    }
}
```

Append-only writes within a sector maximize entries per erase cycle. Size the sector pool based on worst-case write frequency over the full vehicle lifetime. Diagnostic monitoring of wear counters via UDS ReadDataByIdentifier enables proactive maintenance.

Reference: JEDEC JESD218 (Flash endurance specification); ISO 26262 Part 5 (Hardware lifetime requirements); AUTOSAR Fee internal wear leveling
