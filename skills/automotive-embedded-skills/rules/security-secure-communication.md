---
title: Secure In-Vehicle Communication (TLS/DTLS)
impact: HIGH
impactDescription: Protects Ethernet communication from eavesdropping and tampering
tags: security, tls, dtls, ethernet, communication, iso-21434, encryption
---

## Secure In-Vehicle Communication (TLS/DTLS)

Use TLS for TCP-based and DTLS for UDP-based in-vehicle Ethernet communication to prevent eavesdropping and tampering. As vehicles adopt Ethernet (DoIP, SOME/IP), encryption becomes mandatory for safety-critical and privacy-sensitive data.

**Incorrect (unencrypted Ethernet communication):**

```c
Std_ReturnType SendDiagData(int sock, const uint8_t *data, uint16_t len)
{
    send(sock, data, len, 0);
    return E_OK;
}
```

**Correct (TLS-secured communication):**

```c
typedef struct
{
    TlsContext_t *ctx;
    const char   *caCertPath;
    const char   *clientCertPath;
    const char   *clientKeyPath;
    uint16_t      minVersion;  /* TLS_VERSION_1_2 or TLS_VERSION_1_3 */
} SecureConnection_t;

Std_ReturnType Tls_InitClient(SecureConnection_t *conn)
{
    conn->ctx = Tls_CreateContext(TLS_CLIENT_MODE);
    if (conn->ctx == NULL) { return E_NOT_OK; }

    Tls_SetMinVersion(conn->ctx, conn->minVersion);
    Tls_LoadCaCert(conn->ctx, conn->caCertPath);
    Tls_LoadClientCert(conn->ctx, conn->clientCertPath, conn->clientKeyPath);

    /* Disable insecure cipher suites */
    Tls_SetCipherList(conn->ctx,
        "TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384");

    return E_OK;
}
```

Enforce TLS 1.2 as minimum version. Disable SSLv3, TLS 1.0, and TLS 1.1. Use mutual authentication (client certificates) for ECU-to-ECU communication.

Reference: ISO/SAE 21434:2021 — Road vehicles — Cybersecurity engineering
