---
title: SOME/IP Serialization
impact: MEDIUM
impactDescription: correct Adaptive AUTOSAR service-oriented communication
tags: comm, someip, serialization, ethernet, service-oriented, autosar-adaptive
---

## SOME/IP Serialization

Use correct SOME/IP serialization with proper byte order (big-endian for header, configurable for payload), length fields, and padding alignment. The SOME/IP header is 16 bytes and must be serialized in network byte order.

**Incorrect (host byte order, no buffer validation):**

```c
void SomeIp_Send(uint16_t serviceId, const uint8_t *payload, uint16_t len)
{
    uint8_t buf[1400];
    SomeIp_Header_t *hdr = (SomeIp_Header_t *)buf;
    hdr->serviceId = serviceId;  /* Host byte order — wrong */
    hdr->length = len;
    memcpy(&buf[16], payload, len);
    Udp_Send(buf, 16 + len);
}
```

**Correct (network byte order, validated serialization):**

```c
typedef struct __attribute__((packed))
{
    uint16_t serviceId;
    uint16_t methodId;
    uint32_t length;
    uint16_t clientId;
    uint16_t sessionId;
    uint8_t  protocolVersion;
    uint8_t  interfaceVersion;
    uint8_t  messageType;
    uint8_t  returnCode;
} SomeIp_Header_t;

Std_ReturnType SomeIp_SerializeRequest(uint8_t *buf, uint16_t bufSize,
                                        uint16_t serviceId, uint16_t methodId,
                                        const uint8_t *payload, uint16_t payloadLen,
                                        uint16_t *totalLen)
{
    uint16_t headerAndPayload = SOMEIP_HEADER_LEN + payloadLen;
    if (headerAndPayload > bufSize) { return E_NOT_OK; }

    SomeIp_Header_t *hdr = (SomeIp_Header_t *)buf;
    hdr->serviceId        = htons(serviceId);
    hdr->methodId         = htons(methodId);
    hdr->length           = htonl(payloadLen + 8U);
    hdr->clientId         = htons(g_clientId);
    hdr->sessionId        = htons(g_nextSessionId++);
    hdr->protocolVersion  = SOMEIP_PROTOCOL_VERSION;
    hdr->interfaceVersion = 1U;
    hdr->messageType      = SOMEIP_MSG_REQUEST;
    hdr->returnCode       = SOMEIP_RC_OK;

    (void)memcpy(&buf[SOMEIP_HEADER_LEN], payload, payloadLen);
    *totalLen = headerAndPayload;
    return E_OK;
}
```

The `length` field covers clientId through end of payload (payload + 8 bytes). Always use `htons()`/`htonl()` for header fields.

Reference: AUTOSAR PRS_SOMEIP (SOME/IP Protocol Specification)
