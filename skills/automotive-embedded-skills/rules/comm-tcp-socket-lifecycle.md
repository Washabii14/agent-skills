---
title: TCP Socket Lifecycle Management
impact: HIGH
impactDescription: reliable in-vehicle Ethernet communication
tags: comm, tcp, ethernet, socket, lifecycle, timeout, keepalive
---

## TCP Socket Lifecycle Management

Manage TCP socket lifecycle correctly: connection establishment, keepalive monitoring, graceful shutdown, and error recovery. Automotive Ethernet TCP connections must handle ECU sleep/wake transitions and network partitioning.

**Incorrect (no error handling, no timeout, resource leak):**

```c
void SendDiagData(const uint8_t *data, uint16_t len)
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    connect(sock, (struct sockaddr *)&serverAddr, sizeof(serverAddr));
    send(sock, data, len, 0);
    /* Socket never closed — resource leak */
}
```

**Correct (full lifecycle with timeout and cleanup):**

```c
typedef struct
{
    int       fd;
    uint32_t  connectTimeoutMs;
    uint32_t  sendTimeoutMs;
    uint32_t  keepaliveIntervalMs;
    boolean   isConnected;
} TcpConnection_t;

Std_ReturnType Tcp_Connect(TcpConnection_t *conn,
                            const struct sockaddr_in *serverAddr)
{
    conn->fd = socket(AF_INET, SOCK_STREAM, 0);
    if (conn->fd < 0)
    {
        return E_NOT_OK;
    }

    struct timeval timeout;
    timeout.tv_sec  = conn->connectTimeoutMs / 1000U;
    timeout.tv_usec = (conn->connectTimeoutMs % 1000U) * 1000U;
    (void)setsockopt(conn->fd, SOL_SOCKET, SO_SNDTIMEO,
                     &timeout, sizeof(timeout));

    int keepalive = 1;
    (void)setsockopt(conn->fd, SOL_SOCKET, SO_KEEPALIVE,
                     &keepalive, sizeof(keepalive));

    if (connect(conn->fd, (const struct sockaddr *)serverAddr,
                sizeof(*serverAddr)) < 0)
    {
        close(conn->fd);
        conn->fd = -1;
        return E_NOT_OK;
    }

    conn->isConnected = TRUE;
    return E_OK;
}

Std_ReturnType Tcp_Send(TcpConnection_t *conn,
                         const uint8_t *data, uint16_t len)
{
    if (!conn->isConnected || conn->fd < 0)
    {
        return E_NOT_OK;
    }

    ssize_t totalSent = 0;
    while (totalSent < (ssize_t)len)
    {
        ssize_t sent = send(conn->fd, &data[totalSent],
                            len - (uint16_t)totalSent, MSG_NOSIGNAL);
        if (sent <= 0)
        {
            conn->isConnected = FALSE;
            return E_NOT_OK;
        }
        totalSent += sent;
    }
    return E_OK;
}

void Tcp_Disconnect(TcpConnection_t *conn)
{
    if (conn->fd >= 0)
    {
        (void)shutdown(conn->fd, SHUT_RDWR);
        (void)close(conn->fd);
        conn->fd = -1;
    }
    conn->isConnected = FALSE;
}
```

Key considerations for automotive TCP:
- Always set send/receive timeouts to prevent indefinite blocking
- Use `SO_KEEPALIVE` to detect broken connections during ECU idle
- Handle `EPIPE`/`ECONNRESET` for ungraceful peer disconnection
- Implement reconnection logic with exponential backoff
- Close sockets properly on ECU shutdown to release resources

Reference: AUTOSAR SWS Ethernet (SWS_Eth); AUTOSAR SWS TCP/IP Stack
