---
title: ARP Table Management
impact: MEDIUM
impactDescription: deterministic Ethernet address resolution
tags: comm, arp, ethernet, address-resolution, static-arp, deterministic
---

## ARP Table Management

In automotive Ethernet networks, use static ARP entries for known ECU addresses to ensure deterministic communication startup. Dynamic ARP introduces non-deterministic latency at first communication.

**Incorrect (relying solely on dynamic ARP):**

```c
void SendToEcu(uint32_t targetIp, const uint8_t *data, uint16_t len)
{
    Eth_Transmit(targetIp, data, len);  /* First packet delayed by ARP */
}
```

**Correct (static ARP table with dynamic fallback):**

```c
typedef struct
{
    uint32_t ipAddr;
    uint8_t  macAddr[6];
    boolean  isStatic;
    uint32_t lastSeenMs;
    uint16_t timeoutMs;
} ArpEntry_t;

#define ARP_TABLE_SIZE (32U)
static ArpEntry_t g_arpTable[ARP_TABLE_SIZE];

Std_ReturnType Arp_AddStaticEntry(uint32_t ip, const uint8_t mac[6])
{
    for (uint8_t i = 0U; i < ARP_TABLE_SIZE; i++)
    {
        if (g_arpTable[i].ipAddr == 0U)
        {
            g_arpTable[i].ipAddr = ip;
            (void)memcpy(g_arpTable[i].macAddr, mac, 6U);
            g_arpTable[i].isStatic = TRUE;
            return E_OK;
        }
    }
    return E_NOT_OK;  /* Table full */
}

const uint8_t *Arp_Resolve(uint32_t ip)
{
    for (uint8_t i = 0U; i < ARP_TABLE_SIZE; i++)
    {
        if (g_arpTable[i].ipAddr == ip)
        {
            return g_arpTable[i].macAddr;
        }
    }
    Arp_SendRequest(ip);  /* Dynamic resolution fallback */
    return NULL;
}
```

Best practices:
- Pre-populate static ARP entries for all known ECUs at boot
- Implement ARP timeout for dynamic entries (typically 60-300s in automotive)
- Handle duplicate IP detection via gratuitous ARP
- Log ARP conflicts as DTCs for network diagnostics

Reference: AUTOSAR SWS TCP/IP Stack; IEEE 802.3
