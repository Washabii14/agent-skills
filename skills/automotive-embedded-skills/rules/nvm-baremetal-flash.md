---
title: Bare-Metal Flash and EEPROM Operations
impact: HIGH
impactDescription: Incorrect flash handling causes data corruption, ECC errors, or bricked ECU due to read-while-write violations or alignment errors
tags: nvm, flash, eeprom, bare-metal, ecc, page-erase, write-alignment, dual-bank, sector-erase
---

## Bare-Metal Flash and EEPROM Operations

Bare-metal flash/EEPROM programming requires understanding page/sector erase granularity, write alignment constraints, read-while-write restrictions, ECC behavior, and dual-bank flash for safe updates. These constraints are hardware-specific but follow common patterns across automotive MCU families.

**Incorrect (unaligned write, reading from same flash bank during write):**

```c
/* WRONG: Violates multiple flash programming rules */
void Flash_SaveConfig_Bad(const uint8_t *data, uint32_t size)
{
    /* No erase before write — flash can only transition 1→0 */
    Flash_Write(CONFIG_ADDR, data, size);

    /* Unaligned write — flash requires 8/16-byte aligned writes */
    Flash_Write(CONFIG_ADDR + 3, &single_byte, 1);

    /* Read-while-write: executing code from same flash bank being written */
    /* If this function is in flash bank 0 and CONFIG_ADDR is in bank 0,
       the CPU will stall or get corrupted data */
    printf("Write complete\n");  /* printf may be in same flash bank! */
}
```

**Correct (proper erase-write cycle with alignment and RWW handling):**

```c
/* flash_driver.h — hardware abstraction */
typedef enum {
    FLASH_OK,
    FLASH_ERR_ALIGNMENT,
    FLASH_ERR_ERASE,
    FLASH_ERR_WRITE,
    FLASH_ERR_VERIFY,
    FLASH_ERR_ECC,
    FLASH_ERR_BUSY
} FlashStatus_t;

#define FLASH_PAGE_SIZE        256U     /* Minimum erase unit (varies by MCU) */
#define FLASH_SECTOR_SIZE      4096U    /* Larger erase unit */
#define FLASH_WRITE_ALIGNMENT  8U       /* Minimum write granularity */
#define FLASH_BANK0_START      0x00000000U
#define FLASH_BANK1_START      0x00100000U
```

```c
/* flash_driver.c — safe flash operations */

/* Must execute from RAM or different flash bank during write */
__attribute__((section(".ram_func")))
FlashStatus_t Flash_EraseSector(uint32_t sectorAddr)
{
    if ((sectorAddr % FLASH_SECTOR_SIZE) != 0U)
    {
        return FLASH_ERR_ALIGNMENT;
    }

    __disable_irq();  /* ISR vectors may be in same flash bank */

    /* Unlock flash controller */
    FLASH->KEYR = FLASH_KEY1;
    FLASH->KEYR = FLASH_KEY2;

    /* Set sector erase bit and sector address */
    FLASH->CR |= FLASH_CR_SER;
    FLASH->AR = sectorAddr;
    FLASH->CR |= FLASH_CR_STRT;

    /* Wait for completion */
    while (FLASH->SR & FLASH_SR_BSY) { /* spin */ }

    /* Check for errors */
    FlashStatus_t status = FLASH_OK;
    if (FLASH->SR & (FLASH_SR_WRPERR | FLASH_SR_PGAERR))
    {
        FLASH->SR |= (FLASH_SR_WRPERR | FLASH_SR_PGAERR);  /* Clear flags */
        status = FLASH_ERR_ERASE;
    }

    /* Lock flash controller */
    FLASH->CR |= FLASH_CR_LOCK;

    __enable_irq();
    return status;
}

__attribute__((section(".ram_func")))
FlashStatus_t Flash_WriteAligned(uint32_t addr, const uint8_t *data,
                                  uint32_t size)
{
    /* Enforce write alignment */
    if ((addr % FLASH_WRITE_ALIGNMENT) != 0U ||
        (size % FLASH_WRITE_ALIGNMENT) != 0U)
    {
        return FLASH_ERR_ALIGNMENT;
    }

    __disable_irq();

    FLASH->KEYR = FLASH_KEY1;
    FLASH->KEYR = FLASH_KEY2;
    FLASH->CR |= FLASH_CR_PG;

    const uint64_t *src = (const uint64_t *)data;
    volatile uint64_t *dst = (volatile uint64_t *)addr;
    uint32_t doubleWords = size / 8U;

    for (uint32_t i = 0U; i < doubleWords; i++)
    {
        dst[i] = src[i];
        while (FLASH->SR & FLASH_SR_BSY) { /* spin */ }

        if (FLASH->SR & FLASH_SR_PGAERR)
        {
            FLASH->CR &= ~FLASH_CR_PG;
            FLASH->CR |= FLASH_CR_LOCK;
            __enable_irq();
            return FLASH_ERR_WRITE;
        }
    }

    FLASH->CR &= ~FLASH_CR_PG;
    FLASH->CR |= FLASH_CR_LOCK;
    __enable_irq();

    return FLASH_OK;
}
```

