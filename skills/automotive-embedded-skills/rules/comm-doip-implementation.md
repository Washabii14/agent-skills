---
title: Diagnostics over IP (DoIP)
impact: HIGH
impactDescription: ISO 13400 compliance for remote diagnostics
tags: comm, doip, ethernet, diagnostics, iso-13400, routing-activation
---

## Diagnostics over IP (DoIP)

DoIP enables UDS diagnostics over TCP/IP. Implement proper vehicle identification, routing activation, and diagnostic message handling per ISO 13400. DoIP uses TCP port 13400 for diagnostic sessions and UDP port 13400 for vehicle identification broadcasts.

**Incorrect (no header validation, no routing activation):**

```c
void DoIp_HandleMessage(const uint8_t *data, uint16_t len)
{
    ProcessDiagRequest(&data[8], len - 8U);  /* Skip header blindly */
}
```

**Correct (DoIP routing activation with validation):**

```c
#define DOIP_PORT                  (13400U)
#define DOIP_HEADER_LEN            (8U)
#define DOIP_TYPE_ROUTING_ACT_REQ  (0x0005U)
#define DOIP_TYPE_ROUTING_ACT_RESP (0x0006U)
#define DOIP_TYPE_DIAG_MSG         (0x8001U)
#define DOIP_TYPE_DIAG_MSG_ACK     (0x8002U)
#define DOIP_TYPE_VEHICLE_ID_REQ   (0x0001U)
#define DOIP_TYPE_VEHICLE_ID_RESP  (0x0004U)

typedef struct
{
    uint8_t  protocolVersion;
    uint8_t  inverseVersion;
    uint16_t payloadType;
    uint32_t payloadLength;
} DoIp_Header_t;

Std_ReturnType DoIp_HandleRoutingActivation(
    const uint8_t *request, uint16_t len,
    uint8_t *response, uint16_t *respLen,
    uint16_t sourceTesterAddr)
{
    if (len < (DOIP_HEADER_LEN + 7U))
    {
        return E_NOT_OK;
    }

    uint16_t testerAddr = ((uint16_t)request[DOIP_HEADER_LEN] << 8U) |
                           request[DOIP_HEADER_LEN + 1U];

    if (testerAddr != sourceTesterAddr)
    {
        DoIp_SendRoutingResponse(response, respLen, testerAddr,
                                  DOIP_ROUTING_DENIED_UNKNOWN_SA);
        return E_NOT_OK;
    }

    if (g_activeConnections >= DOIP_MAX_CONNECTIONS)
    {
        DoIp_SendRoutingResponse(response, respLen, testerAddr,
                                  DOIP_ROUTING_DENIED_ALL_SOCKETS_ACTIVE);
        return E_NOT_OK;
    }

    DoIp_RegisterConnection(testerAddr);
    DoIp_SendRoutingResponse(response, respLen, testerAddr,
                              DOIP_ROUTING_SUCCESS);
    return E_OK;
}
```

**Vehicle identification via UDP broadcast (port 13400):**

```c
Std_ReturnType DoIp_HandleVehicleIdentRequest(
    uint8_t *response, uint16_t *respLen)
{
    DoIp_Header_t header;
    header.protocolVersion = DOIP_PROTOCOL_VERSION;
    header.inverseVersion  = (uint8_t)~DOIP_PROTOCOL_VERSION;
    header.payloadType     = DOIP_TYPE_VEHICLE_ID_RESP;

    uint16_t offset = DOIP_HEADER_LEN;
    (void)memcpy(&response[offset], g_vin, VIN_LENGTH);
    offset += VIN_LENGTH;
    response[offset++] = (uint8_t)(g_logicalAddr >> 8U);
    response[offset++] = (uint8_t)(g_logicalAddr & 0xFFU);
    (void)memcpy(&response[offset], g_eid, EID_LENGTH);
    offset += EID_LENGTH;
    (void)memcpy(&response[offset], g_gid, GID_LENGTH);
    offset += GID_LENGTH;

    header.payloadLength = offset - DOIP_HEADER_LEN;
    (void)memcpy(response, &header, DOIP_HEADER_LEN);
    *respLen = offset;
    return E_OK;
}
```

Reference: ISO 13400-2:2019 — DoIP transport protocol and network layer services
