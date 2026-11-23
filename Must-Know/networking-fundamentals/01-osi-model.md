# OSI Model

The Open Systems Interconnection model describes networking as seven layers. Each layer has a job, such as carrying bits across a link, forwarding packets between networks, or interpreting an HTTP request. Modern Internet protocols often combine responsibilities that the OSI model describes separately.

[![Seven-layer OSI stack highlighting the application, transport, and network layers](https://www.hellointerview.com/_next/static/media/B5c4KEuENPxB.2r7-h61f15b1u.svg)](images/osi-model.png)

| Layer | Responsibility | Examples or concepts |
| --- | --- | --- |
| 7. Application | Defines messages and operations that applications understand | HTTP, DNS |
| 6. Presentation | Represents data, including encoding and compression | JSON and Protocol Buffers illustrate representation concerns |
| 5. Session | Coordinates a conversation between endpoints | Session establishment, resumption, and synchronization |
| 4. Transport | Moves data between endpoints and may provide reliability, ordering, or flow control | TCP, UDP, QUIC as a transport implemented over UDP |
| 3. Network | Addresses packets and routes them across networks | IPv4, IPv6 |
| 2. Data link | Transfers frames across a local link | Ethernet frames, Wi-Fi frames, MAC addresses |
| 1. Physical | Carries bits using electrical, radio, or optical signals | Copper, radio, fiber |

## Follow an HTTPS request

[![Timeline showing TCP setup, an HTTP exchange, and TCP teardown across the IP, TCP, and HTTP layers](https://www.hellointerview.com/_next/static/media/cvOR4cCwIO9k.0h5q3zu7d0tfu.svg)](images/http-over-tcp-and-ip.png)

The diagram shows one HTTP exchange over a new TCP connection. When a browser starts a new HTTPS connection over TCP, it resolves the hostname through DNS if needed, establishes TCP, and negotiates TLS before sending the HTTP request in the usual handshake. HTTP defines the request, TLS protects its bytes in records, TCP carries those bytes in a stream, IP routes the resulting packets, and the local network carries packets in frames. TCP segment boundaries can differ from TLS records and HTTP messages. A browser can reuse an open connection for later requests.

```text
HTTP request bytes → TLS records → TCP segments → IP packets → link frames
```

On a typical routed path, each router removes the incoming link-layer frame, consults its forwarding table using the destination IP, and sends the packet toward the next hop in a new link-layer frame. The receiving endpoint reconstructs the byte stream, decrypts it, and parses the request. A proxy that terminates the connection can create a separate transport connection to the backend. HTTP/3 uses QUIC, which integrates TLS 1.3 into its transport. [HTTP/3 specification](https://www.rfc-editor.org/rfc/rfc9114.html)

## Use layers to reason about a design

- **L3 routing** selects a next hop based on IP prefixes.
- **L4 load balancing** commonly selects a backend per flow using IPs, ports, and transport protocol.
- **L7 load balancing** can use HTTP host, path, or headers after decoding the request. On an HTTPS hop, this generally means terminating TLS where that inspection occurs.

An HTTP `502 Bad Gateway` tells you that a gateway answered the client and received an invalid response from its upstream server while trying to fulfill the request. An upstream connection or protocol problem may still be involved, so trace the client-to-gateway and gateway-to-backend hops independently.