**Safe read-modify-write with page buffer:**

```c
/* Updating a subset of data within a sector */
FlashStatus_t Flash_UpdatePartial(uint32_t sectorAddr, uint32_t offset,
                                   const uint8_t *data, uint32_t dataSize)
{
    static uint8_t sectorBuf[FLASH_SECTOR_SIZE];

    /* 1. Read entire sector into RAM */
    (void)memcpy(sectorBuf, (const void *)sectorAddr, FLASH_SECTOR_SIZE);

    /* 2. Modify the target region in RAM */
    (void)memcpy(&sectorBuf[offset], data, dataSize);

    /* 3. Erase the sector */
    FlashStatus_t status = Flash_EraseSector(sectorAddr);
    if (status != FLASH_OK) { return status; }

    /* 4. Write back entire sector from RAM */
    status = Flash_WriteAligned(sectorAddr, sectorBuf, FLASH_SECTOR_SIZE);
    if (status != FLASH_OK) { return status; }

    /* 5. Verify written data */
    if (memcmp((const void *)sectorAddr, sectorBuf, FLASH_SECTOR_SIZE) != 0)
    {
        return FLASH_ERR_VERIFY;
    }

    return FLASH_OK;
}
```

**ECC error handling:**

```c
/* Flash ECC: most automotive MCUs use ECC on internal flash.
 * Single-bit errors are corrected silently.
 * Double-bit errors (uncorrectable) trigger an NMI or bus fault.
 *
 * ECC is computed per write-granularity unit (e.g., 64-bit).
 * NEVER write the same flash location twice without erasing first —
 * this corrupts the ECC and causes hard faults on subsequent reads.
 */

void NMI_Handler(void)
{
    if (FLASH->ECCSR & FLASH_ECCSR_ECCD)
    {
        uint32_t faultAddr = FLASH->ECCFAR;
        /* Log the ECC failure address */
        SafeState_EnterDegraded(SAFE_REASON_FLASH_ECC, faultAddr);
        FLASH->ECCSR |= FLASH_ECCSR_ECCD;  /* Clear flag */
    }
}
```

**Dual-bank flash for safe firmware update:**

```c
/* Dual-bank: one bank runs active firmware, other receives update.
 * No read-while-write conflict since banks are independent. */

#define ACTIVE_BANK   FLASH_BANK0_START
#define UPDATE_BANK   FLASH_BANK1_START

FlashStatus_t FirmwareUpdate_DualBank(const uint8_t *newFw, uint32_t fwSize)
{
    /* 1. Erase update bank (code is running from active bank — safe) */
    for (uint32_t addr = UPDATE_BANK;
         addr < UPDATE_BANK + fwSize;
         addr += FLASH_SECTOR_SIZE)
    {
        Flash_EraseSector(addr);
        WDG_Trigger();
    }

    /* 2. Write new firmware to update bank */
    Flash_WriteAligned(UPDATE_BANK, newFw, ALIGN_UP(fwSize, FLASH_WRITE_ALIGNMENT));

    /* 3. Verify */
    if (memcmp((const void *)UPDATE_BANK, newFw, fwSize) != 0)
    {
        return FLASH_ERR_VERIFY;
    }

    /* 4. Swap bank boot selection (MCU-specific register) */
    Flash_SwapBanks();  /* Next reset boots from UPDATE_BANK */

    return FLASH_OK;
}
```

Always trigger the watchdog during long erase/write operations. Place flash-programming routines in RAM (`.ram_func` section) or a different flash bank to avoid read-while-write conflicts. Never write to the same flash doubleword twice without erasing — this corrupts ECC.

Reference: MCU-specific Flash Programming Manual; ISO 26262 Part 5 (Hardware — non-volatile memory integrity); MISRA C:2012 Rule 11.4 (pointer conversions for memory-mapped I/O)
