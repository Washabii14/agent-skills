---
title: DHCP and AutoIP Address Configuration
impact: MEDIUM
impactDescription: IP address management for automotive networks
tags: comm, dhcp, autoip, ip-configuration, link-local, ethernet
---

## DHCP and AutoIP Address Configuration

Implement DHCP client with AutoIP (link-local) fallback for ECU IP address assignment. In automotive networks, static IP is preferred for determinism, but DHCP is used for aftermarket/diagnostic devices.

**Incorrect (hardcoded IP with no fallback):**

```c
void InitNetwork(void)
{
    Eth_SetIpAddress(0xC0A80064U, 0xFFFFFF00U, 0xC0A80001U);
}
```

**Correct (configurable IP with DHCP and AutoIP fallback):**

```c
typedef enum
{
    IP_CONFIG_STATIC,
    IP_CONFIG_DHCP,
    IP_CONFIG_AUTOIP,
    IP_CONFIG_DHCP_WITH_AUTOIP_FALLBACK
} IpConfigMode_t;

typedef struct
{
    IpConfigMode_t mode;
    uint32_t ipAddr;
    uint32_t subnetMask;
    uint32_t gateway;
    uint32_t dhcpLeaseTimeS;
    uint32_t dhcpRetryCount;
    boolean  isConfigured;
} IpConfig_t;

Std_ReturnType IpConfig_Init(IpConfig_t *cfg)
{
    switch (cfg->mode)
    {
        case IP_CONFIG_STATIC:
            Eth_SetIpAddress(cfg->ipAddr, cfg->subnetMask, cfg->gateway);
            cfg->isConfigured = TRUE;
            return E_OK;

        case IP_CONFIG_DHCP:
            return Dhcp_StartDiscovery(cfg);

        case IP_CONFIG_DHCP_WITH_AUTOIP_FALLBACK:
            if (Dhcp_StartDiscovery(cfg) != E_OK)
            {
                return AutoIp_GenerateLinkLocal(cfg);
            }
            return E_OK;

        case IP_CONFIG_AUTOIP:
            return AutoIp_GenerateLinkLocal(cfg);

        default:
            return E_NOT_OK;
    }
}
```

AutoIP generates a link-local address in the 169.254.x.x range per RFC 3927 when no DHCP server is available, ensuring the ECU can still communicate on the local network segment.

Reference: RFC 2131 (DHCP); RFC 3927 (AutoIP / IPv4 Link-Local); AUTOSAR SWS TCP/IP Stack
