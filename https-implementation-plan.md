# HTTPS Implementation Plan for Titanium Web Proxy

This document outlines the plan to add support for an HTTPS listening endpoint and HTTPS upstream proxies in the Titanium Web Proxy.

## Feature Overview

This implementation addresses two key requirements:

1.  **HTTPS Listening Endpoint:** Allows the proxy server to accept direct HTTPS connections from clients without requiring an initial `CONNECT` request over HTTP.
2.  **HTTPS Upstream Proxy Support:** Enables the proxy to forward client requests to an upstream proxy that listens on HTTPS.

## Scenarios

**Scenario 1: Client connecting directly to an HTTPS proxy endpoint**

A client is configured to use the Titanium Web Proxy as an HTTPS proxy (e.g., `https://proxy-ip:proxy-port`). The client sends an HTTPS request directly to the proxy. The proxy performs the SSL handshake with the client, using a dynamically generated or retrieved certificate matching the target hostname requested by the client (via SNI). After the handshake, the proxy decrypts the client's request, processes it, and forwards it to the final destination (either directly or via an upstream proxy).

**Scenario 2: Client connecting to the proxy, which then uses an HTTPS upstream proxy**

A client sends a request to the Titanium Web Proxy. The proxy is configured to use an upstream proxy that listens on HTTPS (e.g., `UpStreamHttpsProxy = new ExternalProxy("upstream-ip", upstream-port, ExternalProxyType.Https)`). When the proxy needs to forward the client's request to the upstream proxy, it establishes a TCP connection to the upstream proxy's IP and port. It then performs an SSL handshake as a client with the upstream proxy. After the secure tunnel to the upstream proxy is established, the proxy sends a `CONNECT` request to the upstream proxy over this encrypted tunnel, followed by the client's actual request.

## Implementation Plan

1.  **Enhance `ExplicitProxyEndPoint` (`src/Titanium.Web.Proxy/Models/ExplicitProxyEndPoint.cs`):**

    - Add a public boolean property named `IsHttps`. This property will indicate if the endpoint should listen for direct HTTPS connections.

2.  **Modify Incoming Connection Handling (`src/Titanium.Web.Proxy/ExplicitClientHandler.cs`):**

    - In the `HandleClient` method, add a check for the `IsHttps` property of the `ExplicitProxyEndPoint` at the beginning of the method.
    - If `IsHttps` is true, perform an SSL/TLS handshake with the client _before_ attempting to read the initial HTTP method.
    - Implement logic to extract the target hostname from the client's SNI during the handshake.
    - Use the `CertificateManager` to generate or retrieve a certificate for the extracted hostname and use it for the server authentication in the SSL handshake.
    - Wrap the existing `clientStream` with an `SslStream` after successful authentication.
    - If `IsHttps` is false, retain the existing logic for handling HTTP and `CONNECT` requests.

3.  **Modify Upstream Proxy Connection Handling (`src/Titanium.Web.Proxy/Network/TcpConnection/TcpConnectionFactory.cs`):**
    - In the `CreateServerConnection` method, locate the section that handles `externalProxy`.
    - Add a condition to check if the `externalProxy.ProxyType` is `ExternalProxyType.Https`.
    - If it is an HTTPS upstream proxy, perform an SSL/TLS handshake as a client with the upstream proxy _after_ establishing the TCP connection but _before_ sending the `CONNECT` request.
    - Wrap the stream connected to the upstream proxy with an `SslStream` after successful client authentication.
    - Continue with sending the `CONNECT` request and relaying data over the encrypted stream.

## File Changes

- `src/Titanium.Web.Proxy/Models/ExplicitProxyEndPoint.cs`: Add `IsHttps` property.
- `src/Titanium.Web.Proxy/ExplicitClientHandler.cs`: Add logic for server-side SSL handshake based on `IsHttps` property, including SNI extraction and certificate handling.
- `src/Titanium.Web.Proxy/Network/TcpConnection/TcpConnectionFactory.cs`: Add logic for client-side SSL handshake when connecting to an HTTPS upstream proxy.
