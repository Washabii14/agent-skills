---
title: UDP Datagram Handling
impact: HIGH
impactDescription: correct service discovery and low-latency communication
tags: comm, udp, ethernet, datagram, multicast, someip-sd
---

## UDP Datagram Handling

UDP is used for SOME/IP-SD (Service Discovery), DoIP vehicle identification, and real-time sensor streaming where low latency matters more than guaranteed delivery. Always validate sender, buffer sizes, and received length.

**Incorrect (no buffer size check, no source validation):**

```c
void UdpReceive(int sock)
{
    uint8_t buf[256];
    recv(sock, buf, sizeof(buf), 0);  /* Ignores sender, no length check */
    ProcessData(buf);
}
```

**Correct (validated reception with sender identification):**

```c
Std_ReturnType Udp_Receive(int sock, uint8_t *buf, uint16_t bufSize,
                            uint16_t *receivedLen,
                            struct sockaddr_in *senderAddr)
{
    socklen_t addrLen = sizeof(*senderAddr);
    ssize_t n = recvfrom(sock, buf, bufSize, 0,
                         (struct sockaddr *)senderAddr, &addrLen);

    if (n < 0)
    {
        *receivedLen = 0U;
        return E_NOT_OK;
    }

    *receivedLen = (uint16_t)n;

    if (!IsAuthorizedSender(senderAddr))
    {
        *receivedLen = 0U;
        return E_NOT_OK;
    }

    return E_OK;
}
```

**UDP multicast for SOME/IP-SD:**

```c
Std_ReturnType Udp_JoinMulticast(int sock, const char *multicastIp,
                                  const char *localIp)
{
    struct ip_mreq mreq;
    mreq.imr_multiaddr.s_addr = inet_addr(multicastIp);
    mreq.imr_interface.s_addr = inet_addr(localIp);

    if (setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP,
                   &mreq, sizeof(mreq)) < 0)
    {
        return E_NOT_OK;
    }
    return E_OK;
}
```

Reference: AUTOSAR SWS Ethernet (SWS_Eth); SOME/IP Protocol Specification (PRS_SOMEIP)
